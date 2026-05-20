# Security policy

BigLake is a GCP-hosted data platform for Australian government datasets. Security findings — whether internal or external — are taken seriously.

## Reporting a vulnerability

**Do not report security vulnerabilities in public issues, pull requests, or discussions.**

Please report via one of:

- GitHub private security advisory on the affected repo: `Security` tab → **Report a vulnerability**.
- Email a maintainer directly (see commit history for current maintainers).

When reporting, include:

- The repo and version / commit affected.
- A description of the issue and its impact.
- Steps to reproduce (proof-of-concept welcome but not required).
- Any suggested mitigation.

You will receive an acknowledgement within 5 business days. We aim to triage and respond with a disclosure timeline within 14 days.

## Disclosure policy

We follow a **coordinated disclosure** model:

1. Reporter contacts us privately.
2. We confirm the issue and develop a fix.
3. A fix is deployed to test, then prod.
4. After deployment, the reporter and maintainers agree on a public disclosure date (typically 30–90 days from the initial report).

## Scope

In scope:

- All repos under the [`big-lake`](https://github.com/big-lake) GitHub org.
- The deployed `*.biglake.au` services.

Out of scope:

- Third-party services (OpenMetadata upstream, GCP itself, etc.) — please report to the upstream vendor.
- Social engineering of maintainers.
- Findings requiring physical access or compromised maintainer credentials.

## How we track findings internally

Each repo maintains a **risk register** at `documentation/security/risks.md` and dated review reports at `documentation/security/reviews/`. See [.github/copilot-instructions.md](.github/copilot-instructions.md) for the convention.

External reports become rows in the risk register once triaged, with severity and an owner.
