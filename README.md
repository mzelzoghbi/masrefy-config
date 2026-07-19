# masrefy-config

Remotely-updatable configuration for the **Masrefy** iOS app.

This repo is **public on purpose** — it contains only bank-SMS parsing rules
(regular-expression patterns and bank metadata). No secrets, no keys, no user
data. Making it public lets the app fetch it over a free CDN with no backend.

## `sms-rules.json`

The manifest of built-in SMS parsing rules + banks. It mirrors the app's
compiled-in seeds (`SMSRuleSeeds.swift` / `SMSBankSeeds.swift`), which stay in
the app bundle as an offline fallback. At runtime the app fetches this file,
and — when its version is newer than what's stored — merges the built-ins into
its local store **without touching user-created/edited rules** (matched by
`key`, same machinery as the in-app seeder).

### URLs the app fetches

| Role | URL |
| --- | --- |
| Primary (CDN, cached ≤12h) | `https://cdn.jsdelivr.net/gh/mzelzoghbi/masrefy-config@main/sms-rules.json` |
| Fallback (no CDN cache) | `https://raw.githubusercontent.com/mzelzoghbi/masrefy-config/main/sms-rules.json` |

### Schema

```jsonc
{
  "schemaVersion": 1,          // format version of THIS file (bump only if the shape changes)
  "rulesVersion": 16,          // bump to trigger a rules refresh on devices
  "banksVersion": 4,           // bump to trigger a banks refresh on devices
  "banks": [
    {
      "key": "CIB",            // stable id; matches rule.bankKey
      "nameEn": "CIB",
      "nameAr": "سي آي بي",
      "colorHex": "A8443E",    // 6-hex, no '#'
      "senderHint": "CIB",     // biases parser priority when the SMS sender matches
      "sortOrder": 0
    }
  ],
  "rules": [
    {
      "key": "cib_credit_card",     // stable id; used to merge without clobbering user forks
      "name": "Credit card · English",
      "pattern": "(?:EGP|LE|...)\\s*([0-9]...)...",  // NSRegularExpression pattern (case-insensitive)
      "type": "expense",            // "income" | "expense"
      "amountGroup": 1,             // 1-based capture-group index (0 = unused)
      "merchantGroup": 2,           // 0 for income / transfers with no merchant
      "priority": 10,               // lower = tried first
      "bankKey": "CIB"              // must reference a banks[].key
    }
  ]
}
```

## How to add or change a rule (no app update required)

1. Edit `sms-rules.json` — add a rule to `rules` (or a bank to `banks`).
   - Pick a new, unique, stable `key`.
   - `pattern` is an `NSRegularExpression` pattern. Backslashes must be
     JSON-escaped (`\s` → `\\s`). Test it against a real SMS body first.
   - Make sure `bankKey` points at an existing `banks[].key`.
2. **Bump `rulesVersion`** (or `banksVersion`) by 1. Devices only refresh when
   the fetched version is higher than what they've already applied.
3. Commit to `main`. The Action below purges the jsDelivr cache automatically.
4. Devices pick it up on their next throttled fetch, or immediately on the next
   SMS that no current rule can parse (the app force-refetches on a parse miss).

> Keep this file in sync with the app's bundled seeds when you cut a new build,
> so a fresh install with no network still parses everything.

## jsDelivr cache

jsDelivr caches the `@main` URL for up to ~12h. `.github/workflows/purge-jsdelivr.yml`
calls the purge endpoint on every push to `main` so a new rule goes live fast.
You can also purge manually:

```sh
curl -sS https://purge.jsdelivr.net/gh/mzelzoghbi/masrefy-config@main/sms-rules.json
```
