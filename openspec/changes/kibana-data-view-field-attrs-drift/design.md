## Context

`data_view.field_attrs` is a map of per-field metadata that Kibana surfaces on data views. It mixes user-authored values (`custom_label`) with a server-maintained value (`count`, the popularity counter updated whenever a field is used in Discover). Today the schema marks the whole map with `mapplanmodifier.RequiresReplace()`, and `populateFromAPI` overwrites the modelled value with whatever the Data Views API returned. The combination causes a perpetual replace-forcing diff as soon as Kibana bumps any `count`, as reported in issue #1287. Replacement issues a new saved object id and breaks dashboards that reference the data view by id (see user report on the issue).

Meanwhile `toAPIUpdateModel` already does **not** send `field_attrs` to the Kibana Update API, so in-place updates to this field have never been possible via this resource. The attribute is effectively write-once on Create.

## Goals / Non-Goals

**Goals:**

- Stop triggering resource replacement on `field_attrs` drift.
- Stop letting Kibana's server-maintained `count` values reach Terraform state during Read.
- Keep existing create-time behavior: if the user supplies `field_attrs` in config, that value is what ends up in state.

**Non-Goals:**

- Making `field_attrs` updatable after create (the upstream Update API does not support it; a separate change would be needed).
- Removing or restructuring the `count` sub-attribute from the schema (would be a breaking change to existing configs).
- Any change to how `field_formats`, `runtime_field_map`, or other sibling attributes are reconciled.

## Decisions

| Topic | Decision | Alternatives considered |
|-------|----------|-------------------------|
| Replacement trigger | Remove `mapplanmodifier.RequiresReplace()` from `field_attrs`. Differences become in-place updates (effectively no-ops, since Update does not send the field). | Keep replacement but add a custom plan modifier that suppresses server-only diffs. Rejected as more complex, and replacement semantics are themselves harmful per the issue. |
| State reconciliation | Preserve the plan/state value for `field_attrs` and ignore the API response for this field during `populateFromAPI`. | Drop only `count` from the API mapping. Rejected because the sibling `custom_label` is also only set at create time, so there is no practical difference, and preserving the whole map is simpler and avoids partial-diff edge cases. |
| Null semantics | When the existing model does not have a known `field_attrs` value (e.g. brand-new plan, import), write `types.MapNull(getFieldAttrElemType())` so the state always has a well-typed null rather than an unset map. | Populate from API on first import. Rejected because importing popularity counts would immediately manifest as plan drift against user config that omits `field_attrs`. |
| Schema backward compatibility | Keep `count` in the schema as `Optional`. Existing configs continue to plan and apply without changes. | Mark `count` `Computed` only or drop it. Both are breaking config changes. |

## Risks / Trade-offs

- [Risk] Users who imported a data view and relied on `field_attrs` being populated in state from Kibana will see those entries disappear after the next plan. -> Mitigation: call this out in the CHANGELOG entry and the spec requirement; users can opt back in by adding the map to config.
- [Risk] If a future Kibana version starts accepting `field_attrs` on Update, this change would still not round-trip values. -> Mitigation: this is explicitly out of scope; a follow-up change would extend `toAPIUpdateModel` and revisit state reconciliation.

## Migration Plan

- No state upgrader needed. Existing state where `field_attrs` is already populated will be kept on Read (since state is preserved, not overwritten). Subsequent plans show no diff for `field_attrs`.
- Users who added `lifecycle.ignore_changes = [data_view.field_attrs]` can safely remove the workaround after upgrading.
- Rollback is a normal code revert with no state implications.

## Open Questions

- None blocking the proposal.
