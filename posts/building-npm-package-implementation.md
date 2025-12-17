<!--
#npm #typescript #packages #implementation #singleton #nextjs #transformers
-->

# Building an npm Package: Implementation

## Introduction

Implementing an npm package with TypeScript. From source code to build output. This post covers the technical implementation, build configuration, and design decisions that make a package production-ready. Perfect for turning reusable code into a distributable package.

## The Problem

You've identified code worth packaging, but how do you structure it? What build tools do you need? How do you handle development vs production? TypeScript compilation, type definitions, and proper exports all need consideration.

## The Solution

Structure your package for development and production. Use TypeScript for type safety, configure builds for distribution, and design APIs that are simple to use but flexible enough for real needs.

**Package structure:**

```
your-package/
├── src/
│   └── index.ts          # Source code
├── dist/                 # Build output (gitignored)
├── package.json
├── tsconfig.json
└── README.md
```

**Source implementation:**

```typescript
// src/index.ts
import { pipeline } from '@huggingface/transformers'

// Singleton pattern for lazy loading
class ServiceSingleton {
  private static instance: Promise<any> | null = null
  private static readonly config = {
    // Your configuration here
  }

  static async getInstance(): Promise<any> {
    if (!this.instance) {
      // Initialize your service here
      this.instance = Promise.resolve({
        process: (input: string) => {
          // Your processing logic
          return { result: input }
        },
      })
    }
    return this.instance
  }
}

// Preserve instance in development to survive hot reloads
const getServiceSingleton = () => {
  if (process.env.NODE_ENV !== 'production') {
    if (!(global as any).ServiceSingleton) {
      ;(global as any).ServiceSingleton = ServiceSingleton
    }
    return (global as any).ServiceSingleton
  }
  return ServiceSingleton
}

/**
 * Your main exported function
 * @param input - Input parameter
 * @returns Promise resolving to result
 */
export async function yourFunction(input: string): Promise<any> {
  const Service = getServiceSingleton()
  const service = await Service.getInstance()
  const result = await service.process(input)

  // Process and validate result
  if (!result) {
    throw new Error('Processing failed')
  }

  return result
}
```

**TypeScript configuration:**

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2017",
    "module": "ESNext",
    "lib": ["ES2017"],
    "moduleResolution": "bundler",
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

**Package configuration:**

```json
// package.json
{
  "name": "your-package-name",
  "version": "1.0.0",
  "description": "Your package description",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "prepublishOnly": "tsc"
  },
  "files": ["dist"],
  "dependencies": {
    // Your dependencies
  },
  "devDependencies": {
    "@types/node": "25.0.2",
    "typescript": "5.9.3"
  }
}
```

**Design decisions:**

1. **Singleton pattern** - Expensive resources load once and reuse
2. **Global variable in dev** - Next.js hot reload clears module state, global survives reloads
3. **Zero configuration** - Sensible defaults, no options needed
4. **Type safety** - Full TypeScript support with declaration files
5. **Build output only** - Source files excluded via `.npmignore`

**Build process:**

```bash
# Development
yarn build        # Compile TypeScript
```

The `prepublishOnly` script in `package.json` ensures the package builds before publishing. The build outputs compiled JavaScript, type definitions, and source maps to the `dist/` directory.

## Benefits

- **Type safety** - Full TypeScript support for consumers
- **Simple API** - Clean interface, zero configuration
- **Production ready** - Proper builds, type definitions, source maps
- **Developer experience** - Hot reload support, fast builds
- **Maintainable** - Clear structure, well-documented

Next: [publishing-npm-package.md](./publishing-npm-package.md)
