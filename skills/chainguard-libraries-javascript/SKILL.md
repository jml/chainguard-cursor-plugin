---
name: chainguard-libraries-javascript
description: Configure an npm/Node.js project to use Chainguard Libraries for hardened JavaScript packages. Use when the user wants to set up Chainguard Libraries for npm, Yarn, or pnpm, or asks about hardened Node.js packages.
---

# Chainguard Libraries for JavaScript

Chainguard Libraries provides hardened npm packages with reduced CVE exposure. Packages are distributed via a private npm-compatible registry that requires an auth token.

## Step 1: Configure npm (recommended)

The fastest path is `chainctl auth configure-npm`, which writes a project-level `.npmrc` for you using your current Chainguard session:

```bash
chainctl auth configure-npm
```

For CI/non-interactive environments, request a long-lived pull token instead:

```bash
# Writes .npmrc with basic-auth credentials backed by a pull token
chainctl auth configure-npm --pull-token --ttl=24h

# Optional: scope the pull token to a specific organization
chainctl auth configure-npm --pull-token --parent=my-org --ttl=24h
```

`--pull-token` accepts the same options as `chainctl auth pull-token create`:
- `--ttl=24h` — token lifetime (max `8760h` / 1 year).
- `--parent=my-org` — issue the pull token under a specific organization.
- `--name=my-ci-token` — label the pull token for easier identification later.

## Step 1b (alternative): Mint a raw pull token

If you need the bare token (e.g., to inject into a `.npmrc` template or another tool), create one directly:

```bash
chainctl auth pull-token create --repository=javascript
```

Export it for use in templated config:

```bash
export CHAINGUARD_LIBRARIES_JS_TOKEN=$(chainctl auth pull-token create --repository=javascript --ttl=24h)
```

## Step 2: Manual npm config (if not using `configure-npm`)

Set the Chainguard registry and auth token:

```bash
# Configure the registry endpoint
npm config set registry https://libraries.cgr.dev/js/

# Set the auth token
npm config set //libraries.cgr.dev/js/:_authToken ${CHAINGUARD_LIBRARIES_JS_TOKEN}
```

Or add a `.npmrc` file to your project root (do NOT commit the token — use an env var):

```ini
registry=https://libraries.cgr.dev/js/
//libraries.cgr.dev/js/:_authToken=${CHAINGUARD_LIBRARIES_JS_TOKEN}
```

## Step 3: Configure Yarn (v1 / Classic)

```bash
yarn config set registry https://libraries.cgr.dev/js/
```

Add to `.yarnrc`:

```
registry "https://libraries.cgr.dev/js/"
//libraries.cgr.dev/js/:_authToken ${CHAINGUARD_LIBRARIES_JS_TOKEN}
```

## Step 4: Configure Yarn (v2+ / Berry) or pnpm

Add to `.yarnrc.yml`:

```yaml
npmRegistryServer: "https://libraries.cgr.dev/js/"
npmAuthToken: "${CHAINGUARD_LIBRARIES_JS_TOKEN}"
```

Add to `.npmrc` for pnpm:

```ini
registry=https://libraries.cgr.dev/js/
//libraries.cgr.dev/js/:_authToken=${CHAINGUARD_LIBRARIES_JS_TOKEN}
```

## Step 5: Verify

```bash
npm install
# or
yarn install
# or
pnpm install
```

## CI/CD Configuration

In GitHub Actions:

```yaml
- name: Set Chainguard Libraries token
  run: echo "CHAINGUARD_LIBRARIES_JS_TOKEN=$(chainctl auth pull-token create --repository=javascript --ttl=2h)" >> $GITHUB_ENV
```

Or use a pre-configured secret:

```yaml
env:
  CHAINGUARD_LIBRARIES_JS_TOKEN: ${{ secrets.CHAINGUARD_LIBRARIES_JS_TOKEN }}
```

## Notes

- Chainguard Libraries mirrors npm packages with patched dependencies. Package names and APIs are identical — no code changes required.
- Add `.npmrc` to `.gitignore` if it contains literal tokens (prefer env var interpolation instead).
- Packages not yet in Chainguard Libraries will not resolve from this registry — configure a fallback or check with Chainguard for coverage.
- Token refresh: re-run `chainctl auth configure-npm` (or `chainctl auth pull-token create --repository=javascript`) when the current token expires. For long-lived CI use, pass `--ttl=24h` (or up to `8760h`).
