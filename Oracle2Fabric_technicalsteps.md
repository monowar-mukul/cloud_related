Here's the detailed technical walkthrough for the three things you asked about, with official Microsoft Learn references for each step.

## 1. Set up the target

**a) Create the workspace and capacity**
- Sign into the Fabric portal → Workspaces → **+ New workspace**, assign it to an F-SKU capacityBefore you can begin creating the lakehouse, you need to create a workspace where you can build out the remainder of the tutorial.

**b) Create the Lakehouse (and/or Warehouse)**
- From the workspace: **New item** → search "Lakehouse" → give it a name → **Create**In the upper left corner of the workspace home page, select New item and then choose Lakehouse from the Store data section. Give your lakehouse a name and select Create. This provisions the Lakehouse on OneLake automaticallyA new lakehouse is created and, if this lakehouse is your first OneLake item, OneLake is provisioned behind the scenes.
- If your BI team needs T-SQL/DirectQuery semantics instead of Spark-first, create a **Fabric Data Warehouse** item the same way and land curated tables there instead of (or alongside) the Lakehouse.
- Decide your schema layout now (e.g. a Bronze/landing schema for raw Oracle-shaped tables, a Silver/curated schema for cleaned/joined tables) — makes reconciliation against Oracle much easier later.

Docs: [Lakehouse tutorial – create a workspace](https://learn.microsoft.com/en-us/fabric/data-engineering/tutorial-lakehouse-get-started) · [Create a lakehouse with OneLake](https://learn.microsoft.com/en-us/fabric/onelake/create-lakehouse-onelake)

## 2. Connect to Oracle

**a) Gateway + Oracle client**
- Install the **On-premises Data Gateway** on a machine with network access to Oracle.
- On that same gateway machine, install **OCMT (Oracle Client for Microsoft Tools)** — this is required specifically for the Oracle connectorInstall an on-premises data gateway by following this guidance. To use Oracle database connector, install Oracle Client for Microsoft Tools (OCMT) on the computer running on-premises data gateway., choosing the default client setup type during installChoose the Default Oracle Client setup type.

**b) Create the Oracle connection in Fabric**
- In Data Factory, when adding a source in a pipeline/Copy job, browse to **Connect data source**, pick the gateway, and enter server, username, passwordIn the Connect data source pane, specify the following field: Server: Specify the location of Oracle database that you want to connect to.
- Use the **Oracle connector v2.0** — v1.0 is deprecated. Note that unlike Azure Data Factory (which requires Oracle 19c+ for v2.0), Fabric's Oracle connector currently doesn't enforce a minimum version, so 11g/12c sources are usable if reachableCurrently, Microsoft Fabric Data Factory does not enforce a strict requirement for Oracle connections to be version 19c or above. In contrast to Azure Data Factory v2.0, which mandates Oracle 19c or later, Fabric Oracle connector provides greater flexibility.
- Supported activities against Oracle over the gateway are Copy, Lookup, and Script, all using Basic authenticationCopy activity (source/destination) | On-premises | Basic; Lookup activity | On-premises | Basic; Script activity | On-premises | Basic

Docs: [Oracle database connector overview](https://learn.microsoft.com/en-us/fabric/data-factory/connector-oracle-database-overview) · [Set up your Oracle database connection](https://learn.microsoft.com/en-us/fabric/data-factory/connector-oracle-database) · [Configure Oracle database in a copy activity](https://learn.microsoft.com/en-us/fabric/data-factory/connector-oracle-database-copy-activity)

## 3. Initial full load

Fabric's **Copy job** is purpose-built for this (simpler than hand-building a pipeline) — it unifies bulk, incremental, and CDC copy patterns in one itemCopy jobs in Data Factory ingest data without the need to create a Fabric data pipeline. It brings together various copy patterns such as bulk or batch, incremental or continuous copy into a unified experience.

1. New item → **Copy job** → choose Oracle as source, select tables/schemas.
2. For the first pass, set mode to **Full copy** — every run copies everythingFull copy: Every time the job runs, it copies all data from your source to your destination.
3. For large tables, Oracle supports built-in **parallel/partitioned copying** to speed the loadParallel copying from an Oracle database source. — auto-partitioning is supported for Oracle sources specificallyAuto-partitioning is supported for watermark-based incremental copy including both initial full copy and incremental copy, on the following connectors: ... Oracle
4. Set destination to your Lakehouse (Delta tables) or Warehouse tables.
5. Run once, then validate row counts/checksums against Oracle before moving to incremental.

Docs: [What is Copy job in Data Factory?](https://learn.microsoft.com/en-us/fabric/data-factory/what-is-copy-job) · [Quickstart: Create a Copy job](https://learn.microsoft.com/en-us/fabric/data-factory/quickstart-copy-job)

## 4. Delta/incremental sync before final sync

1. Edit the same Copy job (or create a new one per table) and switch mode to **Incremental copy**: first run copies everything, every run after only moves new/changed rowsIncremental copy: The first run copies everything, and subsequent runs only move new or changed data since the last run.
2. Pick a **watermark/incremental column** per table — Copy job now supports ROWVERSION, Date, Date/datetime, string-interpreted-as-datetime, and integer columnsCopy job supports watermark-based incremental copy (such as ROWVERSION, datetime, date, string interpreted as datetime, and integer columns) and CDC-based incremental copy when CDC is enabled on the source. — in the UI: select your table, choose the incremental column, pick its typeSelect your source table. Choose an incremental column. Pick ROWVERSION, Date, or String (datetime) based on your schema.
3. Copy job tracks watermark state automatically and only pulls rows greater than the last-copied valueWhen Copy job copies from a database using an incremental column ("watermark column"), each subsequent load copies only rows with a value in that column larger than any row previously copied. It's resilient to failures — a failed run doesn't corrupt the watermark, it just resumesIf a copy job fails, you don't need to worry about data loss. Copy job always resumes from the end of the last successful run.
4. Schedule the job at the cadence you need (Copy job supports multiple schedules on one job, e.g. hourly weekday + a separate weekend cadence)A single copy job can have multiple schedules—for example, one running daily at 6 AM and another running weekly on Sundays.
5. If you ever need to force a full re-baseline (e.g. after a schema change), you can **reset** the incremental state per table or for the whole job without touching existing destination dataReset clears the Copy job's incremental state (watermark/checkpoint) for the selected source(s)... Reset does not delete, truncate, or otherwise change data in the destination.
6. Remember watermark-based incremental copy **won't catch hard deletes** in Oracle — plan periodic full reconciliation or a soft-delete flag for those tables, as discussed earlier.

Docs: [Incremental copy in Copy job](https://learn.microsoft.com/en-us/fabric/data-factory/incremental-copy-job) · [Tutorial: Incremental copy (Warehouse→Lakehouse pattern, applicable structure)](https://learn.microsoft.com/en-us/fabric/data-factory/tutorial-incremental-copy-data-warehouse-lakehouse)

## Before the final sync / cutover

1. Let the incremental Copy jobs run on schedule through your whole parallel-run window.
2. Automate a reconciliation check after each run (row counts + key aggregates, Oracle vs Fabric) — a simple notebook or stored procedure comparing `COUNT(*)`/`SUM()` per table is usually enough.
3. When ready to cut over: freeze Oracle writes briefly, trigger one last manual Copy job run (it will pick up only the final delta since last watermark), validate counts one more time, then repoint consumers and decommission Oracle.Want me to turn this whole thread (architecture diagram + setup steps) into a formatted Word doc for your solution architecture package?
