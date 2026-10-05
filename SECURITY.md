# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability in **Open Component Model** projects, please report it responsibly
through one of the channels below. **Do not open a public issue for security vulnerabilities.**

### What to Include in Your Report

To help us assess and address the vulnerability efficiently, please include:

- **Affected component(s)** and version(s)
- **Steps to reproduce** the vulnerability
- **Impact assessment** — what an attacker could achieve
- Whether the vulnerability is **already publicly known**
- Any suggested fix or mitigation (optional)

### GitHub Private Vulnerability Reporting (Preferred)

Please use GitHub's built-in private vulnerability reporting:

1. Navigate to the **Security** tab of the repository.
2. Click **Report a vulnerability**.
3. Fill in the details and submit.

*For more information, see [Privately reporting a security vulnerability](https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing-information-about-vulnerabilities/privately-reporting-a-security-vulnerability).*

### Email (Fallback)

You may report vulnerabilities via email to the Technical Steering Committee at
[open-component-model-tsc@lists.neonephos.org](mailto:open-component-model-tsc@lists.neonephos.org).

## Security Contacts

The [Technical Steering Committee (TSC)](https://github.com/open-component-model/open-component-model/blob/main/docs/steering/OWNERS.md)
collectively serves as the security contact for Open Component Model projects.

## Supported Versions

Security updates are provided for the latest three minor releases (y, y-1, y-2).

| Version | Supported |
|---------|-----------|
| Latest 3 minors (y, y-1, y-2) | Yes |
| Older minors | No |

## Response Process

This project follows the [NeoNephos Security Guidelines](https://github.com/neonephos/guidelines-development/blob/main/security-guidelines/security-guidelines.md) for vulnerability handling. In summary:

- **Initial response**: We will respond to your report within **14 calendar days** of receipt, in line with the [OpenSSF Best Practices](https://www.bestpractices.dev/) requirement.
- **Embargo**: Vulnerability details will remain confidential for up to **90 days** from report receipt while a fix is developed, consistent with the [Google Project Zero disclosure policy](https://googleprojectzero.blogspot.com/2021/04/policy-and-disclosure-2021-edition.html).
- **Disclosure**: Once a fix is available (or the embargo expires), we will publish a security advisory with full details.

### Severity Response Targets

| Severity | CVSS Score | Fix Target | Disclosure Target |
|----------|------------|------------|-------------------|
| Critical | 9.0 – 10.0 | ≤ 14 days | ≤ 30 days         |
| High | 7.0 – 8.9 | ≤ 30 days     | ≤ 60 days         |
| Medium | 4.0 – 6.9 | ≤ 90 days   | ≤ 90 days         |
| Low | 0.1 – 3.9 | Best effort    | Best effort       |

*These are **SHOULD**-level targets as defined by the [NeoNephos Security Guidelines](https://github.com/neonephos/guidelines-development/blob/main/security-guidelines/security-guidelines.md#7-severity-classification-and-response-targets). The 90-day embargo ceiling is a **MUST** aligned with Google Project Zero. All timelines are measured from report receipt (Day 0); fix and disclosure may occur simultaneously.*

## Disclosure Policy

We follow [coordinated disclosure](https://en.wikipedia.org/wiki/Coordinated_vulnerability_disclosure). We ask that you:

- Allow us reasonable time to investigate and address the vulnerability before public disclosure.
- Do not exploit the vulnerability beyond what is necessary to demonstrate the issue.
- Do not access or modify data belonging to other users.

We are committed to crediting reporters in our security advisories unless you prefer to remain anonymous.

## CRA Stewardship

CRA stewardship: This project is supported under the Linux Foundation CRA stewardship framework, as described at https://www.linuxfoundation.org/security. Security vulnerabilities should be reported through the mechanisms described above, which we will coordinate with our CRA steward. For actively exploited vulnerabilities and severe incidents that may require CRA escalation, please use the project's emergency security reporting mechanisms as appropriate.

**CRA Steward Contact**: `steward@linuxfoundation.org`

**Commercial Intent**: Open Component Model software is designed and intended for use in commercial contexts.

For more information, see the [NeoNephos Security Guidelines §11](https://github.com/neonephos/guidelines-development/blob/main/security-guidelines/security-guidelines.md#11-eu-cyber-resilience-act-cra-compliance).

## Past Security Advisories

Published advisories are listed per repository in the repository's
**Security → Advisories** tab. Direct link (replace `<repository>` with the repository in question,
e.g. `open-component-model`):

```text
https://github.com/open-component-model/<repository>/security/advisories?state=published
```

## Additional Security Documentation

For deeper understanding of OCM's security architecture and design:

- [Secure Design](https://github.com/open-component-model/open-component-model/blob/main/docs/security/secure-design.md) — OCM's security design principles and mechanisms, traceable to architecture decision records and source code.
- [Security Assurance Case](https://github.com/open-component-model/open-component-model/blob/main/docs/security/assurance-case.md) — Structured assurance case mapping identified threats to implemented mitigations with evidence links.

**Note**: These are contributor and auditor documents, not end-user documentation.
