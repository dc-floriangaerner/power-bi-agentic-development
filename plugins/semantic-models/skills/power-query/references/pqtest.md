# PQTest: Local Power Query Execution (Windows Only)

PQTest.exe is a CLI test harness from Microsoft that embeds the full Power Query mashup engine. It can execute arbitrary M expressions locally without any cloud service.

**Platform:** Windows only (.NET Framework 4.7.2)
**Source:** `Microsoft.PowerQuery.SdkTools` NuGet package (verified Microsoft publisher)
**Intended use:** Custom connector development and testing
**Actual use:** Any M expression evaluation

## Architecture

PQTest is a thin CLI host that orchestrates the real mashup engine:

```
PQTest.exe (CLI host)
  |
  | spawns + communicates via named pipes
  v
Microsoft.Mashup.Container.NetFX45.exe (worker process)
  |
  | loads
  v
Microsoft.MashupEngine.dll         (~15MB, the M evaluator)
Microsoft.MashupEngine.Library45.dll (~12MB, standard library)
Microsoft.Mashup.ScriptDom.dll     (~2.4MB, M parser)
Microsoft.Mashup.Document*.dll     (mashup document handling)
+ ~100 more Microsoft.Mashup.*.dll files
+ connector-specific DLLs (SQL, ODBC, Web, etc.)
```

The same DLLs ship with Power BI Desktop. PBI Desktop loads them in-process (no separate mashup container process visible at runtime; the container is only spawned for out-of-process evaluation scenarios like PQTest).

## Installation

```powershell
# Download the NuGet package
nuget install Microsoft.PowerQuery.SdkTools -OutputDirectory C:\Tools\PQTest

# Or via dotnet
dotnet tool restore  # if using a tool-manifest

# The binary is at:
# C:\Tools\PQTest\Microsoft.PowerQuery.SdkTools.<version>\tools\PQTest.exe
```

Or install the Power Query SDK VS Code extension, which bundles PQTest.

## Usage

### Execute an Inline M Expression

Create a `.pq` file with M code:

```powerquery
// test.query.pq
let
    Source = #table({"Product", "Revenue"}, {{"Widget", 42000}, {"Gadget", 18500}}),
    Filtered = Table.SelectRows(Source, each [Revenue] > 20000)
in
    Filtered
```

Run it:

```powershell
PQTest.exe run-test -q test.query.pq -p
```

The `-p` flag pretty-prints the JSON output.

### Execute with Data Source Credentials

For M expressions that access data sources, set credentials first:

```powershell
# Set SQL Server credentials
PQTest.exe set-credential -q query.pq -ak UsernamePassword

# Set anonymous credentials (for Web.Contents, etc.)
PQTest.exe set-credential -q query.pq -ak Anonymous

# Then run
PQTest.exe run-test -q query.pq -p
```

### Key Commands

| Command | Description |
|---------|-------------|
| `run-test` | Execute M and return results as JSON |
| `compare` | Execute and compare against a `.pqout` baseline (regression testing) |
| `set-credential` | Set data source credentials |
| `delete-credential` | Remove stored credentials |
| `list-credential` | List all stored credentials |
| `discover` | Return data source discovery info for an expression |
| `version` | Show PQTest version |

### Key Options

| Option | Short | Description |
|--------|-------|-------------|
| `--queryFile` | `-q` | Path to `.pq` file with M expression |
| `--prettyPrint` | `-p` | Format JSON output for readability |
| `--extension` | `-e` | Custom connector (.mez/.pqx) to load |
| `--parameterQueryFile` | `-pa` | Parameter values file |
| `--authenticationKind` | `-ak` | Auth type: Anonymous, UsernamePassword, Key, OAuth2 |
| `--failOnFoldingFailure` | `-foff` | Fail if query doesn't fully fold (DirectQuery simulation) |
| `--logMashupEngineTraces` | `-l` | Enable trace logging (all, user, engine) |

## Limitations

- **Windows only** (.NET Framework 4.7.2 with native `win-x64` dependencies)
- **1000 row cap** on output
- **Single data source** per query (cannot mix `Sql.Database` + `Web.Contents` in one expression)
- Designed for connector testing; Microsoft says other usage scenarios are not officially supported
- No streaming output; full result must fit in memory

## Testing Query Folding

Use `--failOnFoldingFailure` to verify that an expression folds completely:

```powershell
PQTest.exe run-test -q query.pq -p -foff
```

If any step breaks folding, PQTest returns an error. This simulates DirectQuery behavior.

## Comparison with Fabric executeQuery API

| Feature | PQTest | executeQuery API |
|---------|--------|------------------|
| Platform | Windows only | Any (cloud API) |
| Offline | Yes | No (needs Fabric capacity) |
| Row limit | 1000 | Mashup engine memory |
| Multiple sources | No | Yes (via connection bindings) |
| Auth | Local credential store | Azure AD / connection bindings |
| Folding test | `--failOnFoldingFailure` | Observe timing |
| Output format | JSON | Apache Arrow |

For most semantic model partition testing, the Fabric `executeQuery` API is more practical (no row cap, works on macOS, handles multiple sources). PQTest is useful for offline testing, folding validation, and connector development.

## Power BI Desktop Mashup Internals

PBI Desktop (Windows Store version `2.152.1279.0`) ships the same mashup DLLs at:

```
C:\Program Files\WindowsApps\Microsoft.MicrosoftPowerBIDesktop_<version>_x64__8wekyb3d8bbwe\bin\
```

Key files:
- `Microsoft.MashupEngine.dll` (15MB) -- the M language evaluator
- `Microsoft.MashupEngine.Library45.dll` (12MB) -- standard M library
- `Microsoft.Mashup.ScriptDom.dll` (2.4MB) -- M parser
- `Microsoft.Mashup.Container.NetFX45.exe` (25KB) -- out-of-process container host

At runtime, PBI Desktop loads the mashup engine in-process (no separate container visible). The container exe exists for out-of-process scenarios (PQTest, crash isolation).

The mashup engine communicates with the host via named pipes. The pipe name includes the process ID and a random token (visible in the WebView2 process command line as `--mojo-named-platform-channel-pipe`).

## References

- [PQTest Overview](https://learn.microsoft.com/en-us/power-query/sdk-tools/pqtest-overview)
- [PQTest Commands](https://learn.microsoft.com/en-us/power-query/sdk-tools/pqtest-commands-options)
- [Power Query SDK for VS Code](https://learn.microsoft.com/en-us/power-query/power-query-sdk-vs-code)
- [NuGet Package](https://www.nuget.org/packages/Microsoft.PowerQuery.SdkTools)
