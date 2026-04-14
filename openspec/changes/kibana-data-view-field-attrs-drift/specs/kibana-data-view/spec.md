## MODIFIED Requirements

### Requirement: Lifecycle replacement fields (REQ-006)

Changes to `space_id`, `data_view.id`, or `data_view.allow_no_index` SHALL require resource replacement rather than an in-place update. Changes to `data_view.field_attrs` SHALL NOT require replacement; this attribute is reconciled per REQ-015.

#### Scenario: Replace on immutable data view id

- **GIVEN** an existing managed data view
- **WHEN** `data_view.id` changes in configuration
- **THEN** Terraform SHALL plan replacement for the resource

#### Scenario: field_attrs drift does not trigger replacement

- **GIVEN** a managed data view whose Terraform config does not set `data_view.field_attrs`
- **AND** Kibana has populated `field_attrs[<field>].count` server-side as users interact with the data view in Discover
- **WHEN** `terraform plan` runs
- **THEN** the plan SHALL NOT propose replacement for the resource
- **AND** the plan SHALL NOT show a diff on `data_view.field_attrs`

## ADDED Requirements

### Requirement: field_attrs is plan-authoritative and not round-tripped (REQ-015)

`data_view.field_attrs` SHALL be treated as a plan-authoritative attribute. During create, read, and update, the provider SHALL preserve the plan or prior-state value for `data_view.field_attrs` and SHALL NOT overwrite it with the value returned by the Kibana Data Views API. When the existing plan or state value for `field_attrs` is null or unknown, the provider SHALL write a null map of the `field_attrs` element type to Terraform state rather than populate it from the API response.

#### Scenario: Server-side count drift is not reflected in state

- **GIVEN** state where `data_view.field_attrs` is null
- **AND** the Kibana Data Views API returns `field_attrs` entries containing a server-maintained `count`
- **WHEN** the provider runs read
- **THEN** the provider SHALL leave `data_view.field_attrs` as null in Terraform state

#### Scenario: Plan-provided field_attrs are preserved after create

- **GIVEN** a create request where `data_view.field_attrs` is set in Terraform config
- **WHEN** the create call succeeds
- **THEN** Terraform state SHALL contain the `field_attrs` value from the plan
- **AND** the provider SHALL NOT overwrite those entries with any `field_attrs` returned by the Kibana create response
