---
name: chainguard-compare-versions
description: Compare versions of a Chainguard image, show changelogs, and identify the latest digest. Use when the user asks what changed between image versions or wants to pin to a specific digest.
---

# Compare Chainguard Image Versions

**HARD RULES — read before doing anything else:**
1. **If the user provided a full image reference** (e.g., `cgr.dev/chainguard-private/node`), extract the org slug from it and skip all org resolution steps. Do not run `chainctl config view`. Do not call `cg-api` to list orgs. Do not ask the user which org to use. The org is already in the reference.
2. **Make one bounded MCP call at a time.** Write a one-sentence summary of the result before making the next call. Never issue two tool calls in the same step.
3. **Never call any unbounded enumeration tool — these freeze the UI.** Specifically banned (do not call these under any circumstances):
   - `cg-oci`: `list_repos`, `list_tags`
   - `cg-api`: `registry_repos_list`, `registry_tags_list`, `vulnerabilities_advisories_list`, `api_list`, `api_callers`

   Use `cg-versions`'s `get_project` to list streams (small, EOL-summary response). For `get_stream`, **always pass a specific stream name** (e.g., the major version like `3.12` for python) — never call it for a long-lived project's full history. After receiving the result, summarize only the **top 2 most recent versions** in the response to the user; do not echo the full version array. `cg-versions:search_projects` is allowed **only with an exact project name filter and only on the first page** — never paginate past `has_more: true`. For digest lookups on a specific tag, use `cg-oci`'s `get_manifest` with the fully-qualified reference. The only `iam_*_list` allowed is `iam_groups_list` for org resolution; do not call other IAM listing tools.

   **The same restrictions apply when shelling out via Bash.** Do not run `chainctl images repos list`, `chainctl images list`, `chainctl images tags list`, any `chainctl ... list` over orgs/repos/tags/advisories, or any `curl` against `cgr.dev/v2/_catalog` or similar catalog endpoints. The bans cover both MCP tools and their CLI equivalents — there is no escape hatch.
4. **If `cg-versions` returns no results, stop.** Report that the image has no version history in the versions catalog and ask the user to verify the image name. Do not attempt any fallback via `cg-oci`.

## Steps

1. **Org resolution** — only run this if the user did NOT provide a full `cgr.dev/<org>/` reference.
   a. Run `chainctl config view` and check `default.group` and `default.org-name`. If either is set, use it and proceed to step 2.
   b. If no default is set, call `cg-api` to list organizations.
      - Exactly one org: use it automatically, tell the user, and proceed.
      - Multiple orgs: **display them as a numbered list**, then ask: "Which organization would you like to use?" and stop. Do not proceed until the user replies.
      - No orgs: use `cgr.dev/chainguard` and note the user has no private organizations.

2. Call `cg-versions` for the **two most recent versions only** of the specified image. Output a one-sentence summary of what was returned before continuing.

3. Use any digests returned by `cg-versions` directly. Only call `cg-oci` if a digest is genuinely absent — and only with the exact fully-qualified reference (e.g., `cgr.dev/chainguard-private/node:latest`), never with a listing call.

4. Present the diff:
   - Tag names and dates
   - Package version bumps (e.g., `openssl 3.1.4 → 3.1.5`)
   - CVE fixes included in each version
   - Digests for pinning

## Example Output

```
Image: cgr.dev/mycompany/python

latest (2024-03-15)  sha256:abc123...
  - python 3.12.2 → 3.12.3
  - Fixed: CVE-2024-0450 (zipimport)

2024-03-01           sha256:def456...
  - openssl 3.2.0 → 3.2.1
  - Fixed: CVE-2024-0727
```

## Handling auth errors

- **401 Unauthorized** — the OAuth token has expired. Tell the user:
  > "Your Chainguard MCP session has expired. In Cursor, go to Settings → MCP, find the affected server, and click to re-authenticate. Then open a new agent session and try again."
- **403 Forbidden** — the user is authenticated but lacks access to the requested org or resource. Do NOT advise re-authentication; it will not help. Tell the user:
  > "You don't have access to `<org>` with your current Chainguard identity. Pick a different org, or request access from a Chainguard admin if this is wrong."

Do not attempt to fall back to `chainctl auth token` or direct registry calls — those use a different credential type and will also fail.

## Notes

- Always include digests — do not omit them.
- Recommend digest pinning in production and automated update tooling (e.g., Renovate, Dependabot) to track Chainguard image updates.
- If version history is not available for the user's org, say so explicitly rather than falling back to the public catalog.
