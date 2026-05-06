---
name: flutter-localization-audit
description: >
  Performs a comprehensive localization audit on a Flutter/Dart project and produces a
  structured, actionable markdown report. Use this skill whenever the user wants to:
  audit, scan, or review Flutter localization for completeness and correctness; find
  hardcoded strings that should be translated; check ARB file consistency across locales;
  verify proper intl/l10n setup; detect missing plural forms, RTL support issues,
  or locale-unaware date/number formatting.
  Also trigger when the user says things like "check my Flutter translations", "find
  hardcoded strings", "localization review", "l10n audit", "i18n check", "is my app
  RTL ready?", "missing translations", or "check my ARB files".
  Accepts optional category IDs as arguments to audit only specific checks
  (e.g. "hardcoded", "arb,rtl"). Valid IDs: hardcoded, arb, setup, plurals, rtl,
  formatting, interpolation, fallback.
license: MIT
metadata:
  author: NoaTubic
  version: "1.0"
---

## Parameters

The skill accepts an optional comma-separated list of category IDs to audit only specific checks.

| Argument | Example | Effect |
|---|---|---|
| _(none)_ | `/flutter-localization-audit` | Full audit — all 8 checks |
| Single category | `/flutter-localization-audit rtl` | Only RTL check |
| Multiple categories | `/flutter-localization-audit hardcoded,arb,rtl` | Only listed checks |

**Valid category IDs:**

| ID | What it checks |
|----|----------------|
| `hardcoded` | UI strings not going through the localization system |
| `arb` | Missing or inconsistent keys across ARB locale files |
| `setup` | intl / gen-l10n configuration correctness |
| `plurals` | Missing plural forms, zero cases, gender handling |
| `rtl` | Right-to-left layout support |
| `formatting` | Hardcoded date, time, and number formats |
| `interpolation` | Broken string concatenation across translated segments |
| `fallback` | Missing or misconfigured locale fallbacks |

If arguments are provided, skip all unlisted categories. The report header notes which categories were in scope.

---

# Flutter Localization Audit Skill

You are performing a structured localization audit on a Flutter/Dart project.
Your job is to read the codebase, ARB files, and localization config, find real evidence
of localization issues, and produce a clear prioritized markdown report the developer
can act on immediately.

Before starting, read the detailed check patterns in:
`references/localization_checks.md`

---

## Process

### Step 1 — Understand the project layout

Start with a high-level read before any searching:

- List the root directory; identify: `lib/`, `l10n/`, `assets/l10n/`, `*.arb` files, `l10n.yaml`, `pubspec.yaml`
- Read `l10n.yaml` if present — note `arb-dir`, `template-arb-file`, `output-localization-file`, `nullable-getter`
- Read `pubspec.yaml` — check `flutter.generate: true` and `intl` / `flutter_localizations` dependencies
- List all `.arb` files and identify which locales are supported
- Read the template ARB file in full (usually `app_en.arb`) — this is the source of truth

### Step 2 — Run all 8 audit categories

Work through every category in order (or only the ones requested via parameters). For each:

1. **Search** using the patterns in `references/localization_checks.md`
2. **Collect evidence** — file path, line number, the relevant code or ARB key
3. **Assess severity** using the table below
4. **Write a fix** — one concrete, actionable sentence

Cover all categories even when there are no findings — record "No issues found" for those.

---

### L1 — Hardcoded Strings

Find UI-visible strings that bypass the localization system and are embedded directly in widgets.

- Search for string literals inside common Flutter UI widgets: `Text(`, `AppBar(title:`, `SnackBar(content:`, `AlertDialog(title:`, `TextFormField(decoration:`, `Tooltip(message:`, `ElevatedButton(child:`
- Ignore: non-visible strings (route names, asset paths, log messages, analytics event names, regex patterns, enum values)
- Flag: any user-visible label, button text, error message, or placeholder that is a plain string literal
- Pay extra attention to: error messages, form validation messages, empty state messages — these are most commonly forgotten

### L2 — ARB File Completeness

Compare every `.arb` file against the template file to find missing, extra, or inconsistent keys.

- Read the template ARB (e.g. `app_en.arb`) and extract all keys
- Read every other locale ARB and compare keys
- Flag: keys present in template but missing in one or more locales
- Flag: keys present in a locale file but absent from the template (orphaned keys)
- Flag: keys where the placeholder count or names differ between locales (e.g. template has `{name}` but French file uses `{nom}`)
- Flag: keys with empty string values `""` in any locale
- Report a coverage percentage per locale: `translated / total template keys × 100`

### L3 — intl / l10n Setup

Verify the localization infrastructure is configured correctly end-to-end.

- Check `pubspec.yaml` for: `flutter_localizations` in dependencies, `intl` in dependencies, `flutter.generate: true`
- Check `l10n.yaml` exists; if absent, check for `flutter gen-l10n` config in `pubspec.yaml`
- Check that generated files exist: `lib/l10n/` or the configured output directory should contain `app_localizations.dart` and locale-specific files
- Check `MaterialApp` (or `CupertinoApp`) has `localizationsDelegates` and `supportedLocales` wired up
- Flag: `flutter_localizations` missing from dependencies
- Flag: `flutter.generate: true` not set (codegen won't run automatically)
- Flag: `MaterialApp` missing `localizationsDelegates` or `supportedLocales`
- Flag: `l10n.yaml` missing with no alternative config

### L4 — Pluralization & Gender

Find ARB keys that handle counts or genders incorrectly or incompletely.

- Search ARB files for keys with `{count}` or `{num}` placeholders — these should use ICU plural syntax
- Flag: a count-bearing key that uses simple interpolation (`"You have {count} items"`) instead of ICU plural forms
- Flag: plural keys missing the `zero` case (many languages require it; English UX often benefits too)
- Flag: plural keys missing locale-specific forms (e.g. Arabic needs 6 plural forms; Russian needs 3; Polish needs 4)
- Check for any gender-dependent strings — flag if handled via concatenation rather than ICU select syntax

### L5 — RTL Support

Identify layout and styling code that assumes left-to-right direction and will break in Arabic, Hebrew, Farsi, etc.

- Search for hardcoded `EdgeInsets.only(left:` / `EdgeInsets.only(right:` — should use `EdgeInsetsDirectional`
- Search for `Alignment.centerLeft` / `Alignment.centerRight` — should use `Alignment.centerStart` / `Alignment.centerEnd`
- Search for `TextAlign.left` / `TextAlign.right` — should use `TextAlign.start` / `TextAlign.end`
- Search for hardcoded `CrossAxisAlignment.start` combined with explicit `left` padding
- Search for `Transform` or `Matrix4` that flips/rotates directional icons without checking `Directionality`
- Flag: any explicit `Directionality(textDirection: TextDirection.ltr)` that forces LTR unconditionally
- Check `MaterialApp` — flag if no RTL locale is in `supportedLocales` but the app targets Arabic/Hebrew markets

### L6 — Date, Time & Number Formatting

Find date, time, currency, and number values formatted without locale awareness.

- Search for `DateFormat(` — check if a locale is passed or if it uses the system default blindly
- Search for direct `.toString()` calls on `DateTime` objects in UI code
- Search for hardcoded date format strings like `'dd/MM/yyyy'`, `'MM/dd/yyyy'`, `'yyyy-MM-dd'` in UI widgets
- Search for number formatting like `'\$${amount}'`, `'€${price}'` — should use `NumberFormat.currency(locale:)`
- Search for `intl.DateFormat` without a locale argument
- Flag: any `DateTime` rendered in the UI without going through a locale-aware formatter
- Flag: currency symbols hardcoded as string prefixes/suffixes

### L7 — String Interpolation Issues

Find cases where translated strings are broken by Dart string concatenation — a pattern that works in English but fails in languages with different word order.

- Search for patterns like: `AppLocalizations.of(context).someKey + ' ' + otherValue`
- Search for: `'${AppLocalizations.of(context).someKey} ${variable}'` where the variable is a noun or value that changes word order in other languages
- Search for: multiple localized strings concatenated together to form a sentence
- Flag: any concatenation of a localized string with a variable that represents a noun, name, or count — these must use placeholders inside the ARB key instead

### L8 — Locale Fallback Configuration

Verify the app handles unsupported locales gracefully.

- Check `MaterialApp.localeResolutionCallback` or `localeListResolutionCallback` — flag if absent (Flutter's default fallback may not match your UX intent)
- Check `supportedLocales` — flag if it contains only one locale (no fallback possible)
- Check `l10n.yaml` for `untranslated-messages-file` — useful to track missing translations in CI
- Flag: no fallback locale defined when the app supports 3+ locales
- Flag: `nullable-getter: true` in `l10n.yaml` without null checks in the codebase — will cause runtime null errors on unsupported locales

---

## Severity Guidelines

| Severity | Criteria |
|----------|----------|
| 🔴 Critical | App crashes on unsupported locale, broken localization setup (no delegates wired, codegen not running) |
| 🟠 High | Large number of hardcoded UI strings, missing ARB keys in primary supported locales, no RTL support when targeting RTL markets |
| 🟡 Medium | Missing plural/zero forms, locale-unaware date/currency formatting, string concatenation across translated segments |
| 🔵 Low | Minor missing translations in secondary locales, orphaned ARB keys, missing locale fallback callback |
| ℹ️ Info | Coverage percentages per locale, suggestions for improvement |

---

## Important principles

- **Evidence only** — never report a finding without a concrete file path and code snippet or ARB key.
- **Ignore non-UI strings** — route names, log messages, analytics keys, asset paths, and internal identifiers are not localization issues.
- **Give credit** — note what IS properly localized, especially if the ARB setup is clean and coverage is high.
- **Coverage matters** — always report a per-locale translation coverage percentage in the executive summary.

---

## Report Template

```
# Flutter Localization Audit Report

**App:** [name from pubspec.yaml]
**Date:** [today's date]
**Auditor:** Claude (AI-assisted static analysis)
**Scope:** Dart source code, ARB files, l10n configuration
**Supported locales:** [list from MaterialApp or l10n.yaml]
**Template locale:** [e.g. en]
**Total ARB keys (template):** [count]

---

## Translation Coverage

| Locale | Translated | Missing | Coverage |
|--------|-----------|---------|----------|
| en (template) | N | 0 | 100% |
| [locale] | N | N | N% |

---

## Executive Summary

| Severity | Count |
|----------|-------|
| 🔴 Critical | N |
| 🟠 High | N |
| 🟡 Medium | N |
| 🔵 Low | N |
| ℹ️ Info | N |

[2–4 sentences: overall l10n health, most critical issues, and top recommendation.]

---

## Findings

### L1 — Hardcoded Strings
**Risk level:** Critical / High / Medium / Low / ✅ No issues found

[For each finding:]

**Finding [N]: Hardcoded string in [widget name]**
- **Severity:** 🟠 High
- **File:** `lib/path/to/file.dart` (line N)
- **Issue:** [What the problem is]
- **Evidence:**
  ```dart
  // relevant snippet
  ```
- **Fix:** [Concrete remediation — one sentence]

---

### L2 — ARB File Completeness
**Risk level:** Critical / High / Medium / Low / ✅ No issues found
[same format — list missing keys per locale]

### L3 — intl / l10n Setup
**Risk level:** Critical / High / Medium / Low / ✅ No issues found
[same format]

### L4 — Pluralization & Gender
**Risk level:** Critical / High / Medium / Low / ✅ No issues found
[same format]

### L5 — RTL Support
**Risk level:** Critical / High / Medium / Low / ✅ No issues found
[same format]

### L6 — Date, Time & Number Formatting
**Risk level:** Critical / High / Medium / Low / ✅ No issues found
[same format]

### L7 — String Interpolation Issues
**Risk level:** Critical / High / Medium / Low / ✅ No issues found
[same format]

### L8 — Locale Fallback Configuration
**Risk level:** Critical / High / Medium / Low / ✅ No issues found
[same format]

---

## Positive Findings

[List what is already done well — clean ARB setup, high coverage locales, proper ICU plurals, etc.]

---

## Recommended Next Steps

[Ordered list of 3–5 highest-impact fixes, most urgent first.]
```
