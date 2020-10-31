### Working with `npm link`

1. `cd ./projects/some-dep`
2. `npm link`
3. `cd ./projects/my-app`
4. `npm link some-dep` (creates a symlink to the dependency)

The synbolic link is local and as such any changes won't be committed to git.

When you're done with the symlink and want to return to normal operating order:

1. `cd ./projects/my-app`
2. `nmp unlink --no-save some-dep && npm install`
3. `cd ./projects/some-dep`
4. `npm uninstall` (removes global symlink)
