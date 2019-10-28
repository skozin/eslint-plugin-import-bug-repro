## Minimal reproduction of a bug in the `eslint-plugin-import` package

This repo reproduces a bug that leads to external scoped packages being determined
as internal ones, producing incorrect linting output.
See https://github.com/benmosher/eslint-plugin-import/issues/1597.

### Steps to reproduce

1. Clone this repository

```text
git clone https://github.com/skozin/eslint-plugin-import-bug-repro.git
```

2. Run `yarn` to install dependencies

```text
yarn
```

3. Lint [`packages/package-b/index.js`] file:

```text
yarn lint
```

This command actually does this:

```text
./node_modules/.bin/eslint packages/package-b/index.js
```

### Expected result

According to [`.eslintrc`] config, [`packages/package-b/index.js`] should produce no errors
or warnings. Here is the contents of the file:

```js
import lodash from 'lodash';
import {sharedFn} from '@my-scope/package-a';

import {someFn} from './util';

export default {};
```

### Actual result

However, the following errors are shown:

```text
eslint-plugin-import-bug-repro/packages/package-b/index.js
  1:1  error  There should be at least one empty line between import groups        import/order
  4:1  error  `./util` import should occur before import of `@my-scope/package-a`  import/order

✖ 2 problems (2 errors, 0 warnings)
  2 errors and 0 warnings potentially fixable with the `--fix` option.
```

These errors disappear if the Node resolver is used instead of the Webpack resolver.

### Details of the setup

For the problem to occur, the following conditions should be met:

#### 1. The project should use [Yarn workspaces].

It is a popular monorepo setup, e.g. used by Babel, which allows you to have multiple packages
in a single repo and import them using their name, as if they were published to a registry.
From Yarn documentation:

> Your dependencies can be linked together, which means that your workspaces can depend on one
> another while always using the most up-to-date code available. This is also a better mechanism
> than yarn link since it only affects your workspace tree rather than your whole system.

Yarn does this by creating filesystem symlinks from `node_modules` directory into actual directories
the packages reside in. Run `ls -l node_modules/@my-scope` to see this:

```text
$ ls -l node_modules/@my-scope
total 0
lrwxr-xr-x  1 user  group  24 Oct 28 17:28 package-a -> ../../packages/package-a
lrwxr-xr-x  1 user  group  24 Oct 28 17:28 package-b -> ../../packages/package-b
```

In this repository, all packages reside in [`packages`] directory, as configured by
`workspaces` key in [`package.json`].

#### 2. The packages inside a monorepo should be scoped.

In this case, they use `@my-scope` prefix.

#### 3. Webpack resolver should be used.

When Node resolver is used, the behavior is correct. You can check this by removing this line from
[`.eslintrc`]:

```js
...
  "import/resolver": "webpack",
...
```

Please note that the [`packages`] directory is added to external directories list in [`.eslintrc`].
Nevertheless, scoped packages that reside in this directory are determined as internal.

### The cause

When all of the above conditions are met, the following happens.

1. The plugin calls [`resolveImportType`] function to get import type of the `@my-scope/package-a`
dependency.

2. The `resolveImportType` function calls [`resolve`] function from `eslint-module-utils` package.
It, in turn, uses the Webpack resolver, which returns **the full path to the module main file, after
resolving all symlinks**. In our case, it is `/path/to/the/project/packages/package-a/index.js`.
This is in contrast to the Node resolver, which in this case would return
`/path/to/the/project/node_modules/@my-scope/package-a/index.js`.

3. The `resolveImportType` function then calls [`typeTest`], leading to the following call
stack: [`resolveImportType`] -> [`typeTest`] -> [`isInternalModule`] -> [`isExternalPath`]. The last
function should return `true`, since `packages` directory is contained in the external directories
list.

4. The `isExternalPath` function extracts part of the import declaration before the first slash and
considers it the package name. Then, for each of the configured external module folders, it tries
to find substring `${external-folder}/${package-name}` inside the filesystem path to the module. If it
succeeds, then the path is considered external. In our case, since the import declaration is
`import {sharedFn} from '@my-scope/package-a'`, the `packageName` variable gets set to `@my-scope`,
and the loop on the next line looks for the `node_modules/@my-scope` or `packages/@my-scope`
substring inside the FS path to the module (which is `/path/to/the/project/packages/package-a/index.js`
in the case of the Webpack resolver) and fails to do so.

5. So, `isExternalPath` returns `false`, and this makes the logic inside `isInternalModule`
decide that the package is internal. Previously, `isInternalModule` would consider all scoped
packages external, but some time ago [an issue was submitted][issue2], reporting incorrect behavior
of the plugin when Webpack aliases starting with `@` were used: those aliased imports were
incorrectly determined as external. [The corresponding PR][PR2] fixed this by changing the logic
so that it would always consider scoped imports as internal except when they correspond to external
filesystem paths.

### How to fix

This is pretty complicated actually.

The first problem here is that [`isExternalPath`] incorrectly determines package name for scoped
packages: it sets the `packageName` variable by extracting the part of the import name before
the first slash. But this is incorrect for scoped package names since that part corresponds to the
scope name and not package name. For scoped packages, the package name is the part of the import
name before the second slash (or the end of the string if the number of slashes is less than two).
For example, in our case, for import declaration `import {sharedFn} from '@my-scope/package-a'`
the package name is `@my-scope/package-a` and not `@my-scope`.

But even fixing this won't make the plugin support the case described here since Webpack resolver
returns the fully resolved path to the module's main file (`/path/to/the/project/packages/package-a/index.js`)
which doesn't contain `@my-scope` part.

So, I see two options.

The first one is to make [`isExternalPath`] return `true` for all paths that contain one of the
`import/external-module-folders` as a sub-path, regardless of the name of the import.
According to the documentation, this seems to be the expected behavior:

> `import/external-module-folders` An array of folders. Resolved modules only from those folders
will be considered as "external". By default - `["node_modules"]`. Makes sense if you have
configured your path or webpack to handle your internal paths differently and want to considered
modules from some folders, for example `bower_components` or `jspm_modules`, as "external".

The risk I see here is that the stuff with checking the import name in `isExternalPath` was added
to work around some edge case I don't yet know of, so removing this seemingly unneeded logic might
break something.

The second, safer, way is to add another configuration option, e.g. `import/workspace-module-folders`,
and make [`isExternalPath`] return `true` also when a path to the module contains one of these paths
as a sub-path, in addition to the current logic. This will guarantee that none of the existing setups
are broken, however at the cost of adding an extra configuration option.


[`packages/package-b/index.js`]: packages/package-b/index.js
[`.eslintrc`]: .eslintrc
[Yarn workspaces]: https://yarnpkg.com/lang/en/docs/workspaces
[`packages`]: packages
[`package.json`]: package.json
[`resolveImportType`]: https://github.com/benmosher/eslint-plugin-import/blob/cd25d9d/src/core/importType.js#L91
[`resolve`]: https://github.com/benmosher/eslint-plugin-import/blob/cd25d9d/utils/resolve.js#L213
[`typeTest`]: https://github.com/benmosher/eslint-plugin-import/blob/cd25d9d/src/core/importType.js#L75
[`isInternalModule`]: https://github.com/benmosher/eslint-plugin-import/blob/cd25d9d/src/core/importType.js#L56
[`isExternalPath`]: https://github.com/benmosher/eslint-plugin-import/blob/cd25d9d/src/core/importType.js#L27
[issue2]: https://github.com/benmosher/eslint-plugin-import/issues/1293
[PR2]: https://github.com/benmosher/eslint-plugin-import/pull/1294
