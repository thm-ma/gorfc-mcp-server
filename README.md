# gorfc-mcp-server

A [Model Context Protocol](https://modelcontextprotocol.io/) (MCP) server that connects LLMs to SAP systems via RFC (Remote Function Call). It exposes SAP RFC operations as MCP tools over stdio transport, allowing AI agents to ping SAP systems, describe function modules, invoke BAPIs/RFCs, and retrieve call metrics.

## Prerequisites

- **Go 1.24+**
- **SAP NW RFC SDK** installed at `/usr/local/sap/nwrfcsdk/` (headers in `include/`, libraries in `lib/`)
- A valid `sapnwrfc.ini` configuration file in the working directory with SAP connection destinations

### SAP NetWeaver RFC SDK Proprietary Notice
The **SAP NetWeaver RFC SDK** is proprietary software owned by SAP SE. It is **not open-source** and is subject to specific licensing terms.

#### Licensing & Access
* **No Redistribution:** You are not permitted to redistribute the SDK binaries (`.dll`, `.so`, `.dylib`) in your own applications or repositories.
* **Customer Access:** Access requires a valid SAP license and an **S-User ID** for the SAP Support Portal.
* **Legal Source:** Binaries must only be downloaded from the official SAP Software Download Center.

#### Official Resources
* **Product Page:** [SAP Support Portal - Connectors](https://support.sap.com/en/product/connectors/nwrfcsdk.html)
* **Master Note:** [SAP Note 2573790](https://launchpad.support.sap.com/#/notes/2573790) (Installation and Availability for version 7.50).
* **Technical Guide:** Detailed programming guides are provided within the `doc` folder of the downloaded SDK package.

## Build

```bash
make build           # compiles to ./gorfc-mcp-server
make build-windows   # cross-compiles to ./gorfc-mcp-server.exe (Windows x86_64)
make clean           # removes built binaries
```

The build uses CGO with flags pointing to the SAP NW RFC SDK:

```
CGO_CFLAGS="-I/usr/local/sap/nwrfcsdk/include"
CGO_LDFLAGS="-L/usr/local/sap/nwrfcsdk/lib"
```

### Cross-compiling for Windows

The `build-windows` target cross-compiles for Windows x86_64 using MinGW.
It requires a Windows build of the SAP NW RFC SDK with headers and libraries available at the configured SDK path and the MinGW-w64 cross-compiler (`x86_64-w64-mingw32-gcc`).

Install on Debian/Ubuntu:
  ```bash
  sudo apt install gcc-mingw-w64-x86-64
  ```

## Configuration

The server reads SAP connection destinations from a `sapnwrfc.ini` file (standard SAP NW RFC SDK config format) in the working directory. Example:

```ini
DEST=SID
TYPE=A
ASHOST=sap.example.com
SYSNR=00
CLIENT=100
USER=rfcuser
PASSWD=secret
LANG=EN
```

> **Note:** `sapnwrfc.ini` contains plaintext credentials and is gitignored. Never commit this file.

## Running

Pass the SAP destination name via the `SAP_DEST` environment variable or as the first CLI argument:

```bash
SAP_DEST=SID ./gorfc-mcp-server
```

```bash
./gorfc-mcp-server SID
```
The SID or System Id  is the unique, three-character alphanumeric identifier that serves as the "name" for a specific SAP system installation as described in your `sapnwrfc.ini` configuration file. For example:

| SID | Description |
| :--- | :--- |
| `DEV` | Development and configuration system |
| `QAS` | Quality Assurance and testing system |
| `PRD` | Production (live) environment |


Logs are written to stderr with a `[gorfc-mcp]` prefix.

## MCP Client Configuration

### Claude Desktop

Add to your Claude Desktop config (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "sap": {
      "command": "/path/to/gorfc-mcp-server",
      "args": ["SID"],
      "env": {
        "LD_LIBRARY_PATH": "/usr/local/sap/nwrfcsdk/lib"
      }
    }
  }
}
```

### Claude Code

Add via the CLI:

```bash
claude mcp add sap /path/to/gorfc-mcp-server -- SID
```

## Tools
Model Context Protocol (MCP) tools designed for SAP connectivity, metadata inspection, and table discovery via the NetWeaver RFC SDK.

---

## Tool Overview

| Tool | Description |
| :--- | :--- |
| `rfc_ping` | Verify SAP connectivity. |
| `rfc_connection_info` | Get connection attributes and NW RFC SDK version. |
| `rfc_describe` | Get function module metadata (parameters, types, directions). |
| `rfc_call` | Invoke an RFC function module with parameters. |
| `get_table_metadata` | Retrieve field details (types, length, domain) for a table. |
| `get_table_relations` | Retrieve foreign-key relationships and cardinalities. |
| `search_sap_tables` | Search for tables by description/business term. |
| `metrics_get` | Return call statistics and performance metrics. |

---

## SAP Connectivity Tools

### rfc_ping
Pings the connected SAP system to verify connectivity. 
* **Parameters:** None.

### rfc_connection_info
Returns connection attributes such as System ID, Client, Host, and the SAP NW RFC SDK version. 
* **Parameters:** None.

### rfc_describe
Returns the full metadata of an RFC function module. Use this to understand the interface before calling.

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `function_name` | string | **Yes** | Name of the RFC (e.g. `STFC_CONNECTION`) |

### rfc_call
Invokes an RFC function module with the given parameters and returns the result. 

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `function_name` | string | **Yes** | Name of the RFC function module to call |
| `parameters` | object | No | Input parameters for the function call |

#### Parameter Value Type Mapping

| ABAP Type | JSON Value | Format / Note |
| :--- | :--- | :--- |
| `INT`, `INT1`, `INT2`, `INT8` | number | Standard integer |
| `FLOAT`, `BCD`, `DECF` | number | Floating point or decimal |
| `CHAR`, `STRING`, `NUM` | string | Textual data |
| `DATE` | string | YYYYMMDD |
| `TIME` | string | HHMMSS |
| `BYTE`, `XSTRING` | string | base64-encoded |
| `STRUCTURE` | object | Key-value pairs |
| `TABLE` | array | Array of objects |

---

## Data Dictionary (DDIC) Tools

### get_table_metadata
**SAP Function module:** `DDIF_FIELDINFO_GET`  
Returns the fields of a given SAP table including field name, ABAP type, length, domain, and short description.

| Parameter | Type | Required | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| `table_name` | string | **Yes** | - | The SAP table name (e.g. `SFLIGHT`) |
| `language` | string | No | `D` | Language for text/descriptions |

### get_table_relations
**SAP Function module:** `FAPI_GET_FOREIGN_KEY_RELATIONS`  
Retrieves foreign-key relationships for the specified table, including referenced check tables and cardinalities.

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `table_name` | string | **Yes** | The SAP table name |

### search_sap_tables
**SAP Function module:** `RFC_READ_TABLE` (targeting `DD02T`)  
Performs a LIKE-style search over table short texts to find tables by description.

| Parameter | Type | Required | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| `search_term` | string | **Yes** | - | The text to search for (LIKE semantics) |
| `language` | string | No | `D` | Language for search |
| `max_results` | integer | No | `100` | Maximum results to return |

---

## Additional Helper Functions

* **`sanitizeABAPString`**: Escapes single-quotes and other characters for safely embedding user strings into ABAP `WHERE` clauses or `RFC_READ_TABLE` filters.
* **`parseReadTableResult`**: Parses the legacy `DATA`/`WA` row format from `RFC_READ_TABLE` into a structured array of maps (`[]map[string]string`) keyed by column name.

---

## Monitoring

### metrics_get
Returns in-memory call statistics: total/successful/failed call counts, total and average duration, and per-function call counts.
* **Parameters:** None.

## Architecture

All logic lives in a single file: `cmd/gorfc-mcp-server/main.go`.

- **connManager** -- Thread-safe wrapper around `gorfc.Connection`. All RFC calls are serialized through a mutex since the SAP NW RFC SDK is not thread-safe per connection handle. Includes auto-reconnect with exponential backoff (3 retries, starting at 100ms).
- **coerceParams / coerceValue** -- Type coercion layer that converts JSON-deserialized Go types (`float64`, `string`, etc.) to the specific Go types `gorfc` expects. Recursively handles structures and tables.
- **validateParameters** -- Pre-call validation that all parameter names exist in the function description.
- **metrics** -- In-memory call counter tracking total/success/failure counts, durations, and per-function stats.

## Example Prompts

See [EXAMPLES.md](EXAMPLES.md) for example prompts to use with this server.

## License

[Apache License 2.0](LICENSE)
