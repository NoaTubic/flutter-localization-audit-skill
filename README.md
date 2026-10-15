# Flutter Localization Audit Skill

> Read the article: [Four Claude Code skills to audit a Flutter app](https://medium.com/@noatubic/four-claude-code-skills-to-audit-a-flutter-app-f72823761062)

A Claude Code skill that performs a comprehensive localization audit on a Flutter/Dart project and produces a structured, actionable markdown report.

Covers **8 audit categories** — from hardcoded strings and missing ARB translations to RTL support, plural forms, and locale-unaware date formatting.

---

## What it audits

| ID | Category |
|----|----------|
| L1 | Hardcoded Strings |
| L2 | ARB File Completeness |
| L3 | intl / l10n Setup |
| L4 | Pluralization & Gender |
| L5 | RTL Support |
| L6 | Date, Time & Number Formatting |
| L7 | String Interpolation Issues |
| L8 | Locale Fallback Configuration |

---

## Installation

```bash
npx skills add NoaTubic/flutter-localization-audit-skill
```

Or add it manually by copying `SKILL.md` and `references/` into your Claude Code skills directory.

---

## Usage

### Full audit (all 8 checks)

```
/flutter-localization-audit
```

### Audit specific categories only

Pass a comma-separated list of category IDs:

```
/flutter-localization-audit hardcoded
/flutter-localization-audit arb,rtl
/flutter-localization-audit hardcoded,arb,plurals,rtl
```

Only the listed categories will be run. The report header notes which categories were in scope.

---

## Output

The skill saves a `localization_audit_report.md` file in your project root containing:

- **Translation coverage table** — per-locale key coverage percentage
- **Executive Summary** — severity counts and overall l10n health
- **Per-category findings** — file path, line number, code/ARB evidence, and a one-sentence fix
- **Positive findings** — what's already properly localized
- **Recommended next steps** — 3–5 highest-impact fixes, ordered by urgency

### Severity levels

| Level | Criteria |
|-------|----------|
| 🔴 Critical | App crashes on unsupported locale, broken localization setup |
| 🟠 High | Large number of hardcoded strings, missing keys in primary locales, no RTL support for RTL markets |
| 🟡 Medium | Missing plural forms, locale-unaware date/currency formatting, sentence splitting |
| 🔵 Low | Minor missing translations in secondary locales, orphaned ARB keys |
| ℹ️ Info | Coverage percentages, suggestions |

---

## Requirements

- [Claude Code](https://claude.ai/code) with skills support
- A Flutter project with `pubspec.yaml` and ARB files

---

## License

MIT — see [LICENSE](LICENSE)
