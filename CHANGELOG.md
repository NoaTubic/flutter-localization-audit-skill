# Changelog

## [1.0.0] — 2025-05-06

### Added
- 8 audit categories: Hardcoded Strings, ARB Completeness, l10n Setup, Pluralization, RTL Support, Date/Number Formatting, String Interpolation, Locale Fallback
- Parameter support — pass specific category IDs (e.g. `rtl`, `hardcoded,arb`) to scope the audit
- `references/localization_checks.md` with grep patterns, ICU plural requirements by language, RTL-safe replacements, and known bad patterns
- Per-locale translation coverage table in the report
- Severity rating system: Critical / High / Medium / Low / Info
- Structured markdown report template with executive summary, per-finding evidence, positive findings, and next steps
