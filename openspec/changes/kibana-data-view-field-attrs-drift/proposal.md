## Why

`elasticstack_kibana_data_view` is forced into replacement on every `terraform plan` after the data view has been used in Kibana, as reported in issue [#1287](https://github.com/elastic/terraform-provider-elasticstack/issues/1287). Kibana maintains `field_attrs[<field>].count` (field popularity) server-side and mutates it as users interact with Discover. The current schema pairs `field_attrs` with a `RequiresReplace` plan modifier, so any server-side drift of `count` forces the entire resource to be destroyed and recreated. Recreation issues a new saved object id, which silently breaks dashboards and other saved objects that reference the data view by id.

## What Changes

- Drop the `RequiresReplace` plan modifier from `data_view.field_attrs` so differences never force destroy-and-recreate. Updates in this field are already not forwarded to the Kibana Update API.
- Stop overwriting `data_view.field_attrs` from the Kibana API response during create/read/update state population. The provider SHALL preserve the plan/state value for this attribute so that server-maintained popularity counts do not re-enter Terraform state as drift.
- Out of scope: in-place propagation of user-edited `field_attrs` to Kibana (the existing Update API does not accept this field).

## Capabilities

### New Capabilities

- _(none)_

### Modified Capabilities

- `kibana-data-view`: change `field_attrs` from a replacement-triggering, round-tripped attribute to a plan-authoritative attribute that is not refreshed from the API.

## Impact

- Specs: delta under `openspec/changes/kibana-data-view-field-attrs-drift/specs/kibana-data-view/spec.md`.
- Provider behavior: `internal/kibana/dataview/schema.go` (plan modifier removal) and `internal/kibana/dataview/models.go` (`populateFromAPI` no longer reads `field_attrs` from the API response).
- Tests: `internal/kibana/dataview/models_test.go` updated expectations plus a regression asserting server-side `count` drift does not reach state.
- User-visible change: existing state with `field_attrs = null` and a server-side `count` will no longer produce a plan diff. Users who previously added `lifecycle.ignore_changes = [data_view.field_attrs]` as a workaround can remove it.
