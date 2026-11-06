# Circular project reference silently suppresses TypeScript errors

Minimal reproduction using the unmodified native TypeScript **7.0.2** compiler.
No Oxlint, TSGolint, patches, or custom compiler host is involved.

The project contains an invalid assignment:

```ts
export const value: number = "error";
```

Its `tsconfig.json` references the project itself.

## Reproduce

With Node.js and npm installed, run from this directory:

```sh
npx --yes --package=typescript@7.0.2 tsc --project tsconfig.json --noEmit --pretty false
```

**Actual:** no diagnostics; exit code **0**.

**Expected:** an error for the invalid reference cycle, or the applicable source
diagnostic, rather than success while dropping diagnostics.

## Controls

Remove the `references` property from `tsconfig.json` and repeat the command.
The compiler now exits **1** and reports:

```text
index.ts(1,14): error TS2322: Type 'string' is not assignable to type 'number'.
```

With the original self-reference restored, build mode correctly rejects the cycle:

```sh
npx --yes --package=typescript@7.0.2 tsc --build . --dry --pretty false
```

This exits **4** with TS6202: `Project references may not form a circular graph.`

## Upstream code

The reference mapper excludes the root config by comparing parsed config object
identity. Resolving the cycle reparses the same config into a different object,
so the root project's own files enter the project-reference mappings.

See [the comparison in the upstream source corresponding to TypeScript 7.0.2](https://github.com/microsoft/typescript-go/blob/2bd066d87f5bafd315be9f40889d0a60b9e58e0b/internal/compiler/projectreferenceparser.go#L95).
Comparing normalized config paths instead preserves root-project diagnostics.
