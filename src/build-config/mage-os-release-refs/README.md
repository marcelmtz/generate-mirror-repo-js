# Release refs

Optional per-release overrides of the git ref each repository is built from.

`src/make/mageos-release.js` looks for `<vendor>-release-refs/<version>.js` in
this directory (or an explicit `--releaseRefsFile=`). When the file is absent —
the normal case — every repository is built from the ref in
`mageos-release-build-config.js`, which is the default branch.

A file here exports a map of build-config keys to git refs. The `*` key applies
to every repository:

```js
// src/build-config/mage-os-release-refs/3.4.1.js
module.exports = {
  '*': 'release/3.x',
};
```

Individual repositories can be overridden alongside the wildcard:

```js
module.exports = {
  '*': 'release/3.x',
  'security-package': 'main',
};
```

This exists so a release can be built from a line other than the default
branch — for example a patch release on the previous major while `main` has
already moved on. See `RELEASE-GRACE-WINDOW-PROPOSAL.md`.
