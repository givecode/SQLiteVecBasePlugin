# SQLiteVecBasePlugin for Unreal Engine 5 (UE5)

<!-- Generated from SQLiteVecBasePlugin VersionName 1.1 (Version 2) -->

[English](README.md) | [简体中文](README_CN.md)

SQLiteVecBasePlugin is a lightweight, fully offline SQLite and vector-search runtime plugin for Unreal Engine 5. It exposes Blueprint-callable APIs for standard SQLite operations, `sqlite-vec` vector storage, and K-nearest-neighbor (KNN) search.

The plugin runs locally and does not require a database server or network connection. It is suitable for local AI memory, RAG retrieval, semantic search, NPC knowledge bases, offline datasets, and gameplay tools that need persistent structured data.

> This plugin stores and searches vectors. It does not generate embeddings. Use an embedding model or service to create the `float32` vectors before inserting them.

## Current Compatibility

| Item | Supported configuration |
| --- | --- |
| Plugin version | 1.1 |
| Unreal Engine | 5.6 |
| Platform | Windows 64-bit |
| Module type | Runtime |
| Blueprint access | Yes |
| Packaged builds | Yes, with the bundled DLLs staged by Unreal Build Tool |
| SQLite | 3.50.3 |
| sqlite-vec | v0.1.7-alpha.2 |

The supplied plugin descriptor explicitly targets Unreal Engine 5.6 and allows only `Win64`. Other engine versions and platforms are not claimed as supported by this package.

## Features

- Fully offline SQLite database storage.
- Runtime support for both Editor sessions and packaged games.
- Blueprint-callable database open and close operations.
- Structured creation of standard SQLite tables.
- Standard SQLite types: `INTEGER`, `REAL`, `TEXT`, `BLOB`, and `NUMERIC`.
- Common column constraints, including primary keys, autoincrement, `NOT NULL`, and `UNIQUE`.
- Optional custom SQL types and constraints with basic unsafe-fragment rejection.
- Creation of `vec0` virtual tables with configurable vector dimensions.
- Insert, replace, update, and delete vector rows by `rowid`.
- K-nearest-neighbor vector search with row identifiers and distances.
- Direct execution of SQL statements such as `CREATE`, `INSERT`, `UPDATE`, and `DELETE`.
- SQL queries returned as Blueprint-friendly row structures.
- Conversion of one row or multiple rows to JSON strings.
- Float-array padding and truncation for fixed-size embedding vectors.
- Automatic loading and registration of the bundled `sqlite-vec` extension.
- Automatic Win64 staging of `SQLiteCore.dll` and `SQLiteVec.dll` during normal Unreal builds and packaging.

## Installation

### Install from Fab

Install the plugin from the [Fab listing](https://www.fab.com/listings/0ec90437-839b-4833-bc53-18f27c6ff901), enable **SQLiteVecBasePlugin** in **Edit > Plugins**, and restart Unreal Editor when prompted.

### Install from the repository

1. Close Unreal Editor.
2. Copy the complete plugin directory to one of these locations:

   ```text
   <YourProject>/Plugins/SQLiteVecBasePlugin/
   ```

   or:

   ```text
   <UnrealEngine>/Engine/Plugins/Marketplace/SQLiteVecBasePlugin/
   ```

3. Keep the third-party files in the following layout. The DLL names and paths are significant:

   ```text
   SQLiteVecBasePlugin/
   ├── SQLiteVecBasePlugin.uplugin
   └── Source/
       ├── SQLiteVecBasePlugin/
       │   ├── SQLiteVecBasePlugin.Build.cs
       │   ├── Public/
       │   └── Private/
       └── ThirdParty/
           ├── Include/
           │   ├── sqlite3.h
           │   ├── sqlite3ext.h
           │   └── sqlite-vec.h
           └── Bin/
               ├── SQLiteCore.dll
               └── SQLiteVec.dll
   ```

4. If you are using the source distribution, regenerate the project files and build the project with the toolchain required by Unreal Engine 5.6.
5. Open the project, enable **SQLiteVecBasePlugin** in **Edit > Plugins**, and restart the Editor.

## Blueprint Quick Start

A typical vector-search flow is:

1. Build a database path inside your project's writable `Saved` directory. Make sure its parent directory already exists.
2. Call **Open Database**. A non-zero integer is a valid process-local database handle; `0` means the database could not be opened.
3. Call **Create SQLite Vec Table if not exists** with a table name and the exact dimension produced by your embedding model.
4. Call **Insert Vec Row** for each `float32` vector and integer `rowid`.
5. Call **Vector Query (K-NN)** with a query vector of the same dimension and the desired value of `K`.
6. Read each result's `Id` and `Distance`.
7. Call **Close Database** when the connection is no longer needed.

The vector-table helper creates this schema:

```sql
CREATE VIRTUAL TABLE IF NOT EXISTS items
USING vec0(
    rowid INTEGER PRIMARY KEY,
    embedding float[1536]
);
```

`items` and `1536` are defaults. Replace the dimension with the output size of your embedding model. Inserted and queried vectors must match that dimension.

## Standard SQLite Workflow

The plugin can also be used without vector search:

1. Open or create a database with **Open Database**.
2. Create a table with **Create Table**, or execute your own DDL with **Execute SQL**.
3. Use **Execute SQL** for statements that do not return rows.
4. Use **Query SQL** for `SELECT` statements.
5. Optionally convert the returned `FSQLiteRow` values with **SQLite Row To Json String** or **SQLite Rows To Json String**.
6. Close the database handle.

Example SQL:

```sql
CREATE TABLE IF NOT EXISTS notes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    content TEXT NOT NULL,
    category TEXT
);

INSERT INTO notes (content, category)
VALUES ('Forklift inspection completed', 'maintenance');

SELECT id, content, category FROM notes;
```

## Blueprint API Reference

| Function | Purpose |
| --- | --- |
| `OpenDatabase` | Opens an existing SQLite file or creates one and returns an integer handle. |
| `CloseDatabase` | Closes the SQLite connection associated with a handle. |
| `CreateTable` | Creates a standard SQLite table from `FSQLiteColumnDef` entries. |
| `CreateVecTable` | Creates a `vec0` table with `rowid` and an `embedding float[N]` column. |
| `TableExists` | Checks whether a table or view exists. |
| `DropTable` | Drops a table if it exists. |
| `InsertVecRow` | Inserts or replaces a vector row by `rowid`. |
| `UpdateVecRow` | Replaces a vector column for an existing `rowid`. |
| `DeleteVecRow` | Deletes a vector row by `rowid`. |
| `VectorQuery` | Performs a KNN query and returns identifiers and distances. |
| `ExecuteSQL` | Executes SQL that does not need a result set. |
| `QuerySQL` | Executes a query and returns column names and string values. |
| `SQLiteRowToJsonString` | Converts one `FSQLiteRow` to a JSON object string. |
| `SQLiteRowsToJsonString` | Converts multiple rows to a JSON array string. |
| `PadFloatArray` | Pads or truncates a float array to a fixed length. |

### Structured table definitions

`FSQLiteColumnDef` supports these built-in types:

- `INTEGER`
- `REAL`
- `TEXT`
- `BLOB`
- `NUMERIC`
- Custom SQL type

It supports these constraints:

- None
- `PRIMARY KEY`
- `PRIMARY KEY AUTOINCREMENT`
- `NOT NULL`
- `UNIQUE`
- Custom SQL constraint

`PRIMARY KEY AUTOINCREMENT` requires an `INTEGER` column. Custom fragments reject semicolons and SQL comment markers, but they should still be treated as trusted developer-authored configuration.

## C++ Usage

Add the plugin module to your project's module dependencies:

```csharp
PrivateDependencyModuleNames.AddRange(new string[]
{
    "SQLiteVecBasePlugin"
});
```

Minimal example:

```cpp
#include "SQLiteVecBaseBPLibrary.h"
#include "HAL/FileManager.h"
#include "Misc/Paths.h"

const FString DatabaseDirectory = FPaths::Combine(
    FPaths::ProjectSavedDir(),
    TEXT("SQLiteVec"));
IFileManager::Get().MakeDirectory(*DatabaseDirectory, true);

const FString DatabasePath = FPaths::Combine(
    DatabaseDirectory,
    TEXT("Example.db"));

const int32 DatabaseHandle = USQLiteVecBPLibrary::OpenDatabase(DatabasePath);
if (DatabaseHandle != 0)
{
    FString Error;

    if (USQLiteVecBPLibrary::CreateVecTable(
            DatabaseHandle,
            Error,
            TEXT("items"),
            3))
    {
        const TArray<float> VectorA{0.1f, 0.2f, 0.3f};
        USQLiteVecBPLibrary::InsertVecRow(
            DatabaseHandle,
            TEXT("items"),
            1,
            VectorA,
            Error);

        TArray<FVectorSearchResult> Results;
        const TArray<float> QueryVector{0.1f, 0.2f, 0.25f};
        USQLiteVecBPLibrary::VectorQuery(
            DatabaseHandle,
            TEXT("items"),
            QueryVector,
            5,
            Results,
            Error);
    }

    USQLiteVecBPLibrary::CloseDatabase(DatabaseHandle);
}
```

Check the returned Boolean and `OutError` after each operation in production code.

## Runtime and Packaging Notes

- The module is a runtime module, so its Blueprint APIs are available in packaged applications.
- Unreal Build Tool stages `SQLiteCore.dll` and `SQLiteVec.dll` into the target output directory for Win64 builds.
- At runtime, the plugin first looks for the DLLs next to the project's executable in `Binaries/Win64`.
- In non-Shipping builds, it can also look in the plugin's `Source/ThirdParty/Bin` and plugin `Binaries/Win64` directories.
- Database files should be stored in a writable location. `FPaths::ProjectSavedDir()` is the recommended default for packaged applications.
- Blueprint database and query functions are synchronous. Avoid large queries or heavy KNN workloads on every frame.

## Important Behavior

- Database handles are process-local integers. Do not save them to disk or reuse them after closing the connection or restarting the application.
- Always close every successful database handle.
- `QuerySQL` exposes result values as strings. SQLite `NULL` becomes an empty string, and JSON conversion also serializes returned values as JSON strings.
- `FVectorSearchResult::Id` contains the integer `rowid` formatted as a string.
- Vector dimension mismatches are reported by SQLite/sqlite-vec at execution time; the Blueprint wrapper does not pre-validate the array length.
- `ExecuteSQL` and `QuerySQL` execute developer-supplied SQL. Do not concatenate untrusted player or network input into SQL statements.
- Several vector helper functions accept table or column names directly. Treat those identifiers as trusted configuration, not user input.

## Diagnostic Command

In Editor or a development build, open the Unreal console and run:

```text
SQLiteVec.Test
```

An optional database path can be supplied:

```text
SQLiteVec.Test D:/Temp/sqlite_vec_test.db
```

The command creates a small vector table, inserts test data, performs a KNN query, and writes output to the `LogSQLiteVec` log category. Without a path, it uses `Saved/sqlite_vec_test.db`.

## Troubleshooting

### `OpenDatabase` returns `0`

- Confirm the path is writable.
- Create the parent directory before opening the database.
- Check the Output Log for `LogSQLiteVec` errors.

### `SQLiteCore.dll not found` or `SQLiteVec.dll not found`

- Confirm both DLLs exist in `Source/ThirdParty/Bin` in the source plugin.
- Rebuild or repackage through Unreal Build Tool so the runtime dependencies are staged.
- In the final Win64 output, confirm both DLLs are located next to the executable in the target output directory.

### Vector insertion or search fails

- Confirm the insert vector, query vector, and table dimension are identical.
- Confirm the table was created with `CreateVecTable` or a compatible `vec0` schema.
- Inspect `OutError` and the Unreal Output Log.

### C++ cannot find the Blueprint library header

- Add `SQLiteVecBasePlugin` to the consuming module's dependency list.
- Regenerate project files and rebuild.

## Third-Party Components and Licenses

This package includes:

- **SQLite 3.50.3** — public domain. See the included SQLite public-domain notice.
- **sqlite-vec v0.1.7-alpha.2** — MIT License, Copyright (c) 2024 Alex Garcia and contributors. See the included sqlite-vec MIT license.

Keep the supplied third-party license and notice files with redistributions. The plugin's own source files state `Copyright (c) 2025 haozena. All Rights Reserved` unless separate product or repository terms grant additional rights.

## Documentation and Support

Documentation, bug reports, and feature requests are handled through this repository:

- Repository: [givecode/SQLiteVecBasePlugin](https://github.com/givecode/SQLiteVecBasePlugin)
- Report a bug: [Open a GitHub issue](https://github.com/givecode/SQLiteVecBasePlugin/issues)
- Request a feature: [Open a GitHub issue](https://github.com/givecode/SQLiteVecBasePlugin/issues)

When reporting a problem, include the Unreal Engine version, plugin version, Windows version, whether the issue occurs in Editor or a packaged build, relevant `LogSQLiteVec` output, and minimal reproduction steps.
