---
name: ontology-authoring-cli
description: 'Create and evolve Fabric IQ Ontology (preview) items from CLI — define entity types, properties (including timeseries), relationship types, and bind them to OneLake lakehouse tables (static + timeseries) or Eventhouse / KQL database tables (timeseries only). Uses the Fabric item-definition REST API (Create Item / Update Item Definition) with `InlineBase64` parts. Use when the user wants to create a Fabric Ontology item; add or alter entity types, properties, or keys; add timeseries properties and bindings; bind an entity type to a lakehouse or Eventhouse table; add relationship types and contextualizations; or script ontology deployment from source. Triggers: "create fabric ontology", "add ontology entity type", "bind entity type to lakehouse", "bind entity type to eventhouse", "ontology timeseries binding", "add ontology relationship type", "ontology contextualization", "fabric iq ontology authoring", "update ontology definition"'
---

> **Update Check — ONCE PER SESSION (mandatory)**
> The first time this skill is used in a session, run the **check-updates** skill before proceeding.
> - **GitHub Copilot CLI / VS Code**: invoke the `check-updates` skill (e.g., `/fabric-skills:check-updates`).
> - **Claude Code / Cowork / Cursor / Windsurf / Codex**: read the local `package.json` version, then compare against remote via `git fetch origin main --quiet && git show origin/main:package.json` (or the GitHub API). If remote is newer, show the changelog and update instructions.
> - Skip if the check was already performed earlier in this session.

> **CRITICAL NOTES**
> 1. Ontology is **preview**. The item type value is `Ontology`. Features and wire format may change; validate against the current docs before production use.
> 2. To find the workspace details (including its ID) from workspace name: list all workspaces and use JMESPath filtering.
> 3. To find the item details (including its ID) from workspace ID, item type (`Ontology`), and item name: list all items of that type in that workspace and use JMESPath filtering.
> 4. Authoring a relationship type requires **two distinct entity types** that already exist in the ontology. The `source.entityTypeId` and `target.entityTypeId` values are the **entity type IDs you assigned**, not item IDs.
> 5. Data bindings reference a source table by `workspaceId`, `itemId`, `sourceTableName`, and — for lakehouse sources — `sourceSchema`. Lakehouse (`LakehouseTable`) sources carry the lakehouse item ID; Eventhouse (`KustoTable`) sources carry the **Eventhouse item ID** plus `clusterUri` and `databaseName`. Key column(s) on the source side must match the entity type's key property(ies). Eventhouse sources are `TimeSeries`-only; the static (`NonTimeSeries`) binding must come from a lakehouse.

# ontology-authoring-cli — Fabric Ontology Authoring via CLI

## Table of Contents

| Task                                           | Reference                                                                                                                                              | Notes                                                                        |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------- |
| Finding Workspaces and Items in Fabric         | [COMMON-CLI.md § Finding Workspaces and Items in Fabric](../../common/COMMON-CLI.md#finding-workspaces-and-items-in-fabric)                            | **Mandatory** — resolve workspace/item IDs before authoring                  |
| Fabric Topology & Key Concepts                 | [COMMON-CORE.md § Fabric Topology & Key Concepts](../../common/COMMON-CORE.md#fabric-topology--key-concepts)                                           | Workspace → Item hierarchy                                                   |
| Authentication & Token Acquisition             | [COMMON-CORE.md § Authentication & Token Acquisition](../../common/COMMON-CORE.md#authentication--token-acquisition)                                   | Use `https://api.fabric.microsoft.com` audience for control plane            |
| Core Control-Plane REST APIs                   | [COMMON-CORE.md § Core Control-Plane REST APIs](../../common/COMMON-CORE.md#core-control-plane-rest-apis)                                              | Create Item, Get/Update Item Definition                                      |
| Long-Running Operations (LRO)                  | [COMMON-CORE.md § Long-Running Operations (LRO)](../../common/COMMON-CORE.md#long-running-operations-lro)                                              | Item create/update returns an LRO                                            |
| Rate Limiting & Throttling                     | [COMMON-CORE.md § Rate Limiting & Throttling](../../common/COMMON-CORE.md#rate-limiting--throttling)                                                   |                                                                              |
| Authentication Recipes                         | [COMMON-CLI.md § Authentication Recipes](../../common/COMMON-CLI.md#authentication-recipes)                                                            | `az login`; token acquisition                                                |
| Fabric Control-Plane API via `az rest`         | [COMMON-CLI.md § Fabric Control-Plane API via az rest](../../common/COMMON-CLI.md#fabric-control-plane-api-via-az-rest)                                | **Always** pass `--resource https://api.fabric.microsoft.com`                |
| Long-Running Operations (LRO) Pattern          | [COMMON-CLI.md § Long-Running Operations (LRO) Pattern](../../common/COMMON-CLI.md#long-running-operations-lro-pattern)                                | Poll `Location` header until `Succeeded`                                     |
| Gotchas & Troubleshooting (CLI-Specific)       | [COMMON-CLI.md § Gotchas & Troubleshooting (CLI-Specific)](../../common/COMMON-CLI.md#gotchas--troubleshooting-cli-specific)                           | Token audience, shell escaping                                               |
| `az rest` Template                             | [COMMON-CLI.md § `az rest` Template](../../common/COMMON-CLI.md#az-rest-template)                                                                      |                                                                              |
| Definition Envelope (parts, payloadType)       | [ITEM-DEFINITIONS-CORE.md § Definition Envelope](../../common/ITEM-DEFINITIONS-CORE.md#definition-envelope)                                            | `InlineBase64` parts pattern used for Ontology                               |
| Ontology Definition Reference                  | [ONTOLOGY-AUTHORING-CORE.md § Definition Tree](../../common/ONTOLOGY-AUTHORING-CORE.md#definition-tree)                                                | Authoritative file/folder layout for the ontology item                       |
| EntityType & EntityTypeProperty schema         | [ONTOLOGY-AUTHORING-CORE.md § EntityType file](../../common/ONTOLOGY-AUTHORING-CORE.md#entitytype-file--entitytypesiddefinitionjson)                   | Allowed `valueType` values, key constraints, name regex                      |
| DataBinding schema + source-type mapping       | [ONTOLOGY-AUTHORING-CORE.md § DataBinding file](../../common/ONTOLOGY-AUTHORING-CORE.md#databinding-file--entitytypesiddatabindingsguidjson)           | Lakehouse & Eventhouse shapes; value-type mapping; binding rules             |
| RelationshipType + Contextualization schema    | [ONTOLOGY-AUTHORING-CORE.md § RelationshipType file](../../common/ONTOLOGY-AUTHORING-CORE.md#relationshiptype-file--relationshiptypesiddefinitionjson) | Source/target constraints, link table requirements                           |
| Ontology Concepts                              | [SKILL.md § Ontology Item Concepts](#ontology-item-concepts)                                                                                           | Entity types, properties, bindings, relationship types                       |
| Tool Stack                                     | [SKILL.md § Tool Stack](#tool-stack)                                                                                                                   |                                                                              |
| Connection                                     | [SKILL.md § Connection](#connection)                                                                                                                   | Discover workspace, lakehouse, ontology IDs                                  |
| Authoring Scope                                | [SKILL.md § Authoring Scope](#authoring-scope)                                                                                                         | Supported operations at a glance                                             |
| Authoring Mechanics (full reference)           | [authoring-mechanics.md](references/authoring-mechanics.md)                                                                                            | Envelope, IDs, create, entity types, bindings, relationships, update, verify |
| Worked Examples                                | [examples.md](references/examples.md)                                                                                                                  | End-to-end bash recipes (create → bind → relationship → timeseries)          |
| Preview & Confirm (mandatory before LRO write) | [preview-and-confirm.md](references/preview-and-confirm.md)                                                                                            | Mermaid proposal (greenfield) / change-set diff (brownfield)                 |
| Script Templates                               | [definition-script-templates.md](references/definition-script-templates.md)                                                                            | Bash / PowerShell fetch-mutate-send scaffolds                                |
| Must / Prefer / Avoid / Troubleshooting        | [SKILL.md § Must / Prefer / Avoid / Troubleshooting](#must--prefer--avoid--troubleshooting)                                                            | LLM decision rules                                                           |
| Agentic Workflows                              | [SKILL.md § Agentic Workflows](#agentic-workflows)                                                                                                     | Exploration-before-authoring, script generation                              |
| Agent Integration Notes                        | [SKILL.md § Agent Integration Notes](#agent-integration-notes)                                                                                         | How this skill composes with agents / other skills                           |

---

## Ontology Item Concepts

A Fabric Ontology item is authored as a **tree of JSON files** inside the item definition. Each file is carried as a part in the `parts[]` array of the Create/Update definition envelope (payloadType `InlineBase64`).

| Concept | Definition file path | Purpose |
|---|---|---|
| Ontology envelope | `definition.json` | Empty `{}`; required |
| Platform metadata | `.platform` | `{ "metadata": { "type": "Ontology", "displayName": "<name>" } }` |
| Entity type | `EntityTypes/{entityTypeId}/definition.json` | Name, namespace, key(s), display name property, properties[], timeseriesProperties[] |
| Entity type data binding | `EntityTypes/{entityTypeId}/DataBindings/{guid}.json` | Maps a lakehouse **or eventhouse** table to properties; `dataBindingType` = `NonTimeSeries` or `TimeSeries`. Eventhouse (`KustoTable`) sources are allowed **only** for `TimeSeries` |
| Entity type documents | `EntityTypes/{entityTypeId}/Documents/{name}.json` | Optional doc links |
| Entity type overviews | `EntityTypes/{entityTypeId}/Overviews/definition.json` | Optional widgets layout |
| Entity type resource links | `EntityTypes/{entityTypeId}/ResourceLinks/definition.json` | Optional Power BI / item links |
| Relationship type | `RelationshipTypes/{relTypeId}/definition.json` | Source + target entity type IDs, name |
| Relationship contextualization | `RelationshipTypes/{relTypeId}/Contextualizations/{guid}.json` | Source/target key bindings onto a lakehouse table |

Property `valueType` allowed values (exact): `String`, `Boolean`, `DateTime`, `Object`, `BigInt`, `Double`. Use `BigInt` — **not** `Int64` — for integers; there is no `Guid` value type (model GUIDs as `String`). Timeseries bindings require a timestamp column (source type `datetime` / `date` / `timestamp`) and a `TimeSeries` binding with `timestampColumnName`. See [ONTOLOGY-AUTHORING-CORE.md § EntityTypeProperty](../../common/ONTOLOGY-AUTHORING-CORE.md#entitytypeproperty) for the full source-column → `valueType` mapping.

---

## Tool Stack

| Tool                                                            | Purpose                                                                                      | Install                             |
| --------------------------------------------------------------- | -------------------------------------------------------------------------------------------- | ----------------------------------- |
| **az cli**                                                      | Fabric control-plane calls via `az rest` (Create/Get/Update Item Definition)                 | `winget install Microsoft.AzureCLI` |
| **jq**                                                          | JSON processing and payload extraction                                                       | `winget install jqlang.jq`          |
| **PowerShell `[Convert]::ToBase64String` / `FromBase64String`** | Base64 on Windows — prefer over `certutil -encode`, which wraps lines and adds header/footer | Built-in                            |

### Prerequisite Check

Before authoring, verify that `az` and `jq` are available. On Windows, `winget install` does not update the `PATH` in existing shell sessions — open a new terminal or refresh the path manually after installing.

```bash
# Bash
az version >/dev/null 2>&1  || { echo "Install az: https://aka.ms/installazurecli"; exit 1; }
jq --version >/dev/null 2>&1 || { echo "Install jq: sudo apt install jq  OR  brew install jq"; exit 1; }
```

```powershell
# PowerShell
if (-not (Get-Command az  -ErrorAction SilentlyContinue)) { Write-Error "Install az: winget install Microsoft.AzureCLI"; return }
if (-not (Get-Command jq  -ErrorAction SilentlyContinue)) { Write-Error "Install jq: winget install jqlang.jq  (then open a new terminal)"; return }
```

On Windows, prefer PowerShell for base64. Avoid `certutil -encode`: its output is line-wrapped with header/footer and must be post-processed before use as an `InlineBase64` payload.

> **⚠️ PowerShell `ConvertTo-Json` Warning**: PowerShell's `ConvertTo-Json` can silently reorder keys and serialize `$null` differently than JSON `null`, which can cause `ALMOperationImportFailed` errors on `updateDefinition`. To avoid this:
>
> 1. **Always use `[System.IO.File]::WriteAllText`** with `[System.Text.UTF8Encoding]::new($false)` to write JSON files — `Out-File` and `Set-Content` add a BOM that corrupts the payload.
> 2. **Build JSON with `jq`** instead of `ConvertTo-Json` where possible — `jq -nc` produces deterministic, compact JSON without PowerShell serialization quirks:
>    ```powershell
>    $json = '{}' | jq -nc --arg id "$ET_ID" --arg name "Site" '{id:$id,name:$name}'
>    ```
> 3. **Validate** the JSON before sending: `Get-Content envelope.json | jq .` — if `jq` fails, the payload is malformed.
> 4. **Use `-Depth 10`** on `ConvertTo-Json` — the default depth of 2 silently truncates nested objects.

---

## Connection

Ontology authoring targets the Fabric control plane — not a data-plane endpoint. Resolve the workspace ID and the lakehouse item ID of the table(s) you will bind to **before** composing the definition.

```bash
# 1. Log in
az login

# 2. Resolve workspace ID from name
WS_NAME="Contoso-Analytics"
WS_ID=$(az rest --method GET \
  --url "https://api.fabric.microsoft.com/v1/workspaces" \
  --resource "https://api.fabric.microsoft.com" \
  | jq -r --arg n "$WS_NAME" '.value[] | select(.displayName==$n) | .id')

# 3. Resolve lakehouse item ID (source for bindings)
LH_NAME="ZavaAirlinesLH"
LH_ID=$(az rest --method GET \
  --url "https://api.fabric.microsoft.com/v1/workspaces/${WS_ID}/lakehouses" \
  --resource "https://api.fabric.microsoft.com" \
  | jq -r --arg n "$LH_NAME" '.value[] | select(.displayName==$n) | .id')

echo "WS_ID=${WS_ID}  LH_ID=${LH_ID}"
```

### Folder (optional)

To place the new ontology in a workspace folder, resolve the **folder GUID** — not the numeric `subfolderId` shown in portal URLs. The Fabric API expects a GUID for `folderId` on the create payload and rejects the Power BI numeric ID with `400 InvalidParameter … cannot convert "<numeric>" to Guid`.

```bash
FOLDER_NAME="Ontology"
FOLDER_ID=$(az rest --method GET \
  --url "https://api.fabric.microsoft.com/v1/workspaces/${WS_ID}/folders" \
  --resource "https://api.fabric.microsoft.com" \
  | jq -r --arg n "$FOLDER_NAME" '.value[] | select(.displayName==$n) | .id')

echo "FOLDER_ID=${FOLDER_ID}"   # GUID; pass as top-level "folderId" on POST /items
```

### Eventhouse / KQL Database (for TimeSeries bindings)

If the ontology will carry **timeseries bindings against an Eventhouse**, resolve the Eventhouse item ID, its cluster URI, and the target KQL database name — all three are required in the `KustoTable` data-binding payload.

```bash
# Resolve EH_ID + CLUSTER_URI + DB_NAME from a single /kqlDatabases call.
# The KQL database record carries parentEventhouseItemId (the Eventhouse item ID
# the KustoTable binding requires) and properties.queryServiceUri.
DB_NAME="ZavaTelemetryDB"

KQL_DB=$(az rest --method GET \
  --url "https://api.fabric.microsoft.com/v1/workspaces/${WS_ID}/kqlDatabases" \
  --resource "https://api.fabric.microsoft.com" \
  | jq -c --arg n "$DB_NAME" '.value[] | select(.displayName==$n)')

if [ -z "$KQL_DB" ] || [ "$KQL_DB" = "null" ]; then
  echo "ERROR: KQL database '$DB_NAME' not found in workspace $WS_ID" >&2
  exit 1
fi

EH_ID=$(echo "$KQL_DB"       | jq -r '.properties.parentEventhouseItemId')
CLUSTER_URI=$(echo "$KQL_DB" | jq -r '.properties.queryServiceUri')
DB_NAME=$(echo "$KQL_DB"     | jq -r '.displayName')   # canonical casing from API

# Optional: confirm the Eventhouse item itself still exists (guards against drift).
az rest --method GET \
  --url "https://api.fabric.microsoft.com/v1/workspaces/${WS_ID}/eventhouses/${EH_ID}" \
  --resource "https://api.fabric.microsoft.com" > /dev/null

echo "EH_ID=${EH_ID}  CLUSTER_URI=${CLUSTER_URI}  DB_NAME=${DB_NAME}"
```

> Eventhouse tables can back **`TimeSeries` bindings only**. The entity type's static (`NonTimeSeries`) binding must still come from a managed lakehouse table.

> **Eventhouse ID field mapping**: The KQL databases API returns `properties.parentEventhouseItemId` — this is the value you must use as `itemId` in a `KustoTable` data-binding payload. Do not use the KQL database's own `id` field.

See [COMMON-CLI.md § Finding Workspaces and Items in Fabric](../../common/COMMON-CLI.md#finding-workspaces-and-items-in-fabric) for pagination and JMESPath variants.

### Schema Discovery

Before composing bindings, discover the source table schemas so you map the correct column names.

**Lakehouse tables** — use the Fabric Tables REST API:

```bash
# List tables
az rest --method GET \
  --url "https://api.fabric.microsoft.com/v1/workspaces/${WS_ID}/lakehouses/${LH_ID}/tables" \
  --resource "https://api.fabric.microsoft.com"

# For column-level schema, use the SQL endpoint (INFORMATION_SCHEMA) or the
# OneLake Table API (Iceberg metadata). The Tables API returns table names only.
```

**Eventhouse / KQL tables** — query the Kusto REST API:

```bash
TOKEN=$(az account get-access-token --resource "https://kusto.kusto.windows.net" --query accessToken -o tsv)
curl -s -X POST "${CLUSTER_URI}/v1/rest/query" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"db":"'"$DB_NAME"'","csl":".show table MyTable schema as json"}'
# Parse the OrderedColumns array from the response to get column names and types.
```

### LRO Header Capture with `az rest`

`az rest` does not expose response headers by default. Both `createItem` and `updateDefinition` return **202 Accepted** with a `Location` header pointing at the LRO operation URL. Use `--verbose` and parse stderr to capture it:

```bash
# Bash — capture Location header from az rest --verbose stderr
LRO_URL=$(az rest --method POST \
  --url "https://api.fabric.microsoft.com/v1/workspaces/${WS_ID}/items" \
  --resource "https://api.fabric.microsoft.com" \
  --headers "Content-Type=application/json" \
  --body @envelope.json --verbose 2>&1 \
  | grep -oP "(?<='Location': ')[^']+")
```

```powershell
# PowerShell — capture Location header from az rest --verbose stderr
$result = az rest --method POST `
  --resource "https://api.fabric.microsoft.com" `
  --url "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/items" `
  --headers "Content-Type=application/json" `
  --body "@envelope.json" --verbose 2>&1
$lroUrl = ($result | Select-String -Pattern "'Location': '([^']+)'" |
  ForEach-Object { $_.Matches[0].Groups[1].Value })
```

> **Important**: `createItem` returns 202 with **no response body** — `az rest` exits with code 0 and prints nothing. This is normal. Always list items after the LRO completes to capture the new item ID.

---

## Authoring Scope

| Operation | Fabric REST Call | Definition Parts Touched |
|---|---|---|
| Create empty ontology | `POST /v1/workspaces/{ws}/items` with `type=Ontology` | `.platform`, `definition.json` |
| Add / alter entity type | `POST /v1/workspaces/{ws}/items/{id}/updateDefinition` | `EntityTypes/{id}/definition.json` |
| Bind entity type to table (non-timeseries) | `updateDefinition` | `EntityTypes/{id}/DataBindings/{guid}.json` with `NonTimeSeries` |
| Bind entity type to table (timeseries) | `updateDefinition` | `EntityTypes/{id}/DataBindings/{guid}.json` with `TimeSeries` + `timestampColumnName` |
| Add relationship type | `updateDefinition` | `RelationshipTypes/{id}/definition.json` |
| Bind relationship (contextualization) | `updateDefinition` | `RelationshipTypes/{id}/Contextualizations/{guid}.json` |
| Delete entity / relationship | `updateDefinition` with that path omitted from parts | — |
| Rename ontology | `updateDefinition` with `updateMetadata=true` and new `.platform` | `.platform` |

> **Update Item Definition replaces the full tree of included parts.** Always fetch the current definition via `Get Item Definition`, mutate the parts locally, and resend the complete desired set.

---

## Authoring Reference

Full JSON shapes, field contracts, and verification recipes for each operation live in [authoring-mechanics.md](references/authoring-mechanics.md). Worked end-to-end bash recipes live in [examples.md](references/examples.md). Use the sections below as a quick index:

| Topic | Reference |
|---|---|
| Definition envelope (`parts[]`, `InlineBase64`, base64 helpers) | [authoring-mechanics.md § Definition Envelope](references/authoring-mechanics.md#definition-envelope-for-ontology) |
| ID generation (64-bit ints, GUIDs, `name → id` map) | [authoring-mechanics.md § ID Generation Pattern](references/authoring-mechanics.md#id-generation-pattern) |
| Create empty ontology | [authoring-mechanics.md § Create the Ontology Item](references/authoring-mechanics.md#create-the-ontology-item) |
| Add an entity type | [authoring-mechanics.md § Add an Entity Type](references/authoring-mechanics.md#add-an-entity-type) |
| Bind to lakehouse / eventhouse | [authoring-mechanics.md § Bind an Entity Type](references/authoring-mechanics.md#bind-an-entity-type-to-a-lakehouse-or-eventhouse-table) |
| Relationship types + contextualizations | [authoring-mechanics.md § Add a Relationship Type](references/authoring-mechanics.md#add-a-relationship-type) |
| Apply a definition update (fetch → mutate → send) | [authoring-mechanics.md § Apply a Definition Update](references/authoring-mechanics.md#apply-a-definition-update) |
| Verify and inspect | [authoring-mechanics.md § Verify and Inspect](references/authoring-mechanics.md#verify-and-inspect) |
| Complete Bash / PowerShell scaffolds | [definition-script-templates.md](references/definition-script-templates.md) |

**Core invariants to keep in mind when authoring (full detail in the reference files):**

- Envelope shape: `{ "displayName", "type": "Ontology", "definition": { "parts": [ { "path", "payload", "payloadType": "InlineBase64" } ] } }`; `definition.json` is literally `{}`; `.platform` carries `metadata.type: "Ontology"` + `displayName`.
- IDs: entity / relationship / property IDs are **positive 64-bit integers**, data binding / contextualization IDs are **GUIDs**. Persist the `name → id` map in source control; never reuse an ID for a different concept.

**ID map template** — persist this alongside your deployment scripts (JSON or YAML):

```json
{
  "ontologyName": "SkillTest_Fleet",
  "entityTypes": {
    "Site":      { "id": "1048860412765431174", "properties": { "SiteId": "1428056703884423742", "SiteName": "4251708967918658190" } },
    "Equipment": { "id": "3332700945676096991", "properties": { "EquipmentId": "4585483423451989345" } }
  },
  "relationshipTypes": {
    "EquipmentAtSite": { "id": "4242053467032157032" }
  },
  "bindings": {
    "Site_static":      "25e3a44a-b62a-40e3-a64a-a43caaa92d19",
    "Equipment_static": "5dc4cadd-3700-4c96-bb1e-41e4c909ae4d"
  }
}
```
- Bindings: `NonTimeSeries` is **lakehouse-only** and at most one per entity type; a `NonTimeSeries` binding is required **before** any `TimeSeries` binding on the same entity type; `TimeSeries` can be lakehouse or Eventhouse; for `KustoTable`, `itemId` is the **Eventhouse item ID** (not the KQL database ID).
- Relationships: `source.entityTypeId` and `target.entityTypeId` must be distinct and must reference entity types present in the parts tree.
- Updates replace the included parts wholesale — **always** fetch the current definition, mutate locally, then send.

---

## Must / Prefer / Avoid / Troubleshooting

### Must

- **Clarify before acting on ambiguous prompts** — never infer schema or bindings. If the user says "create an ontology for airline data" without naming entity types, their keys, or the lakehouse tables, ask what entities, what keys, and which lakehouse tables. Irreversible side-effects (replacing an ontology definition) require explicit user intent.
- **Resolve `WS_ID` and source item IDs before composing any binding** — hardcoded GUIDs are a top-3 failure mode. Lakehouse bindings need the lakehouse `itemId`; eventhouse bindings need the eventhouse `itemId`, cluster URI, and database name.
- **Fetch the current definition before any update** — `updateDefinition` replaces included parts wholesale. Merging with stale local state silently drops recent changes. Handle the LRO 202 on `getDefinition` (poll and retrieve via the operation's `result` endpoint).
- **Persist the `name → id` map** for entity types, relationship types, and properties in source control alongside the skill consumer's repo. Regenerating IDs on every run creates duplicates and breaks references.
- **Add the static (`NonTimeSeries`) binding before any timeseries binding** on an entity type — each entity type supports at most one static binding, and timeseries binding requires the static key property to already be populated.
- **Bind only to managed lakehouse tables** — external tables, lakehouses with OneLake security enabled, and delta tables with column mapping enabled are not supported.
- **Restrict entity keys (`entityIdParts`) to properties whose `valueType` is `String` or `BigInt`** — other value types cannot be used as keys.
- **Verify permissions** — authoring requires at least `Contributor` on the workspace.
- **Treat the item type as `Ontology`** (not `OntologyPreview` or similar) in both the envelope's `type` and the `.platform` metadata.
- **Render a Preview & Confirm gate before every LRO write** — render a Mermaid proposal (greenfield) or a change-set diff vs. `getDefinition` (brownfield) and obtain explicit `yes` from the user before calling `createItem` or `updateDefinition`. See [preview-and-confirm.md](references/preview-and-confirm.md). Anything other than `yes` means stop and revise; never partially apply.

### Prefer

- **Building the definition tree on disk** (one JSON per logical part, mirroring the `EntityTypes/{id}/...` layout) and base64-encoding each file just before sending. This keeps diffs reviewable.
- **Starting from a `getDefinition` dump** of an existing known-good ontology when onboarding, then mutating.
- **Lakehouse-first static binding; Eventhouse for time-series** — OneLake (lakehouse) is the only supported source for `NonTimeSeries` bindings. Use Eventhouse (`KustoTable`) for high-volume telemetry on `TimeSeries` bindings, or mirror/shortcut the data into a lakehouse table if the team prefers a single source kind.
- **Idempotent deploy scripts** — re-running the script with unchanged inputs should produce an unchanged ontology.
- **Scripted workflow over UI** when more than one entity type or environment is involved.
- **Triggering a manual graph-model refresh** after upstream data writes — new rows in bound sources are not visible in the preview experience until the ontology is refreshed.

### Avoid

- **Hand-editing base64 payloads** — always decode, edit the JSON, then re-encode.
- **Reusing a property/entity/relationship ID** for a different concept.
- **Relying on relationship-name uniqueness outside the ontology scope** — today, relationship names appear to be unique within an ontology (observed behavior); collect the full set of desired relationship names up front so you can disambiguate with prefixes if needed. Confirm naming collisions with the consumer rather than guessing.
- **Embedding secrets, SAS tokens, or user tokens** in `.platform` or any part.
- **Creating relationship types before their source and target entity types exist in the parts list**.
- **Treating `Object` / JSON properties as fully queryable** — observed behavior today is that nested JSON bound to an `Object` property surfaces as an opaque payload rather than being addressable like a scalar. For nested payloads, keep the raw data in Eventhouse and bind only the addressable scalar fields. Verify with a `getDefinition` round-trip before promising downstream consumers a specific query shape.
- **Relying on "delete by omission"** — parts not included in the `updateDefinition` body are observed to be removed, but the skill should tell the user this is destructive and confirm before generating an envelope that drops parts.

### Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `400 Bad Request` on create — "Invalid item type" | Wrong item type string | Use `"type": "Ontology"` and `metadata.type: Ontology` in `.platform` |
| `400 InvalidParameter` on create — `Error converting value "<number>" to … Guid … Path 'folderId'` | Passed the numeric `subfolderId` from a portal URL instead of the folder GUID | Resolve the folder GUID via `GET /v1/workspaces/{WS_ID}/folders` (see Connection § Folder) and pass that |
| `400` — "Invalid value type" on property | Using `Int64` / `Guid` / `Float` as `valueType` | Allowed values are exactly `String`, `Boolean`, `DateTime`, `Object`, `BigInt`, `Double` |
| `400` — "Invalid identifier" on entity type / property | Name violates regex | Match `^[a-zA-Z][a-zA-Z0-9_-]{0,127}$`; prefer the stricter 1–26 char portal rule to stay portable |
| `400` — "Source and target must differ" | Relationship points at the same entity type twice | Choose distinct source/target entity types |
| `400` — "Referenced property not found" | `targetPropertyId` doesn't match any property in the entity type | Check IDs; ensure property was added in the same update |
| `400` — "Time series binding requires existing static binding" | Timeseries binding added before the static binding on an entity type | Add a `NonTimeSeries` binding with the key property first, then the `TimeSeries` binding |
| `400` — key column issue | Key property `valueType` is not `String` or `BigInt` | Change the property to `String` / `BigInt`, or choose a different key |
| `404` on binding | `workspaceId` / `itemId` wrong, or source item deleted | Re-resolve IDs via `list items` / `list lakehouses` / `list eventhouses` |
| Binding accepted but no instances appear | Source is external table, column-mapped delta, or OneLake-secured | Rebuild the table as a managed delta table without column mapping; remove OneLake security on the lakehouse |
| Instances empty after binding | `propertyBindings` column names don't match source columns | Inspect the source schema and fix `sourceColumnName` / `sourceSchema` |
| New upstream rows not appearing | No refresh performed | Trigger a manual graph-model refresh on the ontology item |
| Timeseries widget shows no data | `timestampColumnName` not set, or timestamp column is not a supported date/time type | Set `timestampColumnName` in the TimeSeries binding; ensure column type is `datetime` / `date` / `timestamp` |
| `getDefinition` returns `202` | LRO response, not the envelope | Poll the operation until `Succeeded`, then `GET {operation-location}/result` — see [LRO Header Capture](#lro-header-capture-with-az-rest) |
| `Conflict` on `updateDefinition` | Concurrent edit from the portal | Re-fetch definition, re-apply mutations, resend |
| `ALMOperationImportFailed` on `updateDefinition` | Malformed JSON payload — often caused by PowerShell `ConvertTo-Json` serialization quirks (`$null` vs `null`, key reordering, BOM in file) | Build JSON with `jq -nc` instead of `ConvertTo-Json`; write files with `[System.IO.File]::WriteAllText` + `UTF8Encoding($false)` to avoid BOM; validate with `jq .` before sending — see [Tool Stack § PowerShell Warning](#tool-stack) |
| `createItem` returns exit code 0 but no output | Normal — `createItem` returns `202 Accepted` with no body; `az rest` treats this as success | List items after the LRO completes to capture the new item ID; use `--verbose` to capture the `Location` header for LRO polling |
| `409 ItemDisplayNameAlreadyInUse` on `createItem` | Ontology with the same `displayName` already exists in the workspace | List existing ontologies first; delete or rename the existing one, or choose a different name |
| `definition.json` payload causes import error | Extra whitespace, BOM, or newlines in the base64 payload | `definition.json` must be exactly `{}` — its base64 is `e30=`. On Windows, ensure no BOM by using `[System.IO.File]::WriteAllText` with `UTF8Encoding($false)` |

---

## Agentic Workflows

### Exploration Before Authoring

```text
Step 0 → Is the request specific? Are the entity types, keys, and lakehouse tables named?
         → NO  → Ask: "Which entity types? What is the key of each? Which lakehouse table binds to each?
                  Any timeseries properties? Any relationships and their link tables?"
                  STOP — do not proceed until the user answers.
         → YES → Continue.
Step 1 → List workspaces → resolve WS_ID                               [COMMON-CLI.md]
Step 2 → List lakehouses in WS → resolve LH_ID and source schemas
Step 3 → If ontology exists: getDefinition → decode parts              (capture current IDs)
         Else: plan `createItem` with `.platform` + empty `definition.json`.
Step 4 → For each entity type:
           a. Generate/reuse 64-bit IDs for entity + properties
           b. Build EntityTypes/{id}/definition.json
           c. Build one or two DataBindings/{guid}.json files
Step 5 → For each relationship:
           a. Confirm both entity types exist in Step 4 output
           b. Generate/reuse relationship type ID
           c. Build RelationshipTypes/{id}/definition.json
           d. Build RelationshipTypes/{id}/Contextualizations/{guid}.json
Step 6 → Base64-encode all parts; assemble envelope
Step 7 → **PREVIEW & CONFIRM** — render proposal (greenfield) or change-set diff (brownfield)
         and obtain explicit `yes` from the user. See [preview-and-confirm.md](references/preview-and-confirm.md).
         Do not proceed on anything other than `yes`.
Step 8 → createItem OR updateDefinition (LRO)
Step 9 → Poll LRO until Succeeded; getDefinition; verify IDs and bindings; persist post-write snapshot for next-run diff
```

### Script Generation Workflow

```text
Step 1 → Capture user intent (entity types, keys, properties, relationships, source tables)
Step 2 → Save intent as a YAML/JSON spec in the consumer's repo — single source of truth
Step 3 → Generate: (a) the ID map, (b) per-file JSON parts, (c) the composite envelope
Step 4 → **PREVIEW & CONFIRM** — render proposal/diff and require explicit `yes`
         (see [preview-and-confirm.md](references/preview-and-confirm.md)). The textual
         diff against the last-applied envelope snapshot feeds the brownfield change-set.
Step 5 → Apply via az rest --body @envelope.json (createItem or updateDefinition)
Step 6 → Poll LRO; on success, commit the envelope snapshot + ID map
```

---

## Examples

End-to-end worked examples (create empty ontology → add entity type + non-timeseries binding → add relationship type + contextualization → add timeseries property + Eventhouse binding) live in [examples.md](references/examples.md). Complete fetch-mutate-send bash and PowerShell scripts live in [definition-script-templates.md](references/definition-script-templates.md).


---

## Agent Integration Notes

- This skill is authoring-focused. Pair with a consumption skill (e.g., a Fabric Graph query skill) to validate the ontology end-to-end.
- When orchestrating multi-step customer workstreams that span Ontology + Eventhouse + Lakehouse, route via an agent (e.g., `FabricDataEngineer`) rather than chaining skills directly.
- Reasonable upstream dependencies to assume: lakehouse tables already exist and have the key columns the user described. If not, the caller should invoke a lakehouse authoring skill first.
