# Ontology Definition — Script Templates

Reference scaffolds for authoring Fabric Ontology items end-to-end from the CLI. Keep these snippets close to your deployment scripts; they are intentionally minimal and meant to be adapted, not executed verbatim.

## Platform assumptions

- Bash template — Linux / WSL / macOS with GNU coreutils. On BSD `base64` (stock macOS), swap `base64 -d` → `base64 -D` and replace `base64 -w 0` with `base64 | tr -d '\n'`. Requires `curl`, `jq`, and `az`.
- PowerShell template — **PowerShell 7+** (pwsh). Relies on `-SkipHttpErrorCheck` and `utf8NoBOM` which are not available in Windows PowerShell 5.1. On 5.1, use the Bash template via WSL or upgrade to PowerShell 7.
- Python section (§ 4) is a **generator sketch** — it composes an envelope from an in-memory spec. It is **not** a full fetch-mutate-send client. For that flow, use the Bash or PowerShell template above.

---

## 1. Bash: Fetch → Decode → Mutate → Re-encode → Send

`getDefinition` is long-running-operation-capable: it may return **200 OK** with the envelope inline, or **202 Accepted** with a `Location` header pointing at the operation (poll until `Succeeded`, then `GET {Location}/result` to retrieve the envelope). This template handles both.

```bash
#!/usr/bin/env bash
set -euo pipefail

: "${WS_ID:?set WS_ID}"; : "${ONTO_ID:?set ONTO_ID}"
WORK=$(mktemp -d)
FABRIC_BASE="https://api.fabric.microsoft.com"
TOKEN=$(az account get-access-token --resource "$FABRIC_BASE" --query accessToken -o tsv)

# 1. Fetch current definition (handle 200 or 202 LRO)
getdef_http=$(curl -sS -D "$WORK/getdef.headers" -o "$WORK/getdef.body" -w "%{http_code}" \
  -X POST "$FABRIC_BASE/v1/workspaces/${WS_ID}/items/${ONTO_ID}/getDefinition" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Length: 0")

if [ "$getdef_http" = "200" ]; then
  cp "$WORK/getdef.body" "$WORK/current.json"
elif [ "$getdef_http" = "202" ]; then
  OP_URL=$(awk 'tolower($1)=="location:" {print $2}' "$WORK/getdef.headers" | tr -d '\r')
  # Poll operation until terminal
  while :; do
    OP_STATUS=$(curl -sS -H "Authorization: Bearer $TOKEN" "$OP_URL" | jq -r .status)
    case "$OP_STATUS" in
      Succeeded) break ;;
      Failed|Cancelled) echo "getDefinition LRO $OP_STATUS" >&2; exit 1 ;;
      *) sleep 2 ;;
    esac
  done
  curl -sS -H "Authorization: Bearer $TOKEN" "${OP_URL}/result" -o "$WORK/current.json"
else
  echo "getDefinition returned $getdef_http" >&2
  cat "$WORK/getdef.body" >&2
  exit 1
fi

# 2. Explode parts to disk (preserve folder layout)
#    Note: base64 -d is GNU/Linux; on macOS (BSD base64) use -D.
jq -r '.definition.parts[] | "\(.path)\t\(.payload)"' "$WORK/current.json" \
  | while IFS=$'\t' read -r path payload; do
      mkdir -p "$(dirname "$WORK/tree/$path")"
      printf '%s' "$payload" | base64 -d > "$WORK/tree/$path"
    done

# 3. Mutate — drop new/updated JSON files into $WORK/tree/...
#    e.g. add a new entity type
#    cp my-new-tank.json "$WORK/tree/EntityTypes/8813598896083/definition.json"

# 4. Rebuild envelope from $WORK/tree/
#    Note: base64 -w 0 is GNU-only; on macOS use `base64 | tr -d '\n'`.
PARTS_JSON=$(cd "$WORK/tree" && find . -type f | sed 's|^\./||' | while read -r p; do
    jq -nc --arg path "$p" --arg payload "$(base64 -w 0 < "$p")" \
      '{path:$path,payload:$payload,payloadType:"InlineBase64"}'
  done | jq -s .)

jq -n --argjson parts "$PARTS_JSON" '{definition:{parts:$parts}}' > "$WORK/envelope.json"

# 5. Send update (also an LRO — poll Location until Succeeded)
upd_http=$(curl -sS -D "$WORK/upd.headers" -o "$WORK/upd.body" -w "%{http_code}" \
  -X POST "$FABRIC_BASE/v1/workspaces/${WS_ID}/items/${ONTO_ID}/updateDefinition" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  --data-binary @"$WORK/envelope.json")

if [ "$upd_http" = "200" ]; then
  echo "updateDefinition: completed synchronously"
elif [ "$upd_http" = "202" ]; then
  OP_URL=$(awk 'tolower($1)=="location:" {print $2}' "$WORK/upd.headers" | tr -d '\r')
  while :; do
    OP_STATUS=$(curl -sS -H "Authorization: Bearer $TOKEN" "$OP_URL" | jq -r .status)
    case "$OP_STATUS" in
      Succeeded) echo "updateDefinition: Succeeded"; break ;;
      Failed|Cancelled) echo "updateDefinition LRO $OP_STATUS" >&2; exit 1 ;;
      *) sleep 2 ;;
    esac
  done
else
  echo "updateDefinition returned $upd_http" >&2
  cat "$WORK/upd.body" >&2
  exit 1
fi
```

> Both `getDefinition` and `updateDefinition` are LRO-capable; the template above polls the `Location` header for `updateDefinition`. See COMMON-CLI.md § Long-Running Operations (LRO) Pattern.

---

## 2. PowerShell: Same Flow (PowerShell 7+ required)

Uses `[Convert]::ToBase64String` / `FromBase64String` for base64 — do **not** use `certutil -encode`, which line-wraps output and adds header/footer that would corrupt the `InlineBase64` payload. `-SkipHttpErrorCheck` and `utf8NoBOM` are PowerShell 7 features; on Windows PowerShell 5.1, run this via WSL (Bash template) or install `pwsh`.

```powershell
$ErrorActionPreference = "Stop"
if (-not $env:WS_ID)   { throw "set WS_ID" }
if (-not $env:ONTO_ID) { throw "set ONTO_ID" }

$work = New-Item -ItemType Directory -Path (Join-Path $env:TEMP ("onto-" + [guid]::NewGuid()))
$base = "https://api.fabric.microsoft.com"
$token = az account get-access-token --resource $base --query accessToken -o tsv
$headers = @{ Authorization = "Bearer $token" }

# 1. Fetch (handle 200 or 202 LRO)
$resp = Invoke-WebRequest -Method POST `
  -Uri "$base/v1/workspaces/$($env:WS_ID)/items/$($env:ONTO_ID)/getDefinition" `
  -Headers $headers -ContentType "application/json" -Body "{}" -SkipHttpErrorCheck

if ($resp.StatusCode -eq 200) {
  $current = $resp.Content | ConvertFrom-Json
} elseif ($resp.StatusCode -eq 202) {
  $op = $resp.Headers.Location
  do {
    Start-Sleep -Seconds 2
    $status = (Invoke-RestMethod -Method GET -Uri $op -Headers $headers).status
  } while ($status -notin 'Succeeded','Failed','Cancelled')
  if ($status -ne 'Succeeded') { throw "getDefinition LRO $status" }
  $current = Invoke-RestMethod -Method GET -Uri "$op/result" -Headers $headers
} else {
  throw "getDefinition returned $($resp.StatusCode): $($resp.Content)"
}

# 2. Explode parts
foreach ($p in $current.definition.parts) {
  $out = Join-Path $work.FullName $p.path
  New-Item -ItemType Directory -Path (Split-Path $out) -Force | Out-Null
  [IO.File]::WriteAllBytes($out, [Convert]::FromBase64String($p.payload))
}

# 3. Mutate — overwrite / add JSON files under $work.FullName as needed

# 4. Rebuild envelope
$parts = Get-ChildItem -Recurse -File $work.FullName | ForEach-Object {
  $rel = $_.FullName.Substring($work.FullName.Length + 1).Replace('\','/')
  [pscustomobject]@{
    path        = $rel
    payload     = [Convert]::ToBase64String([IO.File]::ReadAllBytes($_.FullName))
    payloadType = "InlineBase64"
  }
}
$envelope = @{ definition = @{ parts = $parts } } | ConvertTo-Json -Depth 10 -Compress
$envPath  = Join-Path $work.FullName "envelope.json"
$envelope | Out-File -Encoding utf8NoBOM $envPath

# 5. Send update (LRO — poll Location until Succeeded)
$updResp = Invoke-WebRequest -Method POST `
  -Uri "$base/v1/workspaces/$($env:WS_ID)/items/$($env:ONTO_ID)/updateDefinition" `
  -Headers $headers -ContentType "application/json" -InFile $envPath -SkipHttpErrorCheck

if ($updResp.StatusCode -eq 200) {
  Write-Host "updateDefinition: completed synchronously"
} elseif ($updResp.StatusCode -eq 202) {
  $op = $updResp.Headers.Location
  do {
    Start-Sleep -Seconds 2
    $status = (Invoke-RestMethod -Method GET -Uri $op -Headers $headers).status
  } while ($status -notin 'Succeeded','Failed','Cancelled')
  if ($status -ne 'Succeeded') { throw "updateDefinition LRO $status" }
  Write-Host "updateDefinition: Succeeded"
} else {
  throw "updateDefinition returned $($updResp.StatusCode): $($updResp.Content)"
}
```

---

## 3. Minimal Part Scaffolds

### `.platform`

```json
{ "metadata": { "type": "Ontology", "displayName": "<ontology display name>" } }
```

### `definition.json`

```json
{}
```

### `EntityTypes/{id}/definition.json`

```json
{
  "id": "<entity type id>",
  "namespace": "usertypes",
  "namespaceType": "Custom",
  "name": "<CamelCaseName>",
  "baseEntityTypeId": null,
  "visibility": "Visible",
  "entityIdParts": [ "<key property id>" ],
  "displayNamePropertyId": "<key property id>",
  "properties": [
    { "id": "<key property id>", "name": "<KeyName>", "redefines": null, "baseTypeNamespaceType": null, "valueType": "String" }
  ],
  "timeseriesProperties": []
}
```

### `EntityTypes/{id}/DataBindings/{guid}.json` (NonTimeSeries)

```json
{
  "id": "<guid>",
  "dataBindingConfiguration": {
    "dataBindingType": "NonTimeSeries",
    "propertyBindings": [
      { "sourceColumnName": "<column>", "targetPropertyId": "<property id>" }
    ],
    "sourceTableProperties": {
      "sourceType": "LakehouseTable",
      "workspaceId": "<WS_ID>",
      "itemId": "<LH_ID>",
      "sourceTableName": "<table>",
      "sourceSchema": "dbo"
    }
  }
}
```

### `EntityTypes/{id}/DataBindings/{guid}.json` (TimeSeries — Eventhouse)

```json
{
  "id": "<guid>",
  "dataBindingConfiguration": {
    "dataBindingType": "TimeSeries",
    "timestampColumnName": "<timestamp column>",
    "propertyBindings": [
      { "sourceColumnName": "<timestamp column>", "targetPropertyId": "<timestamp property id>" },
      { "sourceColumnName": "<column>",           "targetPropertyId": "<property id>" }
    ],
    "sourceTableProperties": {
      "sourceType": "KustoTable",
      "workspaceId": "<WS_ID>",
      "itemId": "<EH_ID>",
      "clusterUri": "<eventhouse cluster uri>",
      "databaseName": "<kql database name>",
      "sourceTableName": "<table>"
    }
  }
}
```

### `RelationshipTypes/{id}/definition.json`

```json
{
  "id": "<relationship id>",
  "namespace": "usertypes",
  "namespaceType": "Custom",
  "name": "<camelCaseName>",
  "source": { "entityTypeId": "<source entity type id>" },
  "target": { "entityTypeId": "<target entity type id>" }
}
```

### `RelationshipTypes/{id}/Contextualizations/{guid}.json`

```json
{
  "id": "<guid>",
  "dataBindingTable": {
    "sourceType": "LakehouseTable",
    "workspaceId": "<WS_ID>",
    "itemId": "<LH_ID>",
    "sourceTableName": "<link table>",
    "sourceSchema": "dbo"
  },
  "sourceKeyRefBindings": [
    { "sourceColumnName": "<src key column>", "targetPropertyId": "<source key property id>" }
  ],
  "targetKeyRefBindings": [
    { "sourceColumnName": "<tgt key column>", "targetPropertyId": "<target key property id>" }
  ]
}
```

---

## 4. Intent → Envelope Generator (Python sketch)

> **Scope:** this is a **generator sketch only** — given an in-memory spec, it emits the `parts[]` array ready to be POSTed. It does **not** fetch the current definition, handle the `getDefinition` / `updateDefinition` LROs, or send the envelope. For a full fetch-mutate-send client, use the Bash (§ 1) or PowerShell (§ 2) template above.

```python
# Given a simple YAML/JSON spec of entity types, properties, relationships, and bindings,
# emit the parts[] array ready for Create/Update Item Definition.
# Persist id_map.json alongside the spec so entity/property IDs remain stable across runs.

import base64, json, uuid

def b64(s: str) -> str:
    return base64.b64encode(s.encode("utf-8")).decode("ascii")

def part(path: str, obj) -> dict:
    return {"path": path, "payload": b64(json.dumps(obj)), "payloadType": "InlineBase64"}

def envelope(parts, display_name):
    return {
        "displayName": display_name,
        "type": "Ontology",
        "definition": {"parts": [part(".platform",
                        {"metadata": {"type": "Ontology", "displayName": display_name}})] +
                       [part("definition.json", {})] + parts}
    }
```

Keep the generator deterministic: every entity/property/relationship gets an ID from `id_map.json`; new concepts append a freshly generated positive 64-bit integer, existing concepts reuse the stored ID. Prefer `secrets.randbits(62)` for the 64-bit integer IDs — avoid shell `RANDOM`, which is 15-bit and collides quickly. Data binding and contextualization IDs can be regenerated with `uuid.uuid4()`.

```python
import secrets, uuid
entity_or_property_id = str(secrets.randbits(62))  # positive 64-bit int
binding_guid = str(uuid.uuid4())
```
