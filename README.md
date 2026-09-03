# effect-paths

This provides helpers for working with paths in [Effect-TS](https://effect.website). Specifically, it provides branded types for absolute vs relative paths, and for directories vs files. We also extend Node types (`fs`, `fs/promises`, `path`, etc.) to use these branded types.

 This uses Effect v4 RC, although it may also work with v3, I haven't tried.

If you just want the types without the Effect dependency, import from `effect-paths/vanilla` instead.

## Installation

```bash
npm install effect-paths
```

## Example

```ts
import fs from "node:fs";
import path from "node:path";

import {
  type AbsoluteDir,
  type AbsoluteFile,
  type AbsolutePath,
  RelativeDir,
  RelativeFile,
  type RelativePath,
} from "effect-paths";

// process.cwd() now returns AbsoluteDir
process.cwd() satisfies AbsoluteDir;

// template literals are supported
RelativeFile("package.json") satisfies RelativeFile<`${string}.json`>;

/* ------------------------------ path.join examples ------------------------------ */
// joining absolute dir to relative file/dir gives absolute file/dir
path.join(process.cwd(), RelativeFile("package.json")) satisfies AbsoluteFile;
path.join(process.cwd(), RelativeDir("config")) satisfies AbsoluteDir;

// AbsolutePath = AbsoluteDir | AbsoluteFile
path.join(
  process.cwd(),
  RelativeDir("config") as RelativePath,
) satisfies AbsolutePath;

// joining relative dir to relative file/dir gives relative file/dir
path.join(RelativeDir("a"), RelativeDir("b")) satisfies RelativeDir;
path.join(
  RelativeDir("a"),
  RelativeFile("package.json"),
) satisfies RelativeFile;

// RelativePath = RelativeDir | RelativeFile
path.join(
  RelativeDir("a"),
  RelativeFile("package.json") as RelativePath,
) satisfies RelativePath;

// we also override the types to give error messages for invalid combinations.
// of course path.join still returns a string here, but we change it to Error
// and add a deprecation warning to flag this at the type level
path.join(RelativeFile("x"), RelativeFile("a")) satisfies Error & {
  message: "You can only pass files in the last position";
};

/* ------------------------------ fs examples ------------------------------ */
// passing a plain string produces string[] as usual
fs.readdirSync(".");

// passing an AbsoluteDir produces RelativePath[]
fs.readdirSync(process.cwd()) satisfies RelativePath[];

// with `withFileTypes: true`, Dirent gets proper typing
const entries = fs.readdirSync(process.cwd(), { withFileTypes: true });
const entry = entries[0]!;

entry.parentPath satisfies AbsoluteDir;
entry.name satisfies RelativePath;
```

## Types

These are the branded types for relative/absolute files/directories:


| Types    | File           | Directory     | Either         |
|----------|----------------|---------------|----------------|
| Relative | `RelativeFile` | `RelativeDir` | `RelativePath` |
| Absolute | `AbsoluteFile` | `AbsoluteDir` | `AbsolutePath` |
| Either   | `AnyFile`      | `AnyDir`      | `AnyPath`      |

Additional types:

- `Dirent` - customized version of [`fs.Dirent`](https://nodejs.org/api/fs.html#class-fsdirent) that uses branded types for `name` and `parentPath`, and narrows the type of `name` based on `isDirectory()` and `isFile()`
- `FileExtn` - branded type for file extensions
- `AbsoluteToRelative<P extends AbsolutePath>` - utility to convert `AbsoluteFile` -> `RelativeFile` and `AbsoluteDir` -> `RelativeDir`
- `RelativeToAbsolute<P extends RelativePath>` - utility to convert `RelativeFile` -> `AbsoluteFile` and `RelativeDir` -> `AbsoluteDir`

## Schemas

Schemas for the above types:

| Schemas  | File                 | Directory           | Either               |
|----------|----------------------|---------------------|----------------------|
| Relative | `SchemaRelativeFile` | `SchemaRelativeDir` | `SchemaRelativePath` |
| Absolute | `SchemaAbsoluteFile` | `SchemaAbsoluteDir` | `AbsolutePath`       |
| Either   | `SchemaAnyFile`      | `SchemaAnyDir`      | `SchemaAnyPath`      |

as well as `SchemaFileExtn`.

## Overrides

### fs

- `readdirSync()`
  - returns `RelativePath[]` when passed an `AbsoluteDir`
  - returns an Error when passed an `AnyPath` which is not an `AbsoluteDir`
  - returns `string[]` as usual when passed a plain string
  - returns branded `Dirent[]` when passed `withFileTypes: true`

- `watch()`: when passed an `AbsoluteDir`, the callback receives `RelativePath` instead of `string`

### fs/promises

Same as above but with Promises

### path

- `basename()` returns `RelativeDir` when passed `AnyDir`, and `RelativeFile` when passed `AnyFile`
- `dirname()` returns `AbsoluteDir` when passed `AbsolutePath`, and `RelativeDir` when passed `RelativePath`
- `extname()` returns `FileExtn` when passed `AnyFile`
- `format()` returns `AbsolutePath` when passed `AbsoluteDir` for `dir` or `root`
- `isAbsolute()` narrows the type when passed an `AnyPath`
- `join()` handles all combinations of absolute/relative files/dirs, and returns an Error on invalid combinations (the Error is just in the types, not at runtime)
- `normalize()` preserves the type when passed an `AnyPath`
- `parse()` returns `AbsoluteDir` for `dir`, and `FileExtn` for `ext`
- `relative()` returns `RelativePath` when passed two `AbsolutePath`s
- `resolve()` returns `AbsolutePath` when passed an `AbsoluteDir` and a `RelativePath`

### process

- `process.cwd()` now returns `AbsoluteDir`

### url

- `fileURLToPath()` now returns `AbsolutePath`
