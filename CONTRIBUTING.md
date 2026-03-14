# Contributing

This repository is primarily a hands-on Kubernetes study guide, but contributions that improve clarity, repeatability, and safety are welcome.

## What to optimize for

Prefer changes that make the repo easier to learn from and easier to reuse:

- keep teaching examples small and focused
- preserve pinned versions unless you have verified an upstream update end to end
- update related docs, manifests, and values files together when they refer to the same component
- favor clear explanations over clever shortcuts

## Recommended workflow

1. Start with a small, scoped change.
2. Update the relevant documentation alongside the manifests.
3. Run local checks for the files you touched.
4. Explain what changed and what was verified in the pull request.

## Documentation changes

When updating install guidance or version pins:

- verify version changes against primary upstream sources
- use exact dates when the docs say a version was current "as of" a given date
- prefer stable GA releases over prereleases unless prerelease coverage is intentional
- keep chart version and app version distinct when upstream treats them separately

## Manifests and examples

Most folders in this repo are teaching examples, not production-ready distributions. Please keep that distinction explicit:

- placeholders should stay obviously fake
- secrets should never contain live credentials
- examples should prefer safe defaults like pinned images, namespaces, and preview commands

## Embedded upstream references

Some top-level directories point to embedded upstream projects or reference copies. Avoid incidental edits there unless the change is intentional and documented. If you are improving this repo's guidance, prefer updating the first-party docs around those projects rather than rewriting the mirrored source.

## Pull request notes

A good pull request description for this repo usually includes:

- the user-facing problem being solved
- the files or sections affected
- how the change was verified
- any follow-up work that is intentionally left out
