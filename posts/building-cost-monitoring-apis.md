<!--
#nextjs #postgresql #api #costmonitoring #sql #cte #fullstack #database #median
-->

# Cost Risk Bucketing with SQL Median Calculations

When you need to identify which projects are at risk of going over budget, you could fetch all your data into JavaScript and calculate everything in memory. But as your project count grows, that approach gets slow and error-prone. Instead, let PostgreSQL do what it does best: calculate medians, aggregate costs, and categorize projects into risk buckets—all in a single query.

This post shows how to build a cost monitoring API that uses PostgreSQL's `PERCENTILE_CONT` to calculate medians from historical data, then uses CTEs to categorize active projects into risk buckets. The database handles all the heavy lifting, keeping your Next.js API code simple and performant.

## The Problem

You need to identify which active projects are at risk of cost overruns. This means comparing each project's current spending against a baseline derived from historical data. But there are a few challenges:

**Averages are misleading.** If you have 10 projects that cost $10k each and one that cost $100k, your average is $18k—skewed by that single outlier. That's not a good baseline for predicting typical project costs.

**JavaScript calculations don't scale.** Fetching all completed project costs into memory, calculating averages, then processing every active project in JavaScript works fine for small datasets. But as your project count grows, this becomes slow and error-prone.

```javascript
// This works, but doesn't scale
const average = costs.reduce((a, b) => a + b, 0) / costs.length
const overruns = activeProjects.map((project) => {
  const percent = ((project.cost - average) / average) * 100
  let status = 'ok'
  if (percent > 75) status = 'overrun'
  else if (percent >= 25) status = 'warning'
  return { ...project, status }
})
```

You need a solution that handles outliers better and scales efficiently. That's where PostgreSQL's median calculations and CTEs come in.

## The Solution

The solution has two parts: first, calculate the median cost from completed projects (this gives us a robust baseline that handles outliers). Then, use a CTE pipeline to compare active projects against that baseline and categorize them into risk buckets.

Let's start with calculating the median:

```sql
SELECT 
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY cost::numeric) as global_median,
  COUNT(*) as sample_size
FROM completed_projects
WHERE cost_category_id = $1
  AND cost IS NOT NULL
  AND cost::numeric > 0
```

This query calculates the median cost from completed projects in a given category. The `PERCENTILE_CONT(0.5)` function gives us the 50th percentile, which is the median.

**The CTE pipeline:**

Now we build a CTE pipeline that filters active projects, aggregates their costs, calculates overrun percentages, and applies risk buckets. Each CTE builds on the previous one:

```sql
WITH active_items AS (
  -- Filter active projects (exclude completed)
  SELECT DISTINCT
    p.id,
    p.organization_id,
    o.name as organization_name
  FROM projects p
  JOIN organizations o ON o.id = p.organization_id
  WHERE p.status != 'completed'
    AND p.cost_category_id = $1
),
item_costs AS (
  -- Aggregate actual costs from approved transactions
  SELECT 
    ai.id,
    ai.organization_id,
    ai.organization_name,
    COALESCE(SUM(t.amount), 0) as current_cost
  FROM active_items ai
  LEFT JOIN transactions t ON t.project_id = ai.id
    AND t.cost_category_id = $1
    AND t.status = 'approved'
  GROUP BY ai.id, ai.organization_id, ai.organization_name
  HAVING COALESCE(SUM(t.amount), 0) > 0
),
cost_overruns AS (
  -- Calculate overrun percentage using the median
  SELECT 
    ic.*,
    $2::numeric as predicted_cost,
    CASE 
      WHEN $2::numeric > 0 
      THEN ((ic.current_cost - $2::numeric) / $2::numeric) * 100
      ELSE NULL
    END as overrun_percent
  FROM item_costs ic
),
items_with_status AS (
  -- THE BUCKETING LOGIC: Categorize by risk thresholds
  SELECT 
    *,
    CASE 
      WHEN overrun_percent > 75 THEN 'overrun'    -- Critical: >75% over budget
      WHEN overrun_percent >= 25 THEN 'warning'   -- At risk: 25-75% over
      ELSE 'ok'                                    -- On track: ≤25% over
    END as status
  FROM cost_overruns
)
SELECT 
  id,
  organization_id,
  organization_name,
  current_cost,
  predicted_cost,
  overrun_percent,
  status,
  COUNT(*) OVER() as total_count
FROM items_with_status
WHERE 1=1 ${statusFilter}  -- optional status filter
ORDER BY 
  CASE status
    WHEN 'overrun' THEN 1
    WHEN 'warning' THEN 2
    ELSE 3
  END,
  overrun_percent DESC NULLS LAST
LIMIT $3 OFFSET $4
```

In your API route, you'd execute this query with the median value as `$2`, along with your category ID, status filter (if any), limit, and offset. The query returns projects with their risk status already calculated.

The CTE pipeline works in four steps:

1. **`active_items`** - Filters to active projects (excluding completed ones) and joins with organizations
2. **`item_costs`** - Aggregates actual costs from approved transactions for each project
3. **`cost_overruns`** - Calculates the percentage over/under the predicted median cost
4. **`items_with_status`** - Applies risk buckets based on overrun percentage

The final query filters by status (if provided), orders by risk priority (overruns first), and applies pagination. The `COUNT(*) OVER()` window function gives us the total count before pagination, which is useful for building pagination controls.

**How the bucketing works:**

The bucketing logic in `items_with_status` uses a `CASE` statement to categorize projects into three risk levels:

- **Overrun (>75%)**: Critical projects that are significantly over budget and need immediate attention
- **Warning (25-75%)**: At-risk projects that are trending over budget and should be monitored closely
- **Ok (≤25%)**: Projects on track or only slightly over budget, within acceptable variance

The thresholds are applied after calculating the overrun percentage, which creates a clean separation of concerns. If you need to adjust thresholds or add new risk categories, you only modify the `items_with_status` CTE.

**Why median instead of average:**

Median (`PERCENTILE_CONT(0.5)`) represents the 50th percentile—half the projects cost more, half cost less. Unlike average, median isn't affected by outliers. 

Remember that example from earlier? With 10 projects at $10k and one at $100k, the average is $18k (skewed by the outlier), but the median is $10k (representative of typical costs). Using median gives you a more realistic baseline for cost predictions, especially when your historical data includes occasional high-cost projects that shouldn't influence your baseline expectations.

The `PERCENTILE_CONT` function handles this calculation efficiently in the database, even with large datasets, and returns `NULL` when there's insufficient data.

## Why This Approach Works

This solution gives you several advantages over calculating everything in JavaScript:

**Performance** - PostgreSQL processes the median calculation, cost aggregation, percentage math, and bucketing in a single query. No multiple round trips, no loading data into memory, no JavaScript processing overhead. The database can optimize the entire pipeline and use indexes effectively.

**Accuracy** - Median resists outliers better than average. One extremely expensive project won't skew your baseline prediction, giving you a more representative cost estimate for risk assessment.

**Scalability** - Works efficiently with large datasets. The CTE structure lets PostgreSQL optimize the entire pipeline, and you can add indexes on the columns used in filters and joins (`cost_category_id`, `status`, `project_id`). The database processes everything in one pass rather than loading data into application memory.

**Maintainability** - Complex logic lives in SQL, not application code. The CTE pipeline makes it easy to see how each step transforms the data, from filtering active projects to calculating percentages to applying risk buckets. Changes to thresholds or calculations are localized to the SQL query.

## Performance Tips

For optimal performance with large datasets, ensure you have indexes on:
- `completed_projects(cost_category_id, cost)` - for median calculation
- `projects(cost_category_id, status)` - for filtering active projects  
- `transactions(project_id, cost_category_id, status)` - for cost aggregation

The CTE structure allows PostgreSQL to optimize the entire query plan, but indexes on filter and join columns are essential for large datasets.

## Wrapping Up

By leveraging PostgreSQL's `PERCENTILE_CONT` for median calculations and CTEs for the bucketing pipeline, you get a cost monitoring API that's both performant and maintainable. The database handles all the heavy lifting, your API code stays simple, and you can easily adjust thresholds or add new risk categories as your needs evolve.

The key insight is letting the database do what it does best: aggregate, calculate, and filter data efficiently. Your application code just orchestrates the query and returns the results.

Next: [pagination-strategies.md](./pagination-strategies.md)
