## 1. Spec

- [ ] 1.1 Validate the change with `./node_modules/.bin/openspec validate kibana-data-view-field-attrs-drift`.
- [ ] 1.2 Sync or archive the delta into `openspec/specs/kibana-data-view/spec.md` after implementation is verified.

## 2. Schema

- [ ] 2.1 Remove `mapplanmodifier.RequiresReplace()` from the `field_attrs` attribute in `internal/kibana/dataview/schema.go` and drop the now-unused import if applicable.

## 3. State reconciliation

- [ ] 3.1 Update `populateFromAPI` in `internal/kibana/dataview/models.go` so `field_attrs` is populated from the existing model (plan/state) rather than the API response, with a null fallback when the existing value is unknown.

## 4. Tests

- [ ] 4.1 Update `TestPopulateFromAPI` expectations in `internal/kibana/dataview/models_test.go` so they reflect that `field_attrs` is not refreshed from the response.
- [ ] 4.2 Add a regression case that models issue #1287: existing state has `field_attrs = null`, the API response carries `count`, and the resulting state keeps `field_attrs = null`.

## 5. CHANGELOG and verification

- [ ] 5.1 Add a CHANGELOG entry under `## [Unreleased]` describing the fix and pointing at issue #1287.
- [ ] 5.2 Run `make build`.
- [ ] 5.3 Run `go test ./internal/kibana/dataview/...` (unit).
