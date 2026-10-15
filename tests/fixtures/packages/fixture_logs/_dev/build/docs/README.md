{{- generatedHeader }}
# Fixture Logs

Minimal integration package used to self-test the reusable CI and release
workflows in
[elastic-package-workflows](https://github.com/NateShoffner/elastic-package-workflows).
It is not useful for collecting real data.

## Data streams

The `sample` data stream reads lines shaped like
`2026-01-02T03:04:05.000Z INFO fixture started` from a file and parses them into
`level` and `detail`.

## Reference

### Inputs used

{{ inputDocs }}

### Fields

{{ fields "sample" }}
