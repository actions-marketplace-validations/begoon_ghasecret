# CLAUDE.md

Repo-specific gotchas for future sessions. The README explains *what* the action does and how to use it — this file only captures things that are easy to break by accident.

## What this repo is

A composite GitHub Action published as `begoon/ghasecret`. The whole action is `action.yml` + `dump.sh` (bash). No build step, no `node_modules`, no `dist/`. Do not introduce a JS/Docker rewrite without an explicit user request.

## Triple base64 is load-bearing

`dump.sh` encodes each secret with `base64 | tr -d '\n' | base64 | tr -d '\n' | base64 | tr -d '\n'`. All three rounds are necessary — GitHub Actions masks both the raw secret and its single-base64 form. Do not "simplify" to one or two rounds; the tests will pass locally but the action will be useless inside a real workflow.

The intermediate `tr -d '\n'` is also intentional: it keeps the blob compact and on a single line so the printed `decode:` one-liner stays paste-able. `base64 -d` itself ignores whitespace, so dropping the `tr -d` would not break the round trip — but it would inflate the blob and could push the `decode:` line into multi-line wrapping in the GitHub log UI.

## CI test parses dump.sh output — keep them in sync

`.github/workflows/test.yml` extracts the encoded blob with:

```bash
awk '/^value:$/ { getline; print; exit }'
```

This depends on `dump.sh` printing exactly the line `value:` followed by the blob on the next line. If you change the output format (rename `value:`, put the blob on the same line, add another label between them), update the awk in the smoke test in the same commit or CI will fail.

## Release procedure

1. Land changes on `main`.
2. Add a new section to `CHANGELOG.md` for the version you are about to cut (move items out of `[Unreleased]` if any). Update the link references at the bottom of `CHANGELOG.md`.
3. Commit the changelog: `git commit -m "changelog for vX.Y.Z"`, push.
4. `just release vX.Y.Z` — creates the immutable `vX.Y.Z` tag and re-points the floating `vX` major pointer.
5. Push tags: `git push origin vX.Y.Z && git push --force origin vX`.

The `--force` is correct **only** for the moving major pointer (`v1`). Never force-push immutable `vX.Y.Z` tags — pinned consumers rely on those being stable.

## Editing the Justfile

`tag` is the low-level recipe (any `v`-prefixed name). `release` requires `vX.Y.Z` exactly and additionally moves `vX`. The format check in `release` uses a regex; if you loosen it (e.g. to allow pre-release suffixes like `-rc1`), make sure the major-pointer derivation `${version%%.*}` still produces a sensible value.
