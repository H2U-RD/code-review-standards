# Code Review Standards

A shared repository of code review guidelines used across engineering teams. Standards are organized by a universal base ruleset plus language/platform-specific supplements.

## Severity Levels

| Level | Description | Merge Condition |
|-------|-------------|-----------------|
| **Critical** | Bug, security vulnerability, data loss, or system crash | Must be fixed before merge |
| **Major** | Significant impact on maintainability, performance, observability, or engineering discipline | Should be fixed, or documented rationale required for reviewer to approve |
| **Minor** | Suggested improvement, does not block functionality | May be addressed in a follow-up PR with a tracked issue |
| **Info** | Style suggestions, learning resources, observations | No response required |

## Standards

| Document | Scope |
|----------|-------|
| [code_review_standard.md](code_review_standard.md) | Universal — applies to all languages and platforms |
| [code_review_standard_dotnet.md](code_review_standard_dotnet.md) | .NET / C# specific |
| [code_review_standard_java.md](code_review_standard_java.md) | Java specific |
| [code_review_standard_frontend.md](code_review_standard_frontend.md) | TypeScript / Vue / Nuxt |
| [code_review_standard_python.md](code_review_standard_python.md) | Python / Django / Wagtail |
| [code_review_standard_mobile.md](code_review_standard_mobile.md) | Android / iOS / React Native |

## How to Use

1. Start with the **universal standard** ([code_review_standard.md](code_review_standard.md)) for all reviews.
2. Apply the relevant **language/platform supplement** on top based on the PR's tech stack.
3. When leaving review comments, prefix each with its severity level (e.g. `[Critical]`, `[Major]`).

## Contributing

To propose a new rule or modify an existing one, open a Merge Request with:
- The rule added to the appropriate severity section
- A brief rationale explaining the failure mode it prevents
