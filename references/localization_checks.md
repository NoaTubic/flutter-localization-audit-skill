# Flutter Localization Audit — Check Patterns

This reference file contains what to search for in Dart source, ARB files,
and config files for each audit category.

---

## L1 — Hardcoded Strings

### Search patterns

```bash
# Text widgets with string literals (not variables or localization calls)
grep -rn "Text('" lib/
grep -rn 'Text("' lib/

# AppBar titles
grep -rn "title: Text('" lib/
grep -rn 'title: Text("' lib/

# Hints, labels, helper text in input decorations
grep -rn "hintText: '" lib/
grep -rn 'hintText: "' lib/
grep -rn "labelText: '" lib/
grep -rn 'labelText: "' lib/
grep -rn "helperText: '" lib/
grep -rn "errorText: '" lib/

# Buttons
grep -rn "ElevatedButton.*Text('" lib/
grep -rn "TextButton.*Text('" lib/
grep -rn "OutlinedButton.*Text('" lib/

# Dialogs and overlays
grep -rn "AlertDialog" lib/ | grep -i "title\|content"
grep -rn "SnackBar.*content.*Text('" lib/
grep -rn "Tooltip.*message: '" lib/

# Validation messages
grep -rn "validator:.*return '" lib/
grep -rn "validator:.*return \"" lib/
```

### What to flag
- Any `Text('Some visible label')` — the string is a plain literal
- `hintText: 'Enter your name'` style decorations
- Validation error returns like `return 'Please enter a valid email'`
- `SnackBar(content: Text('Success!'))` style feedback

### What to ignore
- `Text(someVariable)` — using a variable, may already be localized
- `Text(AppLocalizations.of(context)!.someKey)` — correctly localized
- Route name strings: `'/home'`, `'/login'`
- Asset paths: `'assets/images/logo.png'`
- Log/debug strings: inside `debugPrint()`, `print()`, `log()`
- Analytics/event names: typically camelCase identifiers
- Regex patterns, format strings used for parsing (not display)

---

## L2 — ARB File Completeness

### Search patterns

```bash
# Find all ARB files
find . -name "*.arb" -not -path "*/node_modules/*"

# Count keys in template ARB
grep -c '"[a-zA-Z]' l10n/app_en.arb 2>/dev/null || grep -c '"[a-zA-Z]' lib/l10n/app_en.arb 2>/dev/null

# List all keys in template
grep -E '^\s+"[a-zA-Z][^@]' l10n/app_en.arb | grep -v "@"

# Find empty string values
grep -E '"[a-zA-Z][^"]*":\s*""' l10n/
```

### How to compare ARB files
1. Extract all non-metadata keys from the template (lines matching `"keyName":` that don't start with `@`)
2. For each other locale ARB, check which template keys are absent
3. Report: missing keys per locale, orphaned keys (in locale but not template), empty values

### Key metadata to check
```json
{
  "greeting": "Hello {name}!",
  "@greeting": {
    "description": "Greeting with user name",
    "placeholders": {
      "name": {
        "type": "String"
      }
    }
  }
}
```
- Flag: keys with `{placeholder}` in template but different placeholder names in locale files
- Flag: missing `@metadata` for keys with placeholders (prevents proper code generation)

---

## L3 — intl / l10n Setup

### pubspec.yaml checks

```bash
# Check for flutter_localizations dependency
grep "flutter_localizations" pubspec.yaml

# Check for intl dependency
grep "intl:" pubspec.yaml

# Check flutter.generate
grep -A 5 "^flutter:" pubspec.yaml | grep "generate"
```

### l10n.yaml checks

```bash
# Read full l10n.yaml
cat l10n.yaml 2>/dev/null || echo "l10n.yaml not found"
```

Expected `l10n.yaml` contents:
```yaml
arb-dir: lib/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
```

### MaterialApp wiring check

```bash
# Check for localizationsDelegates
grep -rn "localizationsDelegates" lib/

# Check for supportedLocales
grep -rn "supportedLocales" lib/

# Check for AppLocalizations import
grep -rn "AppLocalizations\|app_localizations" lib/main.dart lib/app.dart 2>/dev/null
```

Expected wiring:
```dart
MaterialApp(
  localizationsDelegates: AppLocalizations.localizationsDelegates,
  supportedLocales: AppLocalizations.supportedLocales,
)
```

### Generated files check

```bash
# Check if generated localization files exist
find . -name "app_localizations*.dart" -not -path "*/.dart_tool/*"
find . -name "*.g.dart" -path "*/l10n/*"
```

### What to flag
- `flutter_localizations` absent from `pubspec.yaml` → High
- `flutter.generate: true` missing → High (codegen won't run on `flutter pub get`)
- `l10n.yaml` missing with no alternative config → Medium
- `MaterialApp` without `localizationsDelegates` → Critical
- `MaterialApp` without `supportedLocales` → High
- Generated files missing (may be in `.gitignore` — check if generation is wired in CI) → Medium

---

## L4 — Pluralization & Gender

### Search patterns

```bash
# Find ARB keys that likely need pluralization (contain count/num/amount)
grep -E '"[^"]*[Cc]ount[^"]*":|"[^"]*[Nn]um[^"]*":|"[^"]*[Aa]mount[^"]*":' l10n/app_en.arb 2>/dev/null

# Find Dart code with count interpolation in localized strings
grep -rn "\.tr\|AppLocalizations" lib/ | grep -E "\{count\}|\{num\}|\{n\}"

# Find simple interpolation of counts (bad pattern)
grep -rn '"\${.*count\|num.*}"' lib/
```

### ICU plural syntax (correct)
```json
{
  "itemCount": "{count, plural, =0{No items} =1{One item} other{{count} items}}",
  "@itemCount": {
    "placeholders": {
      "count": { "type": "int" }
    }
  }
}
```

### What to flag
- ARB key with `{count}` but no ICU `plural` syntax → Medium
- Missing `=0{...}` case in plural keys → Low
- Locale files for Arabic (ar) missing 6 plural forms: `zero`, `one`, `two`, `few`, `many`, `other` → Medium
- Locale files for Russian (ru), Polish (pl), Czech (cs) missing appropriate plural forms → Medium
- Gender-dependent strings concatenated as separate keys instead of using ICU `select` → Low

### Plural forms required by language
| Language | Required forms |
|----------|---------------|
| English | `one`, `other` |
| French | `one`, `other` |
| Russian | `one`, `few`, `many`, `other` |
| Arabic | `zero`, `one`, `two`, `few`, `many`, `other` |
| Polish | `one`, `few`, `many`, `other` |
| Chinese, Japanese, Korean | `other` only |

---

## L5 — RTL Support

### Search patterns

```bash
# Hardcoded left/right padding (should be start/end)
grep -rn "EdgeInsets\.only(left:" lib/
grep -rn "EdgeInsets\.only(right:" lib/
grep -rn "EdgeInsets\.fromLTRB" lib/

# Hardcoded left/right alignment
grep -rn "Alignment\.centerLeft\|Alignment\.topLeft\|Alignment\.bottomLeft" lib/
grep -rn "Alignment\.centerRight\|Alignment\.topRight\|Alignment\.bottomRight" lib/

# Hardcoded text alignment
grep -rn "TextAlign\.left\b" lib/
grep -rn "TextAlign\.right\b" lib/

# Forced LTR directionality
grep -rn "TextDirection\.ltr" lib/
grep -rn "Directionality.*ltr" lib/

# Icons that may need mirroring in RTL
grep -rn "Icons\.arrow_forward\b\|Icons\.arrow_back\b\|Icons\.chevron_right\b\|Icons\.chevron_left\b" lib/
```

### RTL-safe replacements

| Hardcoded | RTL-safe replacement |
|-----------|---------------------|
| `EdgeInsets.only(left: x)` | `EdgeInsetsDirectional.only(start: x)` |
| `EdgeInsets.only(right: x)` | `EdgeInsetsDirectional.only(end: x)` |
| `EdgeInsets.fromLTRB(l,t,r,b)` | `EdgeInsetsDirectional.fromSTEB(s,t,e,b)` |
| `Alignment.centerLeft` | `Alignment.centerStart` (Flutter 3.x) or `AlignmentDirectional.centerStart` |
| `TextAlign.left` | `TextAlign.start` |
| `TextAlign.right` | `TextAlign.end` |

### What to flag
- Any `EdgeInsets.only(left:` or `EdgeInsets.only(right:` → Medium per occurrence
- `TextAlign.left` / `TextAlign.right` in UI text → Low
- `Directionality(textDirection: TextDirection.ltr)` wrapping a whole screen → High
- Directional icons (`arrow_forward`, `chevron_right`) without RTL mirroring → Low

---

## L6 — Date, Time & Number Formatting

### Search patterns

```bash
# DateTime.toString() in UI code
grep -rn "\.toString()" lib/ | grep -i "date\|time\|createdAt\|updatedAt"

# Hardcoded date format strings
grep -rn "DateFormat('" lib/
grep -rn 'DateFormat("' lib/

# Currency hardcoded as string prefix
grep -rn '"\$\${' lib/
grep -rn '"€\${' lib/
grep -rn '"£\${' lib/

# Number formatting without locale
grep -rn "\.toStringAsFixed" lib/
grep -rn "NumberFormat(" lib/
```

### What to flag
- `DateFormat('dd/MM/yyyy')` without a locale argument → Medium
- `dateTime.toString()` rendered directly in a `Text` widget → Medium
- `'\$${amount.toStringAsFixed(2)}'` — hardcoded currency symbol → Medium
- `NumberFormat('#,##0.00')` without locale → Low

### Correct patterns
```dart
// Date with locale
DateFormat.yMd(Localizations.localeOf(context).toString()).format(date)
DateFormat.yMMMMd(locale).format(date)

// Currency with locale
NumberFormat.currency(locale: locale, symbol: currencySymbol).format(amount)

// Relative time (use timeago or intl's RelativeDateTimeSymbols)
```

---

## L7 — String Interpolation Issues

### Search patterns

```bash
# Localized string concatenated with + operator
grep -rn "AppLocalizations.*+\|\.tr.*+" lib/

# Localized string in string interpolation with a variable noun
grep -rn '"\${.*AppLocalizations\|"\${.*\.tr' lib/

# Multiple l10n calls on same line (may indicate sentence splitting)
grep -rn "AppLocalizations.*AppLocalizations\|\.tr.*\.tr" lib/
```

### What to flag

Bad pattern — breaks in languages with different word order:
```dart
// BAD
Text(AppLocalizations.of(context)!.hello + ' ' + userName)
Text('${AppLocalizations.of(context)!.welcomeBack} ${user.name}')
```

Correct pattern — use a placeholder inside the ARB key:
```dart
// GOOD
Text(AppLocalizations.of(context)!.helloUser(userName))
// ARB: "helloUser": "Hello {name}!", with placeholder defined
```

- Concatenation of localized string + user-supplied noun → High (broken in most non-English languages)
- Concatenation of two localized strings to form a sentence → Medium

---

## L8 — Locale Fallback Configuration

### Search patterns

```bash
# Check for localeResolutionCallback
grep -rn "localeResolutionCallback\|localeListResolutionCallback" lib/

# Check supportedLocales count
grep -A 10 "supportedLocales" lib/main.dart lib/app.dart 2>/dev/null

# Check l10n.yaml for fallback config
grep "untranslated-messages\|fallback" l10n.yaml 2>/dev/null

# Check for nullable getter setting
grep "nullable-getter" l10n.yaml 2>/dev/null
```

### What to flag
- No `localeResolutionCallback` when app supports 3+ locales → Low
- `nullable-getter: true` in `l10n.yaml` without null-safety guards in code → High
- Only one locale in `supportedLocales` → Medium (implies no fallback)
- No `untranslated-messages-file` in `l10n.yaml` for projects with active translation work → Info

### Recommended fallback setup
```dart
MaterialApp(
  localizationsDelegates: AppLocalizations.localizationsDelegates,
  supportedLocales: AppLocalizations.supportedLocales,
  localeResolutionCallback: (locale, supportedLocales) {
    for (final supported in supportedLocales) {
      if (supported.languageCode == locale?.languageCode) return supported;
    }
    return supportedLocales.first; // fallback to default
  },
)
```
