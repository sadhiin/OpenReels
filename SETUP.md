# OpenReels — TypeScript Project Setup Guide

The complete tooling blueprint for maximum DX.

---

## Stack Decision Summary

| Layer | Choice | Why |
|-------|--------|-----|
| **Runtime** | Node.js 22 LTS | Stable subprocess handling (FFmpeg, Remotion). Bun has known FFmpeg pipe issues. |
| **Package manager** | pnpm | Best monorepo support, strict dependency isolation, fast |
| **TS execution (dev)** | tsx | Zero-config, 25x faster than ts-node, ~20ms startup |
| **Bundler (dist)** | tsup | esbuild-based, simple config, produces clean CJS/ESM output |
| **Testing** | Vitest | 3-5x faster than Jest, zero-config TS, ESM-first |
| **Lint + Format** | Biome | 10-25x faster than ESLint+Prettier, single binary, single config file |
| **CLI framework** | Commander.js | 0 deps, 500M weekly downloads, fast startup, simple API |
| **Terminal output** | chalk + ora | De facto standard for colors + spinners |
| **Schema validation** | Zod | Most mature TS schema lib, excellent JSON Schema generation (needed for LLM tool-use) |
| **Template engine** | Nunjucks | Direct Jinja2 port — existing prompt templates work with zero changes |
| **YAML parsing** | yaml (eemeli/yaml) | Actively maintained (js-yaml is abandoned), comment-preserving |
| **Monorepo** | pnpm workspaces + Turborepo | Lightweight workspace isolation + build caching |
| **Remotion** | @remotion/renderer (programmatic) | Library call instead of subprocess — the whole point of the rewrite |

---

## Project Structure

```
openreels/
├── packages/
│   ├── cli/                          # CLI entry point + pipeline orchestrator
│   │   ├── src/
│   │   │   ├── index.ts              # CLI entry (Commander.js)
│   │   │   ├── pipeline.ts           # Async pipeline orchestrator
│   │   │   ├── config.ts             # Config loader (YAML + env + CLI precedence)
│   │   │   ├── preflight.ts          # Environment validation
│   │   │   ├── progress.ts           # chalk + ora progress display
│   │   │   ├── agents/
│   │   │   │   ├── researcher.ts     # Web-grounded research agent
│   │   │   │   ├── creative-director.ts
│   │   │   │   └── critic.ts
│   │   │   ├── generators/
│   │   │   │   ├── script.ts         # Script generation
│   │   │   │   ├── tts.ts            # TTS orchestration + boundary detection
│   │   │   │   ├── visuals.ts        # Image generation + prompt optimization
│   │   │   │   └── subtitles.ts      # SRT generation
│   │   │   └── providers/
│   │   │       ├── llm.ts            # Anthropic SDK wrapper
│   │   │       ├── tts.ts            # ElevenLabs REST wrapper
│   │   │       └── image.ts          # Gemini SDK wrapper
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   ├── renderer/                     # Remotion video composition
│   │   ├── src/
│   │   │   ├── index.ts              # registerRoot
│   │   │   ├── Root.tsx              # Composition registration
│   │   │   ├── VideoComposition.tsx  # Main composition
│   │   │   ├── CaptionOverlay.tsx    # Word-by-word animated captions
│   │   │   ├── transitions.ts        # Archetype → transition mapping
│   │   │   └── render.ts            # Programmatic render API (bundle + renderMedia)
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   └── shared/                       # Shared types + utilities
│       ├── src/
│       │   ├── types.ts              # All Zod schemas + inferred types
│       │   ├── manifest.ts           # VideoManifest schema (single source of truth)
│       │   ├── template.ts           # Nunjucks template renderer
│       │   ├── archetypes.ts         # Archetype loader
│       │   ├── playbook.ts           # Playbook section extractor
│       │   └── languages/
│       │       ├── index.ts          # Language pack loader
│       │       ├── bn/
│       │       │   ├── config.yaml
│       │       │   └── playbook.md
│       │       └── en/
│       │           ├── config.yaml
│       │           └── playbook.md
│       ├── package.json
│       └── tsconfig.json
│
├── prompts/                          # Prompt templates (shared, not in a package)
│   ├── researcher.md
│   ├── creative-director.md
│   ├── script-writer.md
│   ├── image-prompter.md
│   ├── critic.md
│   └── archetypes/
│       ├── cinematic-documentary.md
│       ├── bold-illustration.md
│       ├── bengali-watercolor.md
│       ├── moody-cinematic.md
│       ├── warm-editorial.md
│       └── dramatic-caricature.md
│
├── config/
│   └── default.yaml                  # Default configuration
│
├── turbo.json                        # Turborepo pipeline config
├── pnpm-workspace.yaml               # Workspace definition
├── biome.json                        # Linting + formatting
├── tsconfig.base.json                # Shared TS compiler options
├── package.json                      # Root package.json
├── .env.example
├── LICENSE
├── README.md
└── PRODUCT_SPEC.md
```

---

## Step-by-Step Setup

### 1. Initialize the monorepo

```bash
mkdir openreels && cd openreels
git init
pnpm init

# Create workspace file
cat > pnpm-workspace.yaml << 'EOF'
packages:
  - "packages/*"
EOF
```

### 2. Root package.json

```json
{
  "name": "openreels",
  "private": true,
  "packageManager": "pnpm@9.15.0",
  "engines": {
    "node": ">=22.0.0"
  },
  "scripts": {
    "dev": "pnpm --filter @openreels/cli dev",
    "build": "turbo build",
    "test": "turbo test",
    "lint": "biome check .",
    "lint:fix": "biome check --write .",
    "format": "biome format --write .",
    "typecheck": "turbo typecheck",
    "clean": "turbo clean"
  },
  "devDependencies": {
    "@biomejs/biome": "^2.0.0",
    "turbo": "^2.4.0",
    "typescript": "^5.7.0"
  }
}
```

### 3. TypeScript base config

```jsonc
// tsconfig.base.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "noUncheckedIndexedAccess": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": false
  }
}
```

### 4. Turborepo config

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "dev": {
      "dependsOn": ["^build"],
      "cache": false,
      "persistent": true
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": []
    },
    "typecheck": {
      "dependsOn": ["^build"],
      "outputs": []
    },
    "clean": {
      "cache": false
    }
  }
}
```

### 5. Biome config

```json
// biome.json
{
  "$schema": "https://biomejs.dev/schemas/2.0.0/schema.json",
  "vcs": {
    "enabled": true,
    "clientKind": "git",
    "useIgnoreFile": true
  },
  "organizeImports": {
    "enabled": true
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "complexity": {
        "noExcessiveCognitiveComplexity": "warn"
      },
      "correctness": {
        "noUnusedImports": "error",
        "noUnusedVariables": "error"
      },
      "style": {
        "noNonNullAssertion": "warn",
        "useConst": "error"
      }
    }
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "double",
      "semicolons": "always",
      "trailingCommas": "all"
    }
  },
  "files": {
    "ignore": ["node_modules", "dist", ".turbo", "*.md", "*.yaml", "*.yml"]
  }
}
```

### 6. Shared package

```bash
mkdir -p packages/shared/src
cd packages/shared
pnpm init
```

```json
// packages/shared/package.json
{
  "name": "@openreels/shared",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    }
  },
  "scripts": {
    "build": "tsup",
    "dev": "tsup --watch",
    "typecheck": "tsc --noEmit",
    "clean": "rm -rf dist"
  },
  "dependencies": {
    "zod": "^3.24.0",
    "nunjucks": "^3.2.4",
    "yaml": "^2.7.0"
  },
  "devDependencies": {
    "@types/nunjucks": "^3.2.6",
    "tsup": "^8.4.0",
    "typescript": "^5.7.0"
  }
}
```

```typescript
// packages/shared/tsup.config.ts
import { defineConfig } from "tsup";

export default defineConfig({
  entry: ["src/index.ts"],
  format: ["esm"],
  dts: true,
  clean: true,
  sourcemap: true,
});
```

```jsonc
// packages/shared/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src/**/*"]
}
```

### 7. CLI package

```bash
mkdir -p packages/cli/src
cd packages/cli
pnpm init
```

```json
// packages/cli/package.json
{
  "name": "@openreels/cli",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "bin": {
    "openreels": "./dist/index.js"
  },
  "scripts": {
    "dev": "tsx src/index.ts",
    "build": "tsup",
    "test": "vitest run",
    "test:watch": "vitest",
    "typecheck": "tsc --noEmit",
    "clean": "rm -rf dist"
  },
  "dependencies": {
    "@openreels/shared": "workspace:*",
    "@openreels/renderer": "workspace:*",
    "@anthropic-ai/sdk": "^0.40.0",
    "commander": "^13.0.0",
    "chalk": "^5.4.0",
    "ora": "^8.2.0",
    "dotenv": "^16.4.0",
    "sharp": "^0.33.0"
  },
  "devDependencies": {
    "tsx": "^4.19.0",
    "tsup": "^8.4.0",
    "typescript": "^5.7.0",
    "vitest": "^3.0.0"
  }
}
```

```typescript
// packages/cli/tsup.config.ts
import { defineConfig } from "tsup";

export default defineConfig({
  entry: ["src/index.ts"],
  format: ["esm"],
  dts: true,
  clean: true,
  sourcemap: true,
  banner: {
    js: "#!/usr/bin/env node",
  },
});
```

```jsonc
// packages/cli/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src/**/*"]
}
```

### 8. Renderer package

```bash
mkdir -p packages/renderer/src
cd packages/renderer
pnpm init
```

```json
// packages/renderer/package.json
{
  "name": "@openreels/renderer",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    }
  },
  "scripts": {
    "build": "tsup",
    "dev": "tsup --watch",
    "preview": "remotion preview",
    "test": "vitest run",
    "typecheck": "tsc --noEmit",
    "clean": "rm -rf dist"
  },
  "dependencies": {
    "@openreels/shared": "workspace:*",
    "remotion": "^4.0.0",
    "@remotion/cli": "^4.0.0",
    "@remotion/renderer": "^4.0.0",
    "@remotion/transitions": "^4.0.0",
    "@remotion/media-utils": "^4.0.0",
    "react": "^18.3.0",
    "react-dom": "^18.3.0"
  },
  "devDependencies": {
    "@types/react": "^18.3.0",
    "tsup": "^8.4.0",
    "typescript": "^5.7.0",
    "vitest": "^3.0.0"
  }
}
```

```jsonc
// packages/renderer/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "jsx": "react-jsx",
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src/**/*"]
}
```

### 9. Vitest config (per package)

```typescript
// packages/cli/vitest.config.ts (same pattern for renderer)
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    globals: true,
    environment: "node",
    include: ["src/**/*.test.ts"],
    coverage: {
      provider: "v8",
      reporter: ["text", "lcov"],
    },
  },
});
```

---

## Key DX Features

### Run in development

```bash
# Run the CLI directly (no build step)
pnpm dev -- generate "topic" --lang en --verbose

# Which runs:
tsx src/index.ts generate "topic" --lang en --verbose
```

### Build all packages

```bash
pnpm build
# Turborepo caches outputs — rebuilds only what changed
```

### Test all packages

```bash
pnpm test
# Or watch mode in a specific package:
pnpm --filter @openreels/cli test:watch
```

### Lint + format everything

```bash
pnpm lint          # Check
pnpm lint:fix      # Auto-fix
pnpm format        # Format
```

### Type-check everything

```bash
pnpm typecheck
```

---

## Remotion: Library vs Subprocess

The key DX win of the rewrite. Instead of:

```python
# Old: subprocess call, parse stderr for progress
process = subprocess.Popen(["npx", "remotion", "render", ...])
for line in process.stderr:
    progress = regex_parse(line)
```

You get:

```typescript
// New: direct function call, typed progress callback
import { bundle } from "@remotion/bundler";
import { renderMedia } from "@remotion/renderer";

const bundled = await bundle({ entryPoint: "./src/index.ts" });

await renderMedia({
  composition,
  serveUrl: bundled,
  codec: "h264",
  outputLocation: "final.mp4",
  onProgress: ({ progress }) => {
    spinner.text = `Rendering: ${Math.round(progress * 100)}%`;
  },
});
```

No symlinks. No subprocess. No stderr parsing. No timeout management. Typed errors. In-process progress callbacks.

---

## Dependency Summary

### Production dependencies (across all packages)

| Package | Purpose | Size |
|---------|---------|------|
| `@anthropic-ai/sdk` | Claude API (LLM) | ~50KB |
| `commander` | CLI framework | 0 deps, ~30KB |
| `chalk` | Terminal colors | ~10KB |
| `ora` | Terminal spinners | ~15KB |
| `zod` | Schema validation | ~12KB |
| `nunjucks` | Jinja2-compatible templates | ~40KB |
| `yaml` | YAML parsing | ~20KB |
| `dotenv` | .env loading | ~5KB |
| `sharp` | Image resize/crop | native addon |
| `remotion` + `@remotion/*` | Video rendering | ~2MB |
| `react` + `react-dom` | Remotion dependency | ~45KB |

### Dev dependencies

| Package | Purpose |
|---------|---------|
| `typescript` | Type checking |
| `tsx` | TS execution in dev |
| `tsup` | Build/bundle |
| `vitest` | Testing |
| `@biomejs/biome` | Lint + format |
| `turbo` | Monorepo task runner |

---

## Why These Choices

**Zod over ArkType**: ArkType is faster but Zod's `.openapi()` / `.jsonSchema()` is critical for generating JSON schemas for Anthropic's tool-use structured outputs. ArkType doesn't have this yet.

**Nunjucks over ETA**: ETA is faster but our prompt templates are already Jinja2 syntax. Nunjucks is a direct Jinja2 port — templates work with zero changes.

**Commander over Clipanion**: Commander is simpler, has 0 dependencies, and 500M weekly downloads. Clipanion's type-safety is nice but overkill for a CLI with ~5 commands.

**Node.js over Bun**: Bun has known FFmpeg subprocess pipe issues (bun#17989). Since we spawn FFmpeg as a fallback and Remotion internally uses Chromium subprocesses, Node.js 22 is the safer choice.

**pnpm over npm/yarn**: Strict dependency isolation prevents Remotion's React version from conflicting with CLI dependencies. Content-addressable store saves disk space.

**Biome over ESLint+Prettier**: 10-25x faster, single config file, single binary. For a new project with no existing ESLint plugin dependencies, there's no reason to use the slower option.
