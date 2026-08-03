<img src="https://kaeris.dev/icon-512.png" alt="KAERIS" width="72" align="left" style="margin-right:16px" />

# KAERIS i18n Translate — GitHub Action

Translate new or changed i18n keys with AI, or gate CI on translation completeness — **46 languages, incremental, placeholder-safe.**

Two modes:

- **`translate`** — translates the keys that are new or changed since the last run and writes the locale files, then emits a `changed` flag and a Markdown `summary` so you can open a PR only when something actually changed.
- **`check`** — a static, offline **i18n firewall**: no API call, fails the build when a locale is missing keys or a translation broke a placeholder. Free, no key required.

Works with JSON, YAML, `.strings`, `.po`, ARB, Android XML, CSV, XLIFF, `.properties`, `.resx` and Mozilla Fluent for translation; the `check` firewall currently gates JSON and ARB (more formats land next release).

## Usage

### Gate every PR (i18n firewall — no API key)

```yaml
# .github/workflows/i18n-check.yml
name: i18n check
on: [pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: RaiGanja/kaeris-action@v1
        with:
          mode: check
          source: locales/en.json
          languages: es,fr,de,ja
```

A broken or incomplete translation now fails the check and can't merge. Add it as a required status check in branch protection to make it a hard gate.

### Auto-translate new strings into a PR

```yaml
# .github/workflows/i18n-translate.yml
name: i18n translate
on:
  push:
    branches: [main]
    paths: ['locales/en.json']
jobs:
  translate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - id: kaeris
        uses: RaiGanja/kaeris-action@v1
        with:
          mode: translate
          source: locales/en.json
          languages: es,fr,de,ja
          api-key: ${{ secrets.KAERIS_API_KEY }}   # omit for the free anonymous tier
      - if: steps.kaeris.outputs.changed == 'true'
        uses: peter-evans/create-pull-request@v6
        with:
          title: "i18n: update translations"
          body: ${{ steps.kaeris.outputs.summary }}
          branch: i18n/update
```

## Inputs

| Input | Description | Default |
|---|---|---|
| `mode` | `translate` (write locale files) or `check` (i18n firewall, no API call) | `translate` |
| `source` | Source strings file, e.g. `locales/en.json` | — (required) |
| `languages` | Comma-separated target languages, e.g. `es,fr,de,ja` | — (required) |
| `out` | Directory for translated / target locale files | source's dir |
| `api-key` | KAERIS API key (Pro/Team). Omit for the free anonymous tier. Unused in `check`. | — |
| `openrouter-key` | OpenRouter key for Lifetime/BYOK plans. Unused in `check`. | — |
| `only-new` | `translate`: only translate keys missing/changed since the last run (uses `kaeris.lock`) | `true` |
| `tone` | `translate`: `neutral`, `formal`, or `casual` | `neutral` |
| `icu` | `translate`: hint that strings use ICU MessageFormat (plurals/select) so it's preserved | `false` |
| `strict` | `check`: also fail on extra/stale keys not in the source | `false` |
| `api-url` | API base URL | `https://kaeris.dev` |

## Outputs

| Output | Description |
|---|---|
| `changed` | `"true"` if `translate` wrote/updated any locale content — open a PR only when this is true |
| `summary` | Markdown summary (keys translated, per-language matrix, placeholder/overflow counts) — use as a PR body |
| `missing` | Total missing-key count across all target languages |
| `placeholder-issues` | Total placeholder-mismatch count (should be `0`) |
| `check-exit-code` | Exit code of `kaeris check` (0 clean, 1 problem, 2 bad usage) |

## Findings inside the pull request (SARIF)

`check` can emit a SARIF 2.1.0 report. Upload it and every finding lands as an annotation on
the exact line of the locale file — in the diff, next to the human reviewer's comments —
instead of hiding in the job log.

```yaml
- uses: RaiGanja/kaeris-action@v1
  with:
    mode: check
    source: locales/en.json
    languages: de,fr,es
    sarif-file: kaeris.sarif

- uses: github/codeql-action/upload-sarif@v3
  if: always()            # upload the report even when the gate failed — that is when it matters
  with:
    sarif_file: kaeris.sarif
```

The report is written even on a clean run (an empty result set), because `upload-sarif`
fails the workflow if the file is missing, and "nothing was wrong" must not look like "the
step is broken".

Note on visibility: code-scanning uploads are free on public repositories; on private ones
they need GitHub Advanced Security. Without it the gate still works exactly as before — the
job fails and the log lists every finding — you just do not get the inline annotations.

## What it protects against

The deterministic checks (no API needed) catch the bugs that actually break apps: lost or renamed placeholders, number drift (`Delete 5 files` → `Delete 50 files`), the ICU/CLDR plural categories a target language requires, and UI-overflow risk. Every number in the PR summary comes from a real `kaeris check --json` run — nothing is fabricated.

## Links

- Website & docs: [kaeris.dev](https://kaeris.dev)
- CLI: `pip install kaeris` ([PyPI](https://pypi.org/project/kaeris/))
- MCP server (localize from Claude/Cursor): `pip install kaeris-mcp`

## License

MIT — see [LICENSE](./LICENSE).
