---
name: chainguard-find-image
description: Find the right Chainguard container image for a given language, framework, or tool. Use when the user asks which cgr.dev image to use, or wants to replace a Docker Hub base image.
---

# Find Chainguard Image

**HARD RULES — read before doing anything else:**
1. **Complete org resolution (Step 1) before any image lookup.** Do not use `cgr.dev/chainguard` as the org without first confirming the user has no private organizations.
2. **Make one bounded MCP call at a time.** Write a one-sentence summary of the result before making the next call. Never issue two tool calls in the same step.
3. **Never call any unbounded enumeration tool — these freeze the UI.** Specifically banned (do not call these under any circumstances):
   - `cg-oci`: `list_repos`, `list_tags`
   - `cg-api`: `registry_repos_list`, `registry_tags_list`, `vulnerabilities_advisories_list`, `api_list`, `api_callers`
   - `cg-versions`: `get_stream` — returns full upstream version history; can be megabytes for long-lived projects like python/openssl/kubernetes. Use `get_project` instead (small streams summary). Leave `get_stream` to the compare-versions skill.

   When you need a single image, package, version, or repo, use the `get_*` variant: `cg-oci`'s `get_manifest` / `get_config`, `cg-api`'s `registry_repos_get` / `registry_tags_get`, `cg-versions`'s `get_project`, `cg-apk`'s `get_sbom`. Search tools (`cg-apk:search_packages`, `cg-versions:search_projects`) are allowed **only with an exact name filter and only on the first page** — never paginate past `has_more: true`. The only `iam_*_list` allowed is `iam_groups_list` for org resolution in Step 1; do not call other IAM listing tools.

   **The same restrictions apply when shelling out via Bash.** Do not run `chainctl images repos list`, `chainctl images list`, `chainctl images tags list`, any `chainctl ... list` over orgs/repos/tags/advisories, or any `curl` against `cgr.dev/v2/_catalog` or similar catalog endpoints. The bans cover both MCP tools and their CLI equivalents — there is no escape hatch.
4. **A 403, 404, or empty result on a targeted lookup means stop and ask. Do not search other organizations or upstream catalogs.** Specifically forbidden after a miss: a second `iam_groups_list` call to discover alternative orgs, any `cg-versions` call to "confirm" the upstream project exists, any `chainctl` shell-out to enumerate. Phrase the user prompt as: "No `<image>` image was found in `<org>`. Would you like me to check the public `cgr.dev/chainguard` catalog instead, or do you know a different name to try?" Wait for the user's reply.

Use the `cg-api` MCP server to identify the user's Chainguard organization, then use `cg-oci` (only) to look up and pin the target image. Do not use `cg-versions` for find-image — it is not needed and its `get_stream` tool can hang the UI on long-lived upstream projects.

## Steps

1. **Resolve the target organization — do this first, before any image lookup.**
   a. Run `chainctl config view` and check `default.group` and `default.org-name`. If either is set (non-empty), use that org and proceed to step 2.
   b. If no chainctl default is set, call the `cg-api` MCP server to list all organizations the user has access to.
      - If exactly one org is returned: use it automatically, tell the user which org you're using, and proceed to step 2.
      - If multiple orgs are returned: **display them as a numbered list**, then ask: "Which organization would you like to use?" and stop. Do not offer an opinion, do not suggest a "normal" or "typical" choice, do not add hints or parenthetical examples, do not proceed until the user replies.
      - If no orgs are returned: only then use `cgr.dev/chainguard` and note that the user appears to have no private organizations.

2. Query `cg-oci` with a **targeted** lookup for the specific image name within the resolved organization (e.g., `image_get` style with the fully-qualified `cgr.dev/<org>/<image>` reference). Do not use any listing or search tool that walks the org's catalog — see HARD RULES rule 3. If the user gave a fuzzy term like "python 3 image for production", map it to the canonical image name (`python`) and look that up directly; if no canonical name comes to mind, ask the user.
3. Check whether a `-dev` variant exists by calling `cg-oci`'s `get_manifest` with the `latest-dev` tag on the same image. A 200 means it exists; a 404 means it doesn't. Do not use `cg-versions` for this — it is not needed for image lookups.
4. Fetch the current digest for the recommended image via `cg-oci` and include it in the output. **Always provide the digest — do not skip this step.**
5. Present the result and explain the difference between the standard image (minimal, distroless) and the `-dev` variant.

## Example Output

```
Organization: mycompany

cgr.dev/mycompany/python:latest@sha256:abc123...   # minimal, production-ready
cgr.dev/mycompany/python:latest-dev                # includes shell + pip for development
```

## Handling auth errors

- **401 Unauthorized** — the OAuth token has expired. Tell the user:
  > "Your Chainguard MCP session has expired. In Cursor, go to Settings → MCP, find the affected server, and click to re-authenticate. Then open a new agent session and try again."
- **403 Forbidden** — the user is authenticated but lacks access to the requested org or resource. Do NOT advise re-authentication; it will not help. Tell the user:
  > "You don't have access to `<org>` with your current Chainguard identity. Pick a different org, or request access from a Chainguard admin if this is wrong."

Do not attempt to fall back to `chainctl auth token` or direct registry calls — those use a different credential type and will also fail.

## Notes

- Always include the digest (`@sha256:...`) in the recommended production reference.
- If the image is not found in the user's org catalog, say so explicitly — do not silently fall back to the public catalog.
- Chainguard images are rebuilt nightly; `:latest` always resolves to the latest patched build.
