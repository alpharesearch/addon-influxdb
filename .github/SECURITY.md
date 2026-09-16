# Security Policy

The security of this app is taken seriously. If you believe you have found a
security vulnerability in this repository, please report it privately as
described below.

**Please DO NOT report security vulnerabilities publicly!**

So... DO NOT create a GitHub issue for it ;)

## Reporting a vulnerability

Use [GitHub's private vulnerability reporting][report] for this repository, or
send a detailed description by email to `schulz@alpharesearch.de`. Please write
your report in English.

In the report, please include as much information as possible, including:

- An extensive description of the vulnerability.
- How it could be exploited.
- The potential impact you think it would have (e.g., DoS attackable, privacy
  concerns, leaking of credentials).
- Steps for reproducing the vulnerability.
- Code (if any), that is needed for reproducing the issue.
- If you have an idea for a fix, patch or any other adjustment for mitigating
  the vulnerability reported.

Please take care not to violate the privacy of other people in your report.
For example, stack traces or exploit scripts sent to me should never contain
private or personally identifiable information.

## After you have reported the vulnerability

Please give me at least a week to investigate and respond to the reported
vulnerability you have found; and up to 60 days to fix and distribute it. This
includes a window for existing users to upgrade, patch or mitigate the issue
as well.

If you intend, at any point, to disclose the vulnerability to someone else or
maybe even publicly, please give me a reasonable advance notice.

## Important limitation of this fork

This app packages InfluxDB 1.8.10, Chronograf and Kapacitor, which are
end-of-life upstream: InfluxData no longer ships fixes for the 1.x line. A
vulnerability in those components can therefore often only be mitigated, not
patched, and in some cases the honest answer is "move to a supported database".
Reports about those components are still very welcome: they determine whether
this fork stays viable, and mitigations (defaults, docs, warnings) may be
possible even when a patch is not.

## Bug bounty

Unfortunately, I cannot offer a paid bug bounty program. I will, however, give
my best efforts to show appreciation towards people that took the time and
effort to disclose vulnerabilities responsibly.

[report]: https://github.com/alpharesearch/addon-influxdb/security/advisories/new
