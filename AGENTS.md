# AGENTS.md — updating Masrefy SMS rules over the CDN

This repo hosts `sms-rules.json`, the remotely-updatable SMS parsing rules for
the **Masrefy** iOS app. The app fetches this file at runtime, so **adding or
fixing a rule here reaches users with no App Store update.**

If you are an agent asked to "add / fix / update an SMS rule," this is the
entire procedure. Follow it top to bottom.

---

## TL;DR

```sh
# 1. edit sms-rules.json  (add/fix a rule, bump "rulesVersion")
# 2. validate + commit + push:
python3 -m json.tool sms-rules.json >/dev/null && echo "JSON OK"   # must pass
git add sms-rules.json && git commit -m "rules: <what changed> (v<N>)" && git push
# 3. (Action auto-purges the CDN; to force it now:)
curl -sS "https://purge.jsdelivr.net/gh/mzelzoghbi/masrefy-config@main/sms-rules.json"
# 4. verify the CDN serves the new version:
curl -sS "https://cdn.jsdelivr.net/gh/mzelzoghbi/masrefy-config@main/sms-rules.json" \
  | python3 -c "import sys,json;print('rulesVersion',json.load(sys.stdin)['rulesVersion'])"
```

---

## The one rule you must not forget

**Bump the version number, or nothing happens.**

- Changed anything under `rules` → increment `rulesVersion` by 1.
- Changed anything under `banks` → increment `banksVersion` by 1.

Devices only re-apply the manifest when the fetched version is **strictly
greater** than the version they've already applied. Edit the rules but forget
the bump, and every device ignores your change. Versions are monotonic — always
go **up**, never reuse or lower a number (see [Rolling back](#rolling-back)).

---

## File layout

`sms-rules.json`:

```jsonc
{
  "schemaVersion": 1,     // format version of THIS file — DO NOT change unless the shape changes
  "rulesVersion": 16,     // bump when you touch "rules"
  "banksVersion": 4,      // bump when you touch "banks"
  "banks": [ ... ],
  "rules": [ ... ]
}
```

### A `rules[]` entry

| Field | Type | Notes |
| --- | --- | --- |
| `key` | string | **Stable, unique id.** Never reuse or rename an existing key — the app matches on it to update built-ins without clobbering a user's edited copy. New rule → new key. |
| `name` | string | Human label shown in the app's rule list. |
| `pattern` | string | An `NSRegularExpression` pattern, matched case-insensitively. **Backslashes must be JSON-escaped** (`\d` → `\\d`, `\s` → `\\s`). |
| `type` | string | `"income"` or `"expense"`. |
| `amountGroup` | int | 1-based capture-group index holding the amount. |
| `merchantGroup` | int | 1-based capture-group index for the merchant; `0` = none (typical for income / transfers). |
| `priority` | int | Lower = tried first. Give a more specific rule a lower number than a generic one it might otherwise lose to. |
| `bankKey` | string | **Must** equal some `banks[].key`. |

### A `banks[]` entry

| Field | Type | Notes |
| --- | --- | --- |
| `key` | string | Stable id, referenced by `rules[].bankKey`. |
| `nameEn` / `nameAr` | string | Display names. |
| `colorHex` | string | 6 hex digits, **no** leading `#`. |
| `senderHint` | string | When an SMS's sender matches this, the bank's rules are tried first. |
| `sortOrder` | int | Order in the bank grid. |

---

## Writing a good pattern

1. **Start from a real SMS.** Get the exact message body you want to parse.
2. **Anchor on a verb/keyword** that fixes the direction (e.g. `debited`,
   `credited`, `تم خصم`, `تم إيداع`) so an expense rule can't fire on an income
   SMS and vice-versa.
3. **Make the amount group tight:** `([0-9][0-9,]*(?:\\.[0-9]{1,2})?)` matches
   `1,234.56`. The currency token and card last-4 are picked up by the app's
   own post-passes — you usually only need amount (+ merchant for expenses).
4. **Keep merchant captures lazy** and stop them at the next anchor (a date, a
   `for`, an `on`, etc.) so trailing text doesn't leak into the merchant.
5. **Escape for JSON.** Every `\` in the regex becomes `\\` in the file. This is
   the most common mistake. Validate (below) before pushing.

### Test the regex before you push

The app uses ICU regex (`NSRegularExpression`), which Swift matches most
closely. A quick pre-flight in Python catches escaping/typo errors (ICU and
Python differ on some edge syntax, but a compile + match here catches ~all
mistakes):

```sh
python3 - <<'PY'
import json, re
m = json.load(open("sms-rules.json"))
sms = "Your credit card ending with#4012 was charged for EGP 667.00 at HYPER AHL ALSUF on 26/04/26"
for r in m["rules"]:
    try:
        rx = re.compile(r["pattern"], re.IGNORECASE)
    except re.error as e:
        print("BAD REGEX", r["key"], "->", e); continue
    hit = rx.search(sms)
    if hit:
        print("MATCH", r["key"], "groups=", hit.groups())
PY
```

---

## Full procedure

1. **Edit** `sms-rules.json` — add/fix the rule (new unique `key`), point
   `bankKey` at an existing bank (or add the bank + bump `banksVersion`).
2. **Bump** `rulesVersion` (and/or `banksVersion`).
3. **Validate**: `python3 -m json.tool sms-rules.json >/dev/null` must exit 0.
   Optionally run the regex pre-flight above.
4. **Commit & push** to `main`.
5. **Purge** the CDN — the `Purge jsDelivr cache` GitHub Action fires on push;
   to do it by hand:
   `curl -sS "https://purge.jsdelivr.net/gh/mzelzoghbi/masrefy-config@main/sms-rules.json"`
6. **Verify** both endpoints report the new version:
   - `https://cdn.jsdelivr.net/gh/mzelzoghbi/masrefy-config@main/sms-rules.json`
   - `https://raw.githubusercontent.com/mzelzoghbi/masrefy-config/main/sms-rules.json`

### How fast does it reach users?

- The app refetches on a throttle, and **immediately re-fetches when an SMS
  arrives that no current rule can parse** (a parse miss forces a refresh +
  retry). So a newly-added rule typically applies on the very next matching
  message.
- The in-app **SMS Rules screen shows the local vs. available version** and an
  up-to-date / update-available badge — the first place to look when debugging
  "did my rule ship?".

---

## Rolling back

Versions must only ever increase, so you cannot "un-bump." To revert a bad rule:

1. Restore the previous `pattern` (or remove the offending rule).
2. **Bump `rulesVersion` again** (to a new, higher number).
3. Commit, push, purge.

`git revert` of the content is fine — just make sure the net result still has a
`rulesVersion` **higher** than any version already released, or devices that
saw the bad version won't pick up the fix.

---

## Keep the app's bundled fallback in mind

The app ships a compiled-in copy of these rules (`SMSRuleSeeds.swift` /
`SMSBankSeeds.swift`) as an **offline fallback** for fresh installs with no
network. This CDN file is the live, always-preferred source. When the app team
cuts a new build they should re-sync the bundled seeds from this file so the two
don't drift — but you do **not** need the app repo to push a rule here.
