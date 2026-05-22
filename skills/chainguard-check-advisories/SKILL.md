---
name: chainguard-check-advisories
description: Check CVEs and security advisories for a Chainguard image or APK package. Use when the user asks about vulnerabilities, CVEs, or the security status of an image.
---

# Check Chainguard Advisories

**HARD RULES — read before doing anything else:**
1. **Complete org resolution (Step 1) before any advisory lookup.** Do not use `cgr.dev/chainguard` as the org without first confirming the user has no private organizations.
2. **Make one bounded MCP call at a time.** Write a one-sentence summary of the result before making the next call. Never issue two tool calls in the same step.
3. **Never call any unbounded enumeration tool — these freeze the UI.** Specifically banned (do not call these under any circumstances):
   - `cg-oci`: `list_repos`, `list_tags`
   - `cg-api`: `registry_repos_list`, `registry_tags_list`, `vulnerabilities_advisories_list`, `api_list`, `api_callers`
   - `cg-versions`: `get_stream` — returns full upstream version history; can be megabytes. Not needed for advisory queries.

   When you need advisories for a specific image or package, use `cg-api`'s `vulnerabilities_advisories_get` with the exact identifier. When you need a single image's metadata, use `cg-oci`'s `get_manifest` / `get_config`. Search tools (`cg-apk:search_packages`, `cg-versions:search_projects`) are allowed **only with an exact name filter and only on the first page** — never paginate past `has_more: true`. The only `iam_*_list` allowed is `iam_groups_list` for org resolution in Step 1; do not call other IAM listing tools.

   **The same restrictions apply when shelling out via Bash.** Do not run `chainctl images repos list`, `chainctl images list`, `chainctl images tags list`, any `chainctl ... list` over orgs/repos/tags/advisories, or any `curl` against `cgr.dev/v2/_catalog` or similar catalog endpoints. The bans cover both MCP tools and their CLI equivalents — there is no escape hatch.
4. **Always scope advisory queries to a specific image or package the user named.** If the user did not name one, ask them which image or APK package to check. Do not enumerate the org's advisories as a fallback. A 403, 404, or empty result means stop and ask the user — do not search other organizations or upstream catalogs.

Use the `cg-api` MCP server to identify the user's organization and query security advisories.

## Steps

1. **Resolve the target organization — do this first, before any advisory lookup.**
   a. If the user has already specified a full image reference (e.g., `cgr.dev/mycompany/nginx:latest`), extract and use that org directly — skip the remaining checks.
   b. Run `chainctl config view` and check `default.group` and `default.org-name`. If either is set (non-empty), use that org and proceed to step 2.
   c. If no chainctl default is set, call the `cg-api` MCP server to list all organizations the user has access to.
      - If exactly one org is returned: use it automatically, tell the user which org you're using, and proceed to step 2.
      - If multiple orgs are returned: **display them as a numbered list**, then ask: "Which organization would you like to use?" and stop. Do not offer an opinion, do not suggest a "normal" or "typical" choice, do not add hints or parenthetical examples, do not proceed until the user replies.
      - If no orgs are returned: only then use `cgr.dev/chainguard` and note that the user appears to have no private organizations.
2. Query `cg-api` for advisories scoped to the **specific image or package the user named**, within that organization. Do not request advisories across the entire org — see HARD RULES rule 3.
3. Report:
   - CVE IDs and severity (Critical / High / Medium / Low)
   - Affected versions
   - Fixed versions (if available)
   - Chainguard's advisory status (e.g., `fixed`, `not_affected`, `under_investigation`)
4. If the user is on an older image tag, use `cg-versions` to compare against the latest and advise an upgrade if it resolves open vulnerabilities.

## Handling auth errors

- **401 Unauthorized** — the OAuth token has expired. Tell the user:
  > "Your Chainguard MCP session has expired. In Cursor, go to Settings → MCP, find the affected server, and click to re-authenticate. Then open a new agent session and try again."
- **403 Forbidden** — the user is authenticated but lacks access to the requested org or resource. Do NOT advise re-authentication; it will not help. Tell the user:
  > "You don't have access to `<org>` with your current Chainguard identity. Pick a different org, or request access from a Chainguard admin if this is wrong."

Do not attempt to fall back to `chainctl auth token` or direct registry calls — those use a different credential type and will also fail.

## Notes

- Advisory data is scoped to the authenticated organization. Results may differ from the public advisory feed at images.chainguard.dev/advisories.
- Distinguish between upstream CVEs (affecting the package) and Chainguard's determination of actual exploitability in their build.
- If querying a non-Chainguard image, tell the user that advisory data is only available for Chainguard-managed images.
- Do not fabricate CVE IDs — if the MCP server returns no results, report that explicitly.
