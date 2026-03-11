# Translate Views Skill

**Invocation**: `/translate-views` or tell Claude to "translate [feature/path] using the translate-views skill"

## What It Does

1. **Discovers** all supported locales in your Rails project
2. **Translates** missing keys to all target languages
3. **Updates** locale files (`config/locales/*.yml`)
4. **Validates** translations render correctly using Playwright
5. **Screenshots** the feature in multiple languages for visual proof

## Usage Examples

### Basic Usage
```
/translate-views /tickets
```

### Specify a view
```
translate the budgets wizard views
```

### Fix existing translations
```
translate /meetings and validate with playwright
```

### Custom locale scope
```
translate building_scores.banners.community_engagement
```

## What Gets Updated

- ✅ `config/locales/en.yml` (baseline)
- ✅ `config/locales/es.yml` (Spanish)
- ✅ `config/locales/ar.yml` (Arabic - RTL)
- ✅ `config/locales/pl.yml` (Polish)
- ✅ `config/locales/tl.yml` (Tagalog)
- ✅ `config/locales/zh.yml` (Chinese)
- ✅ Any hardcoded strings in views/helpers → extracted to locale files

## Validation

The skill uses **Playwright** to:
1. Navigate to the feature
2. Switch between languages using the language selector
3. Capture DOM snapshots to verify translations
4. Take screenshots in 2-3 languages for documentation
5. Verify no `translation missing:` errors

## Output

```
✅ Translations added for 6 locales
✅ 42 translation keys added per locale
✅ Playwright validation passed for ar, es, zh
📸 Screenshots: tickets-ar.png, tickets-es.png, tickets-zh.png
```

## When to Use

- ✅ Adding a new feature that needs i18n
- ✅ Discovered missing translations in non-English locales
- ✅ Found hardcoded strings that need extraction
- ✅ Need to validate existing translations render correctly
- ✅ Want visual proof (screenshots) for PR review

## Project Requirements

- Rails application with i18n configured
- `config/locales/` directory with locale files
- Playwright available for browser testing
- Language selector in the UI
- Dev login backdoor (e.g., `/dev/login/1`) for testing

## Tips

- The skill automatically detects which locales your project supports
- It preserves YAML formatting and structure
- Handles special cases: pluralization, interpolation, HTML-safe strings
- Tests RTL layouts for Arabic
- Catches layout issues (text overflow, broken UI) early
