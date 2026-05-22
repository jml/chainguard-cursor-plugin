---
name: chainguard-inspect-image
description: Inspect a Chainguard image's packages, SBOM, entrypoint, and configuration. Use when the user wants to know what is inside a cgr.dev image or needs an SBOM summary.
---

# Inspect Chainguard Image

**HARD RULES — read before doing anything else:**
1. **Complete org resolution (Step 1) before any image lookup.** Do not use `cgr.dev/chainguard` as the org without first confirming the user has no private organizations.
2. **Make one bounded MCP call at a time.** Write a one-sentence summary of the result before making the next call. Never issue two tool calls in the same step.
3. **Never call any unbounded enumeration tool — these freeze the UI.** Specifically banned (do not call these under any circumstances):
   - `cg-oci`: `list_repos`, `list_tags`
   - `cg-api`: `registry_repos_list`, `registry_tags_list`, `vulnerabilities_advisories_list`, `api_list`, `api_callers`
   - `cg-versions`: `get_stream` — returns full upstream version history; can be megabytes for long-lived projects. Not needed for inspection.

   When you need this image's metadata or SBOM, use the `get_*` variant: `cg-oci`'s `get_manifest` / `get_config` / `get_sbom`, `cg-apk`'s `get_sbom` (for the image's SBOM, then iterate names client-side). Search tools (`cg-apk:search_packages`, `cg-versions:search_projects`) are allowed **only with an exact name filter and only on the first page** — never paginate past `has_more: true`. The only `iam_*_list` allowed is `iam_groups_list` for org resolution in Step 1; do not call other IAM listing tools.

   **The same restrictions apply when shelling out via Bash.** Do not run `chainctl images repos list`, `chainctl images list`, `chainctl images tags list`, any `chainctl ... list` over orgs/repos/tags/advisories, or any `curl` against `cgr.dev/v2/_catalog` or similar catalog endpoints. The bans cover both MCP tools and their CLI equivalents — there is no escape hatch.
4. **Always inspect a specific, fully-qualified image reference** (e.g., `cgr.dev/<org>/<image>:<tag>`). If the user did not provide one, ask for it. Do not try to discover images by listing the org's catalog. A 403, 404, or empty result means stop and ask the user — do not search other organizations or upstream catalogs as a fallback.

Use the `cg-api` MCP server to identify the user's organization, then `cg-oci` and `cg-apk` to inspect image contents.

## Steps

1. **Resolve the target organization — do this first, before any image lookup.**
   a. If the user has already specified a full image reference (e.g., `cgr.dev/mycompany/nginx:latest`), extract and use that org directly — skip the remaining checks.
   b. Run `chainctl config view` and check `default.group` and `default.org-name`. If either is set (non-empty), use that org and proceed to step 2.
   c. If no chainctl default is set, call the `cg-api` MCP server to list all organizations the user has access to.
      - If exactly one org is returned: use it automatically, tell the user which org you're using, and proceed to step 2.
      - If multiple orgs are returned: **display them as a numbered list**, then ask: "Which organization would you like to use?" and stop. Do not offer an opinion, do not suggest a "normal" or "typical" choice, do not add hints or parenthetical examples, do not proceed until the user replies.
      - If no orgs are returned: only then use `cgr.dev/chainguard` and note that the user appears to have no private organizations.
2. Query `cg-oci` with the fully-qualified reference (e.g., `cgr.dev/<org>/<image>:<tag>`) to fetch image metadata: architecture, OS, entrypoint, exposed ports, labels, and current digest. Do not use any tool that lists or searches across the org's catalog — see HARD RULES rule 3.
3. Query `cg-apk` for the APK packages in **this specific image only** (name, version, origin). Do not request a full APK index dump.
4. Present the SBOM summary in a readable table format.
5. Highlight:
   - Whether the image runs as non-root
   - Whether it includes a shell (or is fully distroless)
   - The total package count
   - The current digest

## Example Output

```
Image: cgr.dev/mycompany/nginx:latest
Digest: sha256:abc123...
Architecture: linux/amd64, linux/arm64
User: nonroot (65532)
Shell: none (distroless)

Packages (12 total):
  nginx          1.25.4
  openssl        3.2.1
  ca-certificates 20240101
  ...
```

## Handling auth errors

- **401 Unauthorized** — the OAuth token has expired. Tell the user:
  > "Your Chainguard MCP session has expired. In Cursor, go to Settings → MCP, find the affected server, and click to re-authenticate. Then open a new agent session and try again."
- **403 Forbidden** — the user is authenticated but lacks access to the requested org or resource. Do NOT advise re-authentication; it will not help. Tell the user:
  > "You don't have access to `<org>` with your current Chainguard identity. Pick a different org, or request access from a Chainguard admin if this is wrong."

Do not attempt to fall back to `chainctl auth token` or direct registry calls — those use a different credential type and will also fail.

## Notes

- If the user needs a shell for debugging, recommend the `-dev` variant temporarily — never in production.
- Fewer packages means a smaller attack surface. Note the package count relative to comparable upstream images where relevant.
