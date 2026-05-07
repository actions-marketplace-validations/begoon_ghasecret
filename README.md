# ghasecret

Occasionally you need to recover the plaintext of a value stored in GitHub Actions secrets — the original was never written down, the engineer who set it has moved on, or you are migrating to a different secrets store. The value can in principle be printed from inside a workflow, but the Actions runner automatically scans every log line for any registered secret (and, since 2024, for its single-base64 form too) and replaces matches with `***`, so a plain `echo "$MY_SECRET"` shows up redacted.

This action works around the masker by encoding the secret with `base64` three times before printing it. The triple-encoded blob shares no common substring with the original or its single-base64 form, so the log filter does not match it and the value appears in the run log verbatim. Pasting the printed `decode:` one-liner into a local shell runs `base64 -d` three times and recovers the original value.

## Usage

### A single secret

```yaml
- uses: begoon/ghasecret@v1
  with:
    secret: ${{ secrets.MY_SECRET }}
```

### Multiple secrets

Each line is `NAME=VALUE`. The `NAME` is used as the label in the printed output:

```yaml
- uses: begoon/ghasecret@v1
  with:
    secrets: |
      MY_SECRET=${{ secrets.MY_SECRET }}
      OTHER=${{ secrets.OTHER }}
      API_TOKEN=${{ secrets.API_TOKEN }}
```

## Output

For each secret the action prints a collapsible group like:

```text
::group::MY_SECRET
value:  WkVkb2NHTjVNWEJqZVRGb1RGY3hNVmt5...
decode: echo 'WkVkb2NHTjVNWEJqZVRGb1RGY3hNVmt5...' | base64 -d | base64 -d | base64 -d
::endgroup::
```

Copy the `decode:` line into your terminal (Linux or macOS — both BSD and GNU `base64` accept `-d`) and the original secret is printed to stdout.

## Security

Anyone with read access to the workflow logs can decode these values. Use this only:

- on a private repo, or a repo you fully control
- in a workflow you trust (do not run on untrusted PRs)
- as a one-off — rotate the secret afterwards

## Releasing

Cut a new version:

```bash
just tag v1.0.0
git push origin v1.0.0
```

Publish a moving `v1` pointer so users can pin `uses: begoon/ghasecret@v1`:

```bash
just tag v1
git push origin v1
```

For subsequent releases, use the `release` recipe — it creates the version tag and re-points the matching major pointer in one go:

```bash
just release v1.0.1
git push origin v1.0.1
git push --force origin v1
```

Force-pushing is intentional for the moving `v1` pointer. Do not force-push immutable tags like `v1.0.0`.

## License

MIT
