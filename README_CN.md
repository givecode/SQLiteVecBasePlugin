# SQLiteVecBasePlugin for Unreal Engine 5

<!-- 根据 SQLiteVecBasePlugin VersionName 1.1（Version 2）生成 -->

[English](README.md) | [简体中文](README_CN.md)

SQLiteVecBasePlugin 是一款面向 Unreal Engine 5 的轻量级、完全离线 SQLite 与向量检索运行时插件。它提供可由蓝图调用的标准 SQLite 操作、`sqlite-vec` 向量存储以及 K 近邻（KNN）检索接口。

插件完全在本地运行，不需要数据库服务器或网络连接，适用于本地 AI 记忆、RAG 检索、语义搜索、NPC 知识库、离线数据集，以及需要持久化结构化数据的游戏和仿真工具。

> 本插件负责向量的存储与检索，不负责生成 Embedding。插入数据库之前，需要通过 Embedding 模型或服务生成 `float32` 向量。

## 当前兼容性

| 项目 | 当前支持配置 |
| --- | --- |
| 插件版本 | 1.1 |
| Unreal Engine | 5.6 |
| 平台 | Windows 64 位 |
| 模块类型 | Runtime |
| 蓝图调用 | 支持 |
| 打包后运行 | 支持，Unreal Build Tool 会暂存随附 DLL |
| SQLite | 3.50.3 |
| sqlite-vec | v0.1.7-alpha.2 |

当前 `.uplugin` 文件明确指定 Unreal Engine 5.6，并且只允许 `Win64` 平台。本版本不对其他引擎版本或平台作出兼容性承诺。

## 主要功能

- 完全离线的 SQLite 数据库存储。
- 同时支持编辑器环境和打包后的运行时程序。
- 通过蓝图打开和关闭数据库。
- 使用结构化列定义创建普通 SQLite 数据表。
- 支持 `INTEGER`、`REAL`、`TEXT`、`BLOB`、`NUMERIC` 等常用 SQLite 类型。
- 支持主键、自增主键、`NOT NULL` 和 `UNIQUE` 等常用约束。
- 支持自定义 SQL 类型和约束，并对明显不安全的 SQL 片段进行基础拦截。
- 创建可配置向量维度的 `vec0` 虚拟表。
- 按 `rowid` 插入、替换、更新和删除向量。
- 执行 K 近邻向量检索，并返回记录标识与距离。
- 直接执行 `CREATE`、`INSERT`、`UPDATE`、`DELETE` 等 SQL 语句。
- 将 SQL 查询结果返回为蓝图友好的行结构。
- 将单行或多行查询结果转换为 JSON 字符串。
- 对浮点数组进行补齐或截断，以适配固定维度的 Embedding。
- 自动加载并注册随附的 `sqlite-vec` 扩展。
- 在常规 Win64 编译和打包过程中自动暂存 `SQLiteCore.dll` 与 `SQLiteVec.dll`。

## 安装方法

### 从 Fab 安装

通过 [Fab 商品页面](https://www.fab.com/listings/0ec90437-839b-4833-bc53-18f27c6ff901)安装插件，在 Unreal Editor 中打开 **编辑（Edit）> 插件（Plugins）**，启用 **SQLiteVecBasePlugin**，然后按提示重启编辑器。

### 从 GitHub 仓库安装

1. 关闭 Unreal Editor。
2. 将完整插件目录复制到以下任意位置：

   ```text
   <你的项目>/Plugins/SQLiteVecBasePlugin/
   ```

   或：

   ```text
   <UnrealEngine>/Engine/Plugins/Marketplace/SQLiteVecBasePlugin/
   ```

3. 保持下面的第三方文件目录结构不变。DLL 文件名和路径不可随意修改：

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

4. 如果使用源码版本，请重新生成项目文件，并使用 Unreal Engine 5.6 所需的编译工具链编译项目。
5. 打开项目，在 **编辑（Edit）> 插件（Plugins）** 中启用 **SQLiteVecBasePlugin**，然后重启编辑器。

## 蓝图快速入门

典型的向量检索流程如下：

1. 在项目可写的 `Saved` 目录中构造数据库路径，并提前创建父目录。
2. 调用 **Open Database**。返回非零整数表示成功，该整数是当前进程内有效的数据库句柄；返回 `0` 表示打开失败。
3. 调用 **Create SQLite Vec Table if not exists**，设置表名以及 Embedding 模型输出的准确维度。
4. 调用 **Insert Vec Row**，按整数 `rowid` 插入每个 `float32` 向量。
5. 调用 **Vector Query (K-NN)**，传入相同维度的查询向量和需要返回的结果数量 `K`。
6. 读取每条结果的 `Id` 和 `Distance`。
7. 数据库使用完毕后调用 **Close Database**。

向量表辅助函数创建的结构如下：

```sql
CREATE VIRTUAL TABLE IF NOT EXISTS items
USING vec0(
    rowid INTEGER PRIMARY KEY,
    embedding float[1536]
);
```

`items` 和 `1536` 是默认值。请将维度改为 Embedding 模型的实际输出维度。插入向量和查询向量的长度必须与建表维度一致。

## 普通 SQLite 使用流程

即使不使用向量检索，也可以将本插件作为普通 SQLite 运行时插件使用：

1. 使用 **Open Database** 打开或创建数据库。
2. 使用 **Create Table** 创建数据表，或者使用 **Execute SQL** 执行自定义 DDL。
3. 使用 **Execute SQL** 执行不需要返回结果集的语句。
4. 使用 **Query SQL** 执行 `SELECT` 查询。
5. 根据需要，使用 **SQLite Row To Json String** 或 **SQLite Rows To Json String** 将 `FSQLiteRow` 转换成 JSON。
6. 关闭数据库句柄。

SQL 示例：

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

## 蓝图 API 参考

| 函数 | 用途 |
| --- | --- |
| `OpenDatabase` | 打开已有 SQLite 文件或创建新文件，并返回整数句柄。 |
| `CloseDatabase` | 关闭与句柄关联的 SQLite 连接。 |
| `CreateTable` | 根据 `FSQLiteColumnDef` 数组创建普通 SQLite 数据表。 |
| `CreateVecTable` | 创建包含 `rowid` 和 `embedding float[N]` 的 `vec0` 表。 |
| `TableExists` | 检查指定数据表或视图是否存在。 |
| `DropTable` | 如果数据表存在，则将其删除。 |
| `InsertVecRow` | 按 `rowid` 插入或替换向量记录。 |
| `UpdateVecRow` | 按 `rowid` 更新指定的向量列。 |
| `DeleteVecRow` | 按 `rowid` 删除向量记录。 |
| `VectorQuery` | 执行 KNN 查询，并返回记录标识和距离。 |
| `ExecuteSQL` | 执行不需要返回结果集的 SQL。 |
| `QuerySQL` | 执行查询，并返回列名与字符串形式的列值。 |
| `SQLiteRowToJsonString` | 将一条 `FSQLiteRow` 转换为 JSON 对象字符串。 |
| `SQLiteRowsToJsonString` | 将多条记录转换为 JSON 数组字符串。 |
| `PadFloatArray` | 将浮点数组补齐或截断到固定长度。 |

### 结构化数据表定义

`FSQLiteColumnDef` 支持以下内置类型：

- `INTEGER`
- `REAL`
- `TEXT`
- `BLOB`
- `NUMERIC`
- 自定义 SQL 类型

支持以下列约束：

- 无约束
- `PRIMARY KEY`
- `PRIMARY KEY AUTOINCREMENT`
- `NOT NULL`
- `UNIQUE`
- 自定义 SQL 约束

`PRIMARY KEY AUTOINCREMENT` 只能用于 `INTEGER` 列。自定义 SQL 片段会拦截分号和 SQL 注释标记，但仍应当只允许开发者配置，不能直接接受玩家或网络输入。

## C++ 使用方法

在项目模块的依赖中加入插件模块：

```csharp
PrivateDependencyModuleNames.AddRange(new string[]
{
    "SQLiteVecBasePlugin"
});
```

最小示例：

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

实际项目中，应当检查每次操作返回的布尔值和 `OutError`。

## 运行时与打包说明

- 插件模块类型为 Runtime，因此蓝图接口可以在打包后的程序中使用。
- Unreal Build Tool 会在 Win64 编译时将 `SQLiteCore.dll` 和 `SQLiteVec.dll` 暂存到目标输出目录。
- 运行时首先从项目可执行文件附近的 `Binaries/Win64` 目录查找 DLL。
- 在非 Shipping 构建中，插件还会尝试从 `Source/ThirdParty/Bin` 和插件自身的 `Binaries/Win64` 目录查找 DLL。
- 数据库文件应保存在具有写入权限的位置。打包程序推荐使用 `FPaths::ProjectSavedDir()`。
- 蓝图数据库和查询函数均为同步调用，不要在每一帧执行大型 SQL 查询或大量 KNN 检索。

## 重要行为说明

- 数据库句柄是当前进程内有效的整数。不要将句柄保存到磁盘，也不要在关闭连接或重启程序后继续使用旧句柄。
- 每个成功打开的数据库都应当调用 **Close Database**。
- `QuerySQL` 会将查询结果统一暴露为字符串。SQLite 的 `NULL` 会变为空字符串，转换 JSON 后，各字段值也会被序列化为 JSON 字符串。
- `FVectorSearchResult::Id` 是由整数 `rowid` 转换得到的字符串。
- 蓝图封装层不会提前验证向量数组长度；维度不一致时，错误会在 SQLite/sqlite-vec 执行阶段返回。
- `ExecuteSQL` 和 `QuerySQL` 会执行开发者提供的原始 SQL。不要将未经处理的玩家输入或网络输入拼接到 SQL 中。
- 部分向量辅助函数会直接使用传入的表名或列名，因此这些标识符必须是可信的开发配置，不能来自不可信输入。

## 诊断命令

在编辑器或 Development 构建中打开 Unreal 控制台并执行：

```text
SQLiteVec.Test
```

也可以传入数据库路径：

```text
SQLiteVec.Test D:/Temp/sqlite_vec_test.db
```

该命令会创建一个小型向量表、插入测试数据、执行一次 KNN 查询，并将结果写入 `LogSQLiteVec` 日志分类。不指定路径时，默认使用 `Saved/sqlite_vec_test.db`。

## 常见问题

### `OpenDatabase` 返回 `0`

- 确认目标路径具有写入权限。
- 在打开数据库之前先创建父目录。
- 在输出日志中查找 `LogSQLiteVec` 错误。

### 提示 `SQLiteCore.dll not found` 或 `SQLiteVec.dll not found`

- 确认源码插件的 `Source/ThirdParty/Bin` 中同时存在两个 DLL。
- 通过 Unreal Build Tool 重新编译或打包，使运行时依赖被正确暂存。
- 在最终 Win64 输出目录中，确认两个 DLL 位于可执行文件附近的目标输出目录。

### 插入向量或向量检索失败

- 确认插入向量、查询向量和数据表声明的维度完全一致。
- 确认数据表由 `CreateVecTable` 创建，或者使用了兼容的 `vec0` 结构。
- 查看 `OutError` 和 Unreal 输出日志。

### C++ 找不到蓝图函数库头文件

- 将 `SQLiteVecBasePlugin` 添加到调用方模块的依赖列表。
- 重新生成项目文件并编译。

## 第三方组件与许可证

本插件包包含：

- **SQLite 3.50.3**：公共领域软件，详见随附的 SQLite Public Domain 声明。
- **sqlite-vec v0.1.7-alpha.2**：采用 MIT License，Copyright (c) 2024 Alex Garcia and contributors，详见随附的 sqlite-vec MIT License。

重新分发时，请保留随附的第三方许可证与声明文件。除非产品条款或仓库中的其他授权另有说明，插件自身源码文件标注为 `Copyright (c) 2025 haozena. All Rights Reserved`。

## 文档、问题反馈与功能建议

项目文档、错误反馈和新功能建议均可通过本仓库提交：

- 项目仓库：[givecode/SQLiteVecBasePlugin](https://github.com/givecode/SQLiteVecBasePlugin)
- 反馈问题：[提交 GitHub Issue](https://github.com/givecode/SQLiteVecBasePlugin/issues)
- 提出新需求：[提交 GitHub Issue](https://github.com/givecode/SQLiteVecBasePlugin/issues)

提交问题时，建议附上 Unreal Engine 版本、插件版本、Windows 版本、问题发生在编辑器还是打包程序、相关 `LogSQLiteVec` 日志，以及可以复现问题的最少步骤。
