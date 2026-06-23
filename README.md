# Decode GitHub App Private Key

A composite [GitHub Action](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action) that decodes a GitHub App private key (PEM) from a base64-encoded or raw value and outputs it for use in signing requests — for example, to mint an installation access token with [`actions/create-github-app-token`](https://github.com/actions/create-github-app-token).

The decoded key is explicitly masked in workflow logs to prevent accidental exposure.

## Why

GitHub automatically masks the raw secret values you reference in a workflow. If you store a private key **base64-encoded** (a common way to keep multi-line PEMs intact in a single secret), the value the action produces *after decoding* is a new string the runner has never seen — so GitHub does not mask it automatically. This action decodes the key and re-applies masking line by line, so the PEM never lands unmasked in logs.

## Usage

The action is format-agnostic: pass either a raw PEM or a base64-encoded PEM. It detects the format by looking for the `-----BEGIN` header and skips the decode step when the value is already raw.

```yaml
steps:
  - name: Decode private key
    id: decode
    uses: product-os/decode-private-key@v1
    with:
      secret: ${{ secrets.GH_APP_PRIVATE_KEY }}

  - name: Create installation token
    id: app-token
    uses: actions/create-github-app-token@v3
    with:
      app-id: ${{ vars.GH_APP_ID }}
      private-key: ${{ steps.decode.outputs.private-key }}

  - name: Use the token
    run: gh repo list
    env:
      GH_TOKEN: ${{ steps.app-token.outputs.token }}
```

## Inputs

| Name     | Required | Description                                                                                   |
| -------- | -------- | --------------------------------------------------------------------------------------------- |
| `secret` | yes      | GitHub App private key (PEM) to decode and mask. Accepts a raw PEM or a base64-encoded value. |

## Outputs

| Name          | Description                                                           |
| ------------- | --------------------------------------------------------------------- |
| `private-key` | The decoded GitHub App private key (PEM) to use for signing requests. |

## How it works

1. Fails fast if the `secret` input is empty.
2. If the value contains `-----BEGIN`, it is treated as a raw PEM and used as-is (GitHub already masks it).
3. Otherwise the value is base64-decoded, and each line of the result is masked with `::add-mask::`.
4. The decoded value is validated to confirm it looks like a PEM (`-----BEGIN` header present).
5. The key is written to `$GITHUB_OUTPUT` using a random heredoc delimiter to prevent output injection via crafted secret values.

## License

Copyright 2026 Balena Ltd. Licensed under the [Apache License 2.0](./LICENSE).
