<!--
#npm #publishing #open-source #packages #2fa #npmjs
-->

# Publishing an npm Package

## Introduction

Publishing your package to npm. From local development to public availability. This post covers the publishing process, authentication, and best practices for making your package available to others. Perfect for sharing your solution with the community.

## The Problem

You've built a package, tested it locally, and it works. But how do you share it? Publishing to npm requires authentication, understanding what gets published, and handling versioning. The process can be confusing the first time.

## The Solution

Prepare your package, authenticate with npm, and publish. Use `npm publish` with proper configuration to make your package available to everyone.

**Pre-publish checklist:**

1. ✅ Package builds successfully
2. ✅ README is clear and helpful
3. ✅ Version number is correct
4. ✅ `.npmignore` excludes source files
5. ✅ `files` field in `package.json` is correct
6. ✅ You're logged into npm

**Verify what will be published:**

```bash
npm publish --dry-run
```

This shows exactly what files will be included in the published package.

**Authentication:**

1. **Check if logged in:**

   ```bash
   npm whoami
   ```

2. **Login if needed:**

   ```bash
   npm login
   ```

   Opens browser for authentication.

3. **Enable 2FA (required for publishing):**
   - Go to https://www.npmjs.com/settings/[username]/security
   - Enable two-factor authentication
   - Required for publishing packages

**Publishing:**

```bash
npm publish
```

The `prepublishOnly` script automatically runs `tsc` to build before publishing, ensuring your `dist/` folder is up to date.

**Version management:**

- **Patch** (1.0.0 → 1.0.1): Bug fixes
- **Minor** (1.0.0 → 1.1.0): New features, backward compatible
- **Major** (1.0.0 → 2.0.0): Breaking changes

Update version in `package.json`, then publish:

```bash
# Edit package.json version
npm publish
```

**What gets published:**

Your `.npmignore` and `files` field control what's included:

```json
// package.json
{
  "files": ["dist"]
}
```

```bash
# .npmignore
src/
tsconfig.json
.gitignore
node_modules/
*.log
.DS_Store
.env
*.tsbuildinfo
yarn.lock
package-lock.json
```

**Common issues:**

1. **403 Forbidden - 2FA required:**

   ```
   npm ERR! 403 Two-factor authentication required
   ```

   **Solution:** Enable 2FA in npm account settings

2. **Package name taken:**

   ```
   npm ERR! 403 Package name already exists
   ```

   **Solution:** Choose a different name or use scoped package (`@username/package-name`)

3. **Version already published:**
   ```
   npm ERR! 403 You cannot publish over the previously published versions
   ```
   **Solution:** Bump version number in `package.json`

**After publishing:**

Your package is available at:

```
https://www.npmjs.com/package/your-package-name
```

Users can install with:

```bash
npm install your-package-name
```

**Updating the package:**

1. Make changes
2. Update version in `package.json`
3. Build: `yarn build`
4. Publish: `npm publish`

## Benefits

- **Public availability** - Anyone can install and use your package
- **Version control** - Semantic versioning tracks changes
- **Distribution** - No need to share code manually
- **Discovery** - Others can find your solution
- **Professional** - Demonstrates package development skills

Next: Your package is live and ready to use!
