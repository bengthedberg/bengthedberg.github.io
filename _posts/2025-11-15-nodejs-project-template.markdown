---
layout: post
title: "Setting Up a Production Node.js Project with TypeScript"
date: 2025-11-15 00:00:00 +0000
tags:
  - nodejs
  - typescript
  - jest
  - eslint
  - prettier
  - project-setup
---


# Setting Up a Production Node.js Project with TypeScript

Starting a new Node.js project from scratch means making dozens of tooling decisions -- TypeScript configuration, test runner, linter, formatter, dev server, git hooks. Get it wrong and you will be fighting your tools instead of shipping features.

This guide walks through setting up a production-ready Node.js project template with TypeScript, Jest, ESLint, Prettier, Nodemon, and Husky. By the end you will have a solid foundation you can reuse for any new project.

## Step 1: Initialize the Project with TypeScript

Start by creating a new Node.js project and adding TypeScript:

```bash
npm init -y
npm install typescript ts-node @types/node @tsconfig/node22 --save-dev
```

The packages:

- **typescript** -- the TypeScript compiler
- **ts-node** -- run TypeScript files directly without a separate compile step
- **@types/node** -- type definitions for Node.js APIs
- **@tsconfig/node22** -- community-maintained base TypeScript configuration for Node.js 22

### Configure TypeScript

Create `tsconfig.json`:

```json
{
  "extends": "@tsconfig/node22/tsconfig.json",
  "compilerOptions": {
    "rootDir": "./" ,
    "outDir": "./dist",
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

This extends the recommended Node.js 22 base configuration and adds sensible defaults -- source files live in `src/`, compiled output goes to `dist/`.

## Step 2: Add Jest for Testing

```bash
npm install jest ts-jest @types/jest --save-dev
```

Create `jest.config.ts`:

```ts
import type { Config } from "jest";

const config: Config = {
  preset: "ts-jest",
  testEnvironment: "node",
  roots: ["<rootDir>/tests"],
  transform: {
    "^.+\\.tsx?$": "ts-jest",
  },
  testRegex: "((\\.|/)(test|spec))\\.tsx?$",
  moduleFileExtensions: ["ts", "tsx", "js", "jsx", "json", "node"],
};

export default config;
```

### Verify the Setup

Add a simple source file `src/utils.ts`:

```ts
export function add(a: number, b: number) {
  return a + b;
}
```

And a test file `tests/add.test.ts`:

```ts
import { add } from "../src/utils";

it("should add two numbers", () => {
  expect(add(1, 2)).toBe(3);
});
```

To enforce TypeScript in tests, add a `tests/tsconfig.json`:

```json
{
  "extends": "@tsconfig/node22/tsconfig.json"
}
```

Add the build and test scripts to `package.json`:

```json
{
  "scripts": {
    "build": "tsc",
    "test": "jest",
    "test:watch": "jest --watch"
  }
}
```

Run `npm run test` to confirm everything works:

```
PASS  tests/add.test.ts
  ✓ should add two numbers (1 ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
```

## Step 3: Configure ESLint and Prettier

### Install ESLint

```bash
npm init @eslint/config@latest
```

Select the following options when prompted:

- How would you like to use ESLint? **To check syntax and find problems**
- What type of modules? **ESM**
- Which framework? **None**
- Does your project use TypeScript? **Yes**
- Where does your code run? **Browser, Node**
- Install dependencies? **Yes**
- Package manager? **npm**

### Configure ESLint

Update `eslint.config.mjs`:

```js
import globals from "globals";
import pluginJs from "@eslint/js";
import tseslint from "typescript-eslint";

export default [
  {
    ignores: ["dist/"],
  },
  {
    files: ["src/**/*.{js,mjs,cjs,ts}", "tests/**/*.{js,mjs,cjs,ts}"],
  },
  {
    files: ["**/*.js"],
    languageOptions: { sourceType: "commonjs" },
  },
  {
    languageOptions: {
      globals: { ...globals.browser, ...globals.node },
    },
  },
  pluginJs.configs.recommended,
  ...tseslint.configs.recommended,
];
```

### Add ESLint Plugin for Jest

```bash
npm install eslint-plugin-jest --save-dev
```

Add the Jest configuration to `eslint.config.mjs`:

```js
import jest from "eslint-plugin-jest";

// Add to the export array:
{
  files: ["tests/**/*.{js,ts,jsx,tsx}"],
  ...jest.configs["flat/recommended"],
  rules: {
    ...jest.configs["flat/recommended"].rules,
    "jest/prefer-expect-assertions": "off",
  },
},
```

### Install and Configure Prettier

```bash
npm install --save-dev eslint-plugin-prettier eslint-config-prettier
npm install --save-dev --save-exact prettier
```

Add Prettier to `eslint.config.mjs`:

```js
import eslintPluginPrettierRecommended from "eslint-plugin-prettier/recommended";

// Add to the export array:
{
  rules: {
    "@typescript-eslint/no-unused-vars": "off",
    "prettier/prettier": [
      "error",
      {
        endOfLine: "auto",
      },
    ],
  },
},
eslintPluginPrettierRecommended,
```

### Add Lint Scripts

Update `package.json`:

```json
{
  "scripts": {
    "build": "tsc",
    "test": "jest",
    "test:watch": "jest --watch",
    "lint": "eslint",
    "lint:fix": "eslint --fix"
  }
}
```

Run `npm run lint` to check for issues and `npm run lint:fix` to auto-fix them.

## Step 4: Add Development Tools

### Nodemon for Auto-Reload

```bash
npm install --save-dev nodemon
```

Create `nodemon.json`:

```json
{
  "watch": ["src"],
  "ext": "js,ts,json",
  "ignore": ["src/**/*.test.ts"],
  "exec": "ts-node ./src/index.ts"
}
```

### Environment Variables with dotenv-cli

```bash
npm install --save-dev dotenv-cli
```

Create a `.env` file:

```
APP_DEBUG=true
```

Create a `.env.sample` file (committed to git as documentation):

```
APP_DEBUG=true
```

Add an entry point `src/index.ts`:

```ts
import { add } from "./utils";

console.log("sum", add(1, 3));
console.log("debug", process.env.APP_DEBUG);
```

### Final Scripts

Update `package.json` with all scripts:

```json
{
  "scripts": {
    "build": "tsc",
    "dev": "dotenv -- nodemon",
    "test": "jest",
    "test:watch": "jest --watch",
    "lint": "eslint",
    "lint:fix": "eslint --fix",
    "prepare": "husky",
    "format": "prettier --write src/."
  }
}
```

## Step 5: Git Hooks with Husky

Enforce code quality on every commit with [Husky](https://typicode.github.io/husky/):

```bash
npm install --save-dev husky
npx husky init
```

Update `.husky/pre-commit` to run formatting, linting, and tests before each commit:

```bash
npm run format
npm run lint
npm run test
```

## Step 6: Git and .gitignore

Set up version control:

```bash
wget -O .gitignore https://raw.githubusercontent.com/github/gitignore/main/Node.gitignore
git init --initial-branch=main
git add .
git commit -m 'Initial Project Setup'
```

## Optional: Add Express

If you are building a web API, add Express:

```bash
npm install express serverless-http
npm install @types/node @types/express --save-dev
```

The `serverless-http` package makes it easy to deploy your Express app to AWS Lambda or similar serverless platforms later.

## VS Code Integration

For the best development experience, add recommended extensions and settings via `.vscode/extensions.json` and `.vscode/settings.json` in your project root. Key extensions to include:

- **ESLint** -- inline linting feedback
- **Prettier** -- format on save
- **Jest** -- test runner integration

## Conclusion

With this template you get a consistent, production-ready starting point for any Node.js project:

- **TypeScript** for type safety
- **Jest** for testing
- **ESLint + Prettier** for consistent code style
- **Nodemon** for fast development iteration
- **dotenv-cli** for environment variable management
- **Husky** for automated quality gates on commit

Clone this setup, strip out the sample code, and start building.

## References

- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/)
- [Jest Documentation](https://jestjs.io/docs/getting-started)
- [ESLint Configuration](https://eslint.org/docs/latest/use/configure/)
- [Prettier Documentation](https://prettier.io/docs/en/)
- [Husky Documentation](https://typicode.github.io/husky/)
- [Nodemon](https://nodemon.io/)
- [tsconfig/bases](https://github.com/tsconfig/bases)
