# AGENTS.md

## Repo-specific guidance

- Treat version pins in this repo as intentional documentation, not filler text. If you touch a pinned version, update it end to end or leave it alone.
- Prefer small, targeted updates over broad version churn. If the user asks for one component, do not opportunistically rewrite every pinned version in the README.

## Updating pinned versions

- Verify every version change against a primary source first: official project docs, official release notes, Helm chart repo output, or the project's release page. Do not rely on third-party blog posts.
- When the README says "latest ... as of <date>", update the version and the date together. If release publication dates are mentioned, update those too.
- Prefer stable GA releases. Do not switch the guide to alpha, beta, or release-candidate builds unless the user explicitly asks for prerelease coverage.
- Update all linked pins in the same change: README text, Helm commands, image tags, sample values files, manifest URLs, and any "current as of" notes that refer to the same component.
- Keep chart version and app version separate when upstream treats them separately. Document both when that distinction matters.
- Avoid floating install sources in teaching material when a component is described as pinned. If a quick-start command uses `latest` for convenience, add or preserve a pinned alternative.
- Use exact dates in documentation updates.
- If you cannot confidently verify a newer version, do not guess. Leave the current pin in place and note the verification gap in your response.

## Documentation style for version bumps

- State what was verified in plain language, for example: "latest upstream release as of March 14, 2026".
- Keep version guidance actionable. If the repo uses placeholders like `<tested_k3s_version>`, preserve them unless you have verified and intentionally chosen a concrete replacement.
- Do not introduce contradictions between prose and manifests. If a README command, a sample manifest, and a values file all pin the same thing, keep them aligned.
