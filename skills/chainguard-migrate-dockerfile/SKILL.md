---
name: chainguard-migrate-dockerfile
description: Migrate an existing Dockerfile to use a Chainguard base image. Use when the user wants to harden a Dockerfile, replace a Docker Hub base image, or apply a multi-stage build pattern with cgr.dev images.
---

# Migrate Dockerfile to Chainguard

**HARD RULES — read before doing anything else:**
1. **Complete org resolution (Step 3) before any image lookup.** Do not use `cgr.dev/chainguard` as the org without first confirming the user has no private organizations.
2. **Make one bounded MCP call at a time.** Write a one-sentence summary of the result before making the next call. Never issue two tool calls in the same step.
3. **Never call any unbounded enumeration tool — these freeze the UI.** Specifically banned (do not call these under any circumstances):
   - `cg-oci`: `list_repos`, `list_tags`
   - `cg-api`: `registry_repos_list`, `registry_tags_list`, `vulnerabilities_advisories_list`, `api_list`, `api_callers`
   - `cg-versions`: `get_stream` — returns full upstream version history; can be megabytes. Not needed for Dockerfile migration.

   When you need to check a single image's existence or fetch its digest, use `cg-oci`'s `get_manifest` / `get_config` with the fully-qualified reference. Search tools (`cg-apk:search_packages`, `cg-versions:search_projects`) are allowed **only with an exact name filter and only on the first page** — never paginate past `has_more: true`. The only `iam_*_list` allowed is `iam_groups_list` for org resolution in Step 3; do not call other IAM listing tools.

   **The same restrictions apply when shelling out via Bash.** Do not run `chainctl images repos list`, `chainctl images list`, `chainctl images tags list`, any `chainctl ... list` over orgs/repos/tags/advisories, or any `curl` against `cgr.dev/v2/_catalog` or similar catalog endpoints. The bans cover both MCP tools and their CLI equivalents — there is no escape hatch.
4. **Map the upstream base image to a specific Chainguard image name and look that name up directly.** Common mappings: `python:*` → `python`, `node:*` → `node`, `golang:*` → `go`, `openjdk:*`/`eclipse-temurin:*` → `jdk`, `nginx:*` → `nginx`. If you can't confidently map the upstream image, ask the user — do not enumerate the org's catalog to search. A 403, 404, or empty result on the mapped image means stop and ask whether to try the public `cgr.dev/chainguard` catalog — do not search other organizations.

Read the user's Dockerfile, identify the base image, find the Chainguard equivalent scoped to the user's organization, and produce a hardened replacement.

## Steps

1. Read the Dockerfile (ask the user to share it if not already visible).
2. Identify the `FROM` instruction(s) and note the current base image (e.g., `python:3.12-slim`, `node:20-alpine`).
3. **Resolve the target organization — do this before any image lookup.**
   a. Run `chainctl config view` and check `default.group` and `default.org-name`. If either is set (non-empty), use that org and proceed to step 4.
   b. If no chainctl default is set, call the `cg-api` MCP server to list all organizations the user has access to.
      - If exactly one org is returned: use it automatically, tell the user which org you're using, and proceed to step 4.
      - If multiple orgs are returned: **display them as a numbered list**, then ask: "Which organization would you like to use?" and stop. Do not offer an opinion, do not suggest a "normal" or "typical" choice, do not add hints or parenthetical examples, do not proceed until the user replies.
      - If no orgs are returned: only then use `cgr.dev/chainguard` and note that the user appears to have no private organizations.
   Use the resolved org slug in all image references: `cgr.dev/<org-slug>/<image>`. Do not use `cgr.dev/chainguard` without first completing this step.
4. Query `cg-oci` with the **targeted, fully-qualified** reference `cgr.dev/<org>/<mapped-image>` to confirm it exists in the user's organization. Do not use any tool that lists or searches across the catalog — see HARD RULES rule 3. If the targeted lookup fails, ask the user whether they want to try the public `cgr.dev/chainguard` catalog instead.
5. Fetch the current digest for the runtime image via `cg-oci` (targeted, by reference) and pin to it in the final `FROM`.
6. Rewrite the Dockerfile applying these best practices:
   - Replace `FROM` with `cgr.dev/<org>/<image>:latest@sha256:<digest>`
   - Use a multi-stage build: `-dev` variant for the build stage, minimal variant for runtime
   - Remove `apt-get install` / `apk add` from the runtime stage
   - Ensure the final stage runs as `nonroot` user (UID 65532)
   - Use `COPY --chown=nonroot:nonroot` for file ownership
7. Show a diff of the changes and explain each one.

## Multi-Stage Pattern

```dockerfile
# Build stage — use -dev variant which includes shell, package manager, compilers
FROM cgr.dev/<org>/python:latest-dev AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt --target /app/deps

# Runtime stage — minimal, distroless, no shell, digest-pinned
FROM cgr.dev/<org>/python:latest@sha256:<digest>
WORKDIR /app
COPY --from=builder /app/deps /app/deps
COPY --chown=nonroot:nonroot src/ .
ENV PYTHONPATH=/app/deps
ENTRYPOINT ["python", "main.py"]
```

## Handling auth errors

- **401 Unauthorized** — the OAuth token has expired. Tell the user:
  > "Your Chainguard MCP session has expired. In Cursor, go to Settings → MCP, find the affected server, and click to re-authenticate. Then open a new agent session and try again."
- **403 Forbidden** — the user is authenticated but lacks access to the requested org or resource. Do NOT advise re-authentication; it will not help. Tell the user:
  > "You don't have access to `<org>` with your current Chainguard identity. Pick a different org, or request access from a Chainguard admin if this is wrong."

Do not attempt to fall back to `chainctl auth token` or direct registry calls — those use a different credential type and will also fail.

## Notes

- Always pin the runtime stage to a digest for reproducibility.
- If the image is not available in the user's organization catalog, say so — do not silently substitute the public catalog image.
- If the Dockerfile installs OS packages in the runtime stage, suggest moving them to the build stage or contacting Chainguard to add the package to the base image.
- For distroless images, `CMD ["/bin/sh"]` will fail — guide the user to exec-form `ENTRYPOINT`.
