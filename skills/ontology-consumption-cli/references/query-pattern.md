**# Microsoft Fabric Data Access Patterns**

**## Authentication**

- Use `azure-identity` → `DefaultAzureCredential()` (chains through AzureCliCredential on local dev)

- Token scopes:

  - ****SQL (Lakehouse/Warehouse)****: `[https://database.windows.net/.default`](https://database.windows.net/.default` "https://database.windows.net/.default%60")

  - ****KQL (Eventhouse)****: `[https://help.kusto.windows.net/.default`](https://help.kusto.windows.net/.default` "https://help.kusto.windows.net/.default%60") (NOT `kusto.fabric.microsoft.com`)

  - ****Fabric REST API****: `[https://api.fabric.microsoft.com/.default`](https://api.fabric.microsoft.com/.default` "https://api.fabric.microsoft.com/.default%60")

**## Lakehouse SQL Endpoint (via pyodbc)**

- Driver: `ODBC Driver 18 for SQL Server`

- Token auth: pack token as UTF-16LE struct → `attrs_before={1256: token_struct}`

  ```python

  raw = credential.get_token(SQL_SCOPE).token.encode("utf-16-le")

  token_struct = struct.pack(f"<I{len(raw)}s", len(raw), raw)

  conn = pyodbc.connect(conn_str, attrs_before={1256: token_struct})

  ```

- Connection string: `Driver={ODBC Driver 18 for SQL Server};Server=<sql-endpoint-host>;Database=<lakehouse-name>;Encrypt=Yes;TrustServerCertificate=No`

- ****READ-ONLY****: No INSERT, UPDATE, DELETE, ALTER TABLE, CREATE TABLE

- Use `SELECT TOP N` for pagination; `ORDER BY` works

**## Eventhouse / KQL Database (via REST)**

- ****Query endpoint****: `POST {kusto_uri}/v2/rest/query` with `{"db": "...", "csl": "..."}`

  - Response: array of frames; find `FrameType == "DataTable"` and `TableKind == "PrimaryResult"`

- ****Management endpoint****: `POST {kusto_uri}/v1/rest/mgmt` with `{"db": "...", "csl": "..."}`

- Kusto URI discovery: `GET [https://api.fabric.microsoft.com/v1/workspaces/{ws_id}/kqlDatabases/{db_id}`](https://api.fabric.microsoft.com/v1/workspaces/%7Bws_id%7D/kqlDatabases/%7Bdb_id%7D` "https://api.fabric.microsoft.com/v1/workspaces/%7bws_id%7d/kqldatabases/%7bdb_id%7d%60") → `properties.queryServiceUri`

**### KQL Command Support in Fabric Eventhouse**

- ****WORKS****: `.set-or-replace TableName <| query` — replaces table contents with query results

- ****WORKS****: `.show table T extents`, `.show table T ingestion mappings`, `.drop table T ifexists`

- ****DOES NOT WORK****: `.delete table T records <| ...` → returns 400 BadRequest

- ****DOES NOT WORK****: `.purge table T records <| ...` → returns 403 Forbidden

- ****DOES NOT WORK****: `.undo drop extents` → returns 400

- To "delete" specific records: use `.set-or-replace` with a filtered query that excludes unwanted rows

**## Fabric REST API**

- List workspace items: `GET [https://api.fabric.microsoft.com/v1/workspaces/{ws_id}/items`](https://api.fabric.microsoft.com/v1/workspaces/%7Bws_id%7D/items` "https://api.fabric.microsoft.com/v1/workspaces/%7bws_id%7d/items%60")

- Item types include: Lakehouse, Eventhouse, KQLDatabase, Notebook, SQLEndpoint, SemanticModel, etc.

**## Ontology (LogisticsMD-style)**

- Ontology defines entities (Vehicle, Driver, Route, Warehouse, Shipment) with properties

- Lakehouse tables store entity data; Eventhouse or Lakehouses can store time-series telemetry (VehicleLocation, VehicleFuelLevel)

- Lakehouse tables often lack FK columns — relationships defined in ontology must be computed/enriched at query time

- Deterministic FK assignment: use `hashlib.sha256(seed).hexdigest()` for stable entity-to-entity mapping