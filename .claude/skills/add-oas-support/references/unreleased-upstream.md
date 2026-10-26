# Unreleased Upstream

How a step starts before the ApiDOM release it needs. The ApiDOM PR must be ready: its exit gate
passes and its fresh-context review is clean. This repository then installs tarballs packed from
that PR's branch, opens its own PR as a draft, and merges only after the ApiDOM release, with the
tarballs replaced by it. Without a ready ApiDOM PR, Step 0's `UPSTREAM` still means stop.

npm can't take these packages from a git URL: ApiDOM is a private workspace monorepo, its build
output (`src/**/*.cjs`, `types/`) isn't committed, and it has no `prepare` script.

## 1. Pack the ApiDOM Branch

Keep the repositories, worktrees and tarballs side by side (`~/WebstormProjects/{apidom,
swagger-client, upstream-tarballs, ...}`), so the relative `file:` paths below resolve from every
checkout and worktree.

```bash
git -C ../apidom fetch origin <branch>
git -C ../apidom worktree add ../apidom-pr-<n> FETCH_HEAD   # its build stays out of ../apidom
cd ../apidom-pr-<n> && npm i && npm run build
sha=$(git rev-parse --short HEAD); out="$(cd .. && pwd)/upstream-tarballs"; mkdir -p "$out"
for p in $(git diff --name-only origin/main...HEAD -- packages | cut -d/ -f2 | sort -u); do
  f=$(cd "packages/$p" && npm pack --silent --pack-destination "$out")
  mv "$out/$f" "$out/${f%.tgz}-$sha.tgz"
done
```

Keep the commit in the file name. npm caches a tarball's content by the integrity in the
lockfile, so a rebuilt tarball under the same name installs the old code, with `npm ci` too.

## 2. Wire Them

For each packed package that is in this repository's tree (`npm ls <name>`) or that this PR adds
as a dependency:

- **The spec goes in the root.** `file:../upstream-tarballs/<file>.tgz` in `dependencies` when
  the package is already listed there, else in `devDependencies`.
- **A `$` override for nested copies.** Add `"<name>": "$<name>"` to `overrides`. npm refuses a
  plain override of a direct dependency (`EOVERRIDE`), and once a lockfile exists it resolves a
  relative `file:` path written inside `overrides` from `node_modules/` (`ENOENT`), so the path
  stays in the root fields.

Then `npm install` and re-run Step 0: the `UPSTREAM` lines turn `OK`, and each tarball is an
`UNRELEASED` line (`spec-status.mjs` exits 4 when nothing else is open). Commit `package.json`
and `package-lock.json` on their own:
`build(deps): use unreleased ApiDOM from swagger-api/apidom#<n> (drop before merge)`. CI can't
reach the tarballs, so the PR stays a draft until the release.

## 3. When the ApiDOM PR Changes

Pack again (the new commit gives new file names), point the specs at the new files,
`npm install`, and re-run the gates.

## 4. Before Merging

Once the ApiDOM release is out, drop the tarball commit, bump every `@swagger-api/apidom-*`
package to the release (Step 2 of `/add-oas-support`), `npm install`, and run every gate again.
`spec-status.mjs` must exit 0, with no `UNRELEASED` line. The release can differ from the branch
this PR was built on (review changes land after "ready"), so the gates run on the release, not on
the tarballs.

## Downstream

Swagger UI takes this repository's PR the same way before a swagger-client release (its
`references/unreleased-upstream.md`). If this PR uses ApiDOM tarballs, they reach Swagger UI
inside the swagger-client tarball's `package.json` as paths that only resolve here, so Swagger UI
wires those ApiDOM tarballs too; the `UNRELEASED` lines list them.
