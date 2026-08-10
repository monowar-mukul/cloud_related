# Moving off Oracle: a solution architecture design document for migrating to Microsoft Fabric

Every migration project eventually runs into the same tension: the move itself is temporary, but the mistakes you make while moving can become permanent. That's the single most important idea behind the architecture in this document. There are two flows at play — a **migration/cutover flow** that exists only to get data from Oracle into Fabric safely, and a **steady-state target flow** that is what your analytics platform looks like forever after. Confusing the two, or building your permanent pipelines around temporary migration logic, is the most common way these projects go wrong.

This document walks through both flows — the six-step cutover sequence for taking Oracle out of the picture, and the permanent Bronze/Silver/Gold architecture that replaces it — and then extends into the enterprise governance framework needed once Fabric becomes the platform for more than one application: workspace and domain structure, capacity and region strategy, per-application medallion conventions, and retention/backup policy. It's written for three audiences at once — the solution architect who needs the full picture, the Oracle DBA or developer who's comfortable with databases but new to Fabric, and the non-technical stakeholder who just wants to know where their reports come from.

---

## The big picture: two flows, not one

Look at the diagram as three horizontal bands:

1. **Migration / cutover flow (temporary).** Six steps, left to right, that exist for the duration of the project only. Once step 6 (decommission) happens, this entire band disappears from your architecture.
2. **Target Fabric data flow (steady state).** The permanent pipeline: source system → Fabric Data Factory → OneLake landing → Bronze → Silver → Gold → semantic model → Power BI. This is what runs forever, long after Oracle is gone.
3. **Cross-cutting governance, security, and operations.** Identity, data governance, monitoring, release management, and retention — the guardrails that wrap around everything in bands 1 and 2.

The note under the migration band says it plainly: *this flow is temporary and should not be mixed with the steady-state target data path.* In practice, that means your migration scripts, one-time reconciliation jobs, and cutover checklists should live in their own project folder, their own pipeline items, and ideally their own workspace stage — not tangled into the Bronze/Silver/Gold pipelines that your data engineering team will own for years.

> **The order is non-negotiable.** Oracle stays live and writable through the full migration. Only once the final incremental sync has landed the last outstanding changes in Fabric — and reconciliation plus UAT have confirmed the two systems match — does cutover happen: reports get switched to Fabric. Oracle is decommissioned only *after* that cutover has been validated in production and formally approved. At no point is Oracle turned off before Fabric has proven itself on the final synced data.

---

## For the non-technical reader: think of it like moving house

If the pipeline diagram looks like plumbing, here's the version without the pipes.

Imagine your company's data currently lives in a house (Oracle, running on a virtual machine). You're moving to a new house (Microsoft Fabric) and you want the new house to be *better organized* than the old one — not just a copy of the same clutter.

- **Bronze Lakehouse** is the moving truck unloading boxes into the garage, exactly as they were packed. Nothing is unpacked or rearranged yet — it's just all safely inside the new house, with a label on every box (that's the "audit and metadata" you saw in the diagram).
- **Silver Lakehouse** is unpacking those boxes into the right rooms, checking nothing is broken, and throwing out duplicates. This is where messy, inconsistent Oracle data gets cleaned into something trustworthy.
- **Gold Warehouse** is the fully furnished, ready-to-use rooms — the kitchen is stocked, the office is set up. This is data organized specifically for the reports and dashboards people actually use, built as facts, dimensions, and marts.
- **Semantic model** is the house's directory — the sign that tells visitors "living room this way, kitchen that way" — a shared, agreed set of business definitions so two people asking "what's our revenue" get the same answer.
- **Power BI / analytics** is you, actually living in the house: dashboards, reports, and self-service tools people use every day.

The **migration flow** is simply the moving day logistics — baseline what you have, load the truck, do one last sweep for anything left behind, check nothing's missing, switch your mailing address (reports) to the new house, and only then give up the keys to the old one (decommission Oracle). Once you've moved in, nobody talks about the moving truck anymore — that's the point.

---

## For the Oracle-savvy, Fabric-first-timer: the six-step cutover, explained

You know Oracle. You don't yet know Fabric. Before the six cutover steps, two things need to exist: a place in Fabric to land the data, and a working connection back to Oracle.

**Setting up the target.** In the Fabric portal, create a workspace and assign it to an F-SKU capacity. From that workspace, use **New item** → **Lakehouse**, give it a name, and Fabric provisions the Lakehouse on OneLake automatically. If your BI team needs T-SQL/DirectQuery semantics rather than Spark-first, also create a **Fabric Data Warehouse** item the same way. Decide your schema layout up front — a Bronze/landing schema for raw Oracle-shaped tables and a Silver/curated schema for cleaned data — it makes reconciliation against Oracle much easier later.

**Connecting to Oracle.** Install the **On-premises Data Gateway** on a machine with network access to Oracle, then install **OCMT (Oracle Client for Microsoft Tools)** on that same gateway machine — this is required specifically for the Oracle connector, and the default client setup type is fine. In Data Factory, when adding a source in a pipeline or Copy job, browse to **Connect data source**, pick the gateway, and enter server, username, and password. Use the **Oracle connector v2.0** (v1.0 is deprecated) — unlike Azure Data Factory, which requires Oracle 19c+ for v2.0, Fabric's Oracle connector doesn't currently enforce a minimum version, so 11g/12c sources work if reachable. Supported activities against Oracle over the gateway are Copy, Lookup, and Script, all using Basic authentication.

With target and connectivity in place, here's what each cutover step actually looks like in the tool:

### Step 1 — Baseline source DB for migration
Before touching Fabric, document what you're moving: schemas, table row counts, data types (pay attention to Oracle `NUMBER`, `DATE`, `CLOB` — these are the usual type-mapping surprises), and which tables genuinely feed analytics versus pure OLTP noise you don't need to bring over. Capture baseline row counts and key aggregates (sums, min/max dates) per table — you'll compare against these at every later step. This baseline is your ground truth for the entire project.

### Step 2 — Initial full load to Fabric
This is Fabric's **Copy job** — purpose-built for this and simpler than hand-building a pipeline, since it unifies bulk, incremental, and CDC copy patterns in one item. From the workspace, **New item** → **Copy job** → choose Oracle as the source and select your tables/schemas. For the first pass, set the mode to **Full copy**, so every run copies everything; set the destination to your Bronze Lakehouse (Delta tables) or Warehouse tables. For large tables, use Oracle's built-in parallel/partitioned copy support — auto-partitioning is supported specifically for Oracle sources — so the load doesn't run overnight unnecessarily. Land the data as-is — don't transform anything yet. Bronze exists specifically so you have an unmodified, auditable copy of what Oracle actually contained. Run it once, then validate row counts and checksums against Oracle before moving to incremental.

### Step 3 — Incremental / final delta sync
Edit the same Copy job and switch mode to **Incremental copy**: the first run copies everything, every run after only moves new or changed rows. Pick a watermark column per table — Copy job supports ROWVERSION, date/datetime, string-interpreted-as-datetime, and integer columns — by selecting your source table, choosing the incremental column, and picking its type in the UI. Copy job tracks the watermark state automatically, pulling only rows with a value greater than the last-copied one, and it's resilient to failures: a failed run doesn't corrupt the watermark, it just resumes from the last successful run. Schedule it at the cadence you need — a single Copy job supports multiple schedules, e.g. hourly on weekdays plus a separate weekend cadence — and let it run throughout the parallel-run period, with Oracle continuing to take writes as normal. If you ever need to force a full re-baseline (say, after a schema change), you can **reset** the incremental state per table or for the whole job without touching existing destination data.

This step ends with one specific run that matters more than the rest: the **final delta sync**. Once you're ready to cut over, trigger one last incremental run to pull every remaining change out of Oracle, right up to the cutover moment. This final sync is what step 4's reconciliation is checked against — Fabric should now match Oracle exactly, with nothing left behind. Remember: watermark-based incremental copy won't catch hard deletes in Oracle, so either use a soft-delete flag or schedule periodic full reconciliation for tables where deletes matter, and fold that check into the final sync too.

### Step 4 — Reconcile record counts + UAT
This is where the discipline pays off. Compare row counts, checksums, and key business metrics between Oracle and Fabric after every incremental run, ideally automated (a notebook or scheduled query that flags mismatches). In parallel, have business users run their existing Oracle-fed reports side by side with the equivalent new Fabric-fed reports (built on your Silver/Gold layers) and sign off that the numbers match. Don't skip UAT because the row counts look right — a right row count with wrong values is worse than an obvious failure.

### Step 5 — Switch reports to Fabric
Once reconciliation against the final sync and UAT both pass, repoint Power BI reports and any downstream consumers from Oracle to Fabric, one workspace or report group at a time rather than all at once. This is the actual cutover moment — it only happens after the final sync in step 3 has landed and been validated, never before. A staged cutover also limits blast radius if something was missed, and gives you a rollback path (Oracle is still running, untouched, and still holding the pre-cutover state) until you're fully confident.

### Step 6 — Decommission Azure DB server when approved
Only after every consumer has been repointed and a sign-off has been obtained do you turn Oracle off. Oracle is never stopped or made read-only before this point — it stays live and writable all the way through steps 1–5 so there's always a safe fallback. Because your permanent pipeline (Bronze → Silver → Gold) never depended on Oracle staying live — it was fed by the migration Copy jobs, not a live mirror — Fabric doesn't break when Oracle disappears. This is the payoff of keeping the two flows separate from day one.

**References:** [Lakehouse tutorial – create a workspace](https://learn.microsoft.com/en-us/fabric/data-engineering/tutorial-lakehouse-get-started) · [Create a lakehouse with OneLake](https://learn.microsoft.com/en-us/fabric/onelake/create-lakehouse-onelake) · [Oracle database connector overview](https://learn.microsoft.com/en-us/fabric/data-factory/connector-oracle-database-overview) · [Set up your Oracle database connection](https://learn.microsoft.com/en-us/fabric/data-factory/connector-oracle-database) · [Configure Oracle database in a copy activity](https://learn.microsoft.com/en-us/fabric/data-factory/connector-oracle-database-copy-activity) · [What is Copy job in Data Factory?](https://learn.microsoft.com/en-us/fabric/data-factory/what-is-copy-job) · [Quickstart: Create a Copy job](https://learn.microsoft.com/en-us/fabric/data-factory/quickstart-copy-job) · [Incremental copy in Copy job](https://learn.microsoft.com/en-us/fabric/data-factory/incremental-copy-job)

---

## The permanent architecture: what's left after Oracle is gone

Once the migration band is retired, this is what remains — and what your data engineering team owns going forward.

**Source → Fabric Data Factory → OneLake landing.** Data Factory pipelines or Copy activities, connecting over the gateway (or private connectivity if you've moved to cloud-native sources), land raw files into OneLake as Parquet, CSV, or Delta. This landing zone is explicitly called out as **permanent storage in OneLake** — it isn't just a migration staging area, it's the ongoing ingestion point for whatever your source systems are going forward.

**Bronze Lakehouse — raw Delta tables, audit + metadata.** The unmodified, schema-preserved copy of source data, with lineage and audit metadata attached. Nobody builds reports directly on Bronze.

**Silver Lakehouse — cleanse, validate, conform.** Reusable curated tables: deduplicated, type-corrected, business-rule-validated. This is the layer other teams should build on top of, rather than each team re-cleaning the same raw data independently.

**Gold Warehouse — facts, dimensions, marts.** A SQL-first analytical model purpose-built for reporting: star schemas, business-ready aggregates. The diagram notes the **warehouse serves curated analytics** — this is the layer optimized for query performance and business consumption, not raw flexibility.

**Semantic model — measures, RLS, business definitions.** A single governed layer of business logic (measures, row-level security, naming) sitting between Gold and Power BI, so every report and every team uses the same definition of revenue, customer, or region.

**Power BI / analytics — dashboards, self-service BI, data products.** The consumption layer: certified reports, self-service exploration, and packaged data products for downstream teams or systems.

Two supporting functions feed this pipeline continuously:

- **Data Engineering** (Spark notebooks, Dataflows Gen2, data quality rules, orchestration) owns the Bronze → Silver transformation logic.
- **Data Analytics** (Warehouse SQL, semantic modeling, Power BI) owns the Silver/Gold → semantic model → Power BI layer, publishing certified datasets and reports.

---

## The part people forget: governance, security, and operations

Running underneath both flows is a set of cross-cutting concerns that apply from day one of migration through the life of the platform:

- **Identity and access** — Microsoft Entra ID, RBAC, workspace roles, and row-level security control who can see what, in both the migration tooling and the permanent platform.
- **Data governance** — Microsoft Purview provides the catalog, lineage tracking, and sensitivity labeling that lets you answer "where did this number come from" months after the migration team has moved on.
- **Operations** — monitoring, alerting, pipeline run history, and audit logs, so failed incremental syncs or broken Gold-layer refreshes get caught before a business user notices bad numbers.
- **DevOps / release** — Git integration, deployment pipelines, and environment promotion (dev → test → prod) so changes to Bronze/Silver/Gold logic go through the same rigor as any other production system.
- **Retention / archive** — OneLake retention policy and backup/export rules, decided deliberately rather than left as a default.

None of this is optional scaffolding — it's what turns "we copied Oracle into Fabric" into "we have a governed, auditable analytics platform." Oracle was one application. Once Fabric is the platform of record, other applications will land on it too — and each one needs its own workspace, its own retention posture, and its own place in the medallion structure. The rest of this section is the governance framework for that broader estate.

### Domain and workspace structure — decide this before onboarding the next application

The most common failure mode in multi-application Fabric estates: teams create workspaces first and retrofit structure later, and the catalog never reflects reality after thatIn practice, domain design gets skipped more often than any other governance step. Teams create workspaces first, then try to retrofit domains onto an estate that has already grown — and the catalog never reflects reality after that. The fifteen minutes spent on a domain map before provisioning saves weeks of cleanup later.

- **Domains first, workspaces second.** Use Fabric domains to group workspaces by business area (sales, finance, HR, and — for this migration — the Oracle-sourced analytics domain), each with a named owner<cite index="48-1">Sales, finance, operations, customer service, and HR should not all dump assets into one shared workspace. Use domains to reflect those business areas, then assign each domain an owner.</cite> Domains let users filter and govern the OneLake catalog by business area rather than hunting through a flat list of workspaces<cite index="46-1">Use domains to logically group all the data in an organization that's relevant to particular areas or fields, such as by business unit... Grouping data into domains and subdomains enables better discoverability and governance.</cite>
- **One workspace per application/data product is the default**, with each team owning its full pipeline<cite index="45-1">Option 1: One workspace per data product. In this model, each data product gets one Fabric workspace... Choose this option when one team owns the full pipeline and when moderate isolation meets governance needs.</cite> Only consolidate multiple low-risk apps into a shared workspace when capacity or licensing genuinely forces it, and treat that as a temporary exception, not a pattern<cite index="45-1">Option 3: Consolidate multiple data products into one workspace only when constrained... Limit use to low-criticality workloads and plan a future move to dedicated workspaces.</cite>
- **Separate dev/test/prod workspaces per application** so developers don't collide with shared environments<cite index="46-1">For development purposes, a best practice is to have isolated workspaces per developer, so each developer can work on their own without interfering with the shared workspace.</cite> — this also gives a clean environment-promotion boundary for CI/CD.
- **Decide item separation before workspace separation.** Will Bronze/Silver/Gold for each app live in one Lakehouse or three separate items? Will each app's Gold layer live in its own workspace, or will multiple apps share a curated workspace? This cascades directly into the access-control model.
- **Enforce a naming/creation policy** — who's allowed to create workspaces, naming conventions, mandatory domain assignment — or workspaces accumulate with unclear ownership and no catalog visibility<cite index="44-1">Without a creation policy, workspaces accumulate. Unmanaged workspaces have unclear ownership, inconsistent role assignments, and no catalog visibility. They become governance blind spots with no straightforward path to cleanup.</cite>

### Capacity and region

- Each capacity lives in one Azure region, which determines where compute and OneLake data for its workspaces reside — a data-residency and latency decision, not just a technical one<cite index="43-1">Each Fabric capacity runs in a single Azure region. That region determines where compute and OneLake data for workspaces on that capacity reside. Region decisions affect data residency, latency, and service availability. Best practices: treat region selection as a governance decision.</cite>
- Decide per-application capacity isolation vs. shared capacity: isolated capacity gives an app predictable performance and clean cost attribution; shared capacity is cheaper but one noisy app can starve others. This is usually the single biggest cost-governance decision in a multi-app estate.
- Publish an approved region list and enforce it through policy rather than leaving it to individual teams<cite index="43-1">Publish a short list of supported regions and enforce that list through policy.</cite>

### Medallion architecture — per application, with shared conventions

Every application follows the same Bronze/Silver/Gold pattern established for the Oracle migration, but each new app still needs explicit decisions:

- **Shared conventions, independent instances.** Each app gets its own Bronze → Silver → Gold, but naming, schema-tagging, and folder structure should be standardized org-wide so any engineer can navigate any app's Lakehouse the same way.
- **Cross-domain sharing via OneLake shortcuts**, not copies. If another application needs a table from this Oracle-sourced Gold layer, shortcut into it rather than duplicating data — keeping a single source of truth. Set policy on which layers are shortcut-able across domains (typically Gold only, never Bronze).
- **Document ownership of each medallion layer per app** — data engineering typically owns Bronze→Silver, data analytics owns Silver→Gold→semantic model — so retention and access decisions below have a clear owner.

### Retention and backup — different per application, and worth designing deliberately

This is the area most teams assume Fabric "just handles," and it only partially does:

- **OneLake soft delete is automatic but shallow** — deleted files are retained for 7 days by default before permanent removal<cite index="52-1">OneLake automatically protects your data with soft delete, which retains deleted files for seven days before permanent removal. This built-in protection helps you recover from accidental deletions or user errors.</cite> That's protection against accidental deletion, not a compliance-grade retention policy.
- **Geo-redundancy/disaster recovery is a capacity-level setting**, not per-workspace or per-app<cite index="53-1">Geo-redundancy (OneLake disaster recovery) is enabled at the Fabric capacity level, not at workspace or item level.</cite> If different applications need different DR postures, that's a capacity-design decision. It also only replicates OneLake data, asynchronously, and does not back up workspace items, pipelines, or semantic models — recovery is a manual redeploy-and-rehydrate exercise, not an automatic restore<cite index="53-1">Fabric protects data automatically through OneLake geo-replication (capacity-level, Microsoft-managed, no performance impact). This only safeguards the data, it does NOT replicate workspaces, pipelines, or semantic models. So DR in Fabric = Redeploy + Rehydrate, not restore.</cite>
- **Code/metadata backup is Git, not a storage feature.** Pipelines, notebooks, and semantic model definitions need Git integration as their backup and version history mechanism<cite index="57-1">Microsoft recommends leveraging Delta Lake versioning and OneLake redundancy for data resilience, using Git integration or REST API-based exports for code and metadata, and implementing cross-region replication or Azure Backup for disaster recovery.</cite> — non-negotiable per app, since it's also the DR runbook.
- **Fabric SQL Database has native point-in-time restore** (7 days by default) if any app uses that item type — the one item type with a built-in backup feature, unlike Lakehouse/Warehouse.
- **For compliance-driven retention that differs per application** (e.g., a regulated finance app needing 7-year retention vs. an internal dashboard needing 90 days), build custom retention tiers on top of the platform — typically scheduled notebooks that snapshot/archive Delta table versions on a defined cadence<cite index="55-1">Define retention tiers—daily, weekly, monthly, and yearly—to balance compliance requirements and storage efficiency... Utilize Fabric notebooks or scheduled Spark jobs to automate backup.</cite>
- Attach a retention/backup decision matrix to every application's workspace documentation — *DR required? Retention period? Backup automation owner? Compliance driver?* — rather than leaving it as tribal knowledge.

### Security and data classification

- Model access with **Entra ID security groups mapped to domains and workspace roles**, not individual user assignments<cite index="47-1">The guidance on organizing workspaces by business domain and managing access through Microsoft Entra security groups is also very practical for maintaining governance as projects grow.</cite>
- Enforce **row-level security and sensitivity labels** at the semantic model layer so classification applies consistently regardless of which report or tool consumes the data.
- Integrate **Microsoft Purview** across every application's workspace for catalog, lineage, and sensitivity labeling from day one — retrofitting labels onto historical items after governance was skipped is expensive<cite index="44-1">lineage data for earlier work is missing, sensitivity labels were never applied to historical items, and the audit trail has gaps.</cite>
- Delegate day-to-day management to domain/workspace admins, with tenant-level settings owned centrally<cite index="46-1">Fabric admins should define tenant-wide settings, and domain admins should override delegated settings as needed. Individual teams (workspace owners) define their own more granular workspace-level controls and settings.</cite>

### Operations, cost, and CI/CD across the estate

- **Monitoring and audit** — pipeline run history, capacity utilization, and audit logs per app, surfaced centrally through the Fabric admin portal rather than per-workspace.
- **Cost attribution** — if apps share capacity, track CU consumption per workspace for chargeback and right-sizing; this is much harder to retrofit than to design in from the start.
- **CI/CD** — Git-integrated deployment pipelines with dev → test → prod promotion per app, so schema/logic changes go through the same rigor as any production system, and so the DR "redeploy" story above actually works when needed.

**Framework summary:** domain map first, one workspace per application as the default with documented exceptions, capacity/region as a deliberate isolation decision, medallion conventions standardized but instances per-app, and an explicit retention/backup matrix per application — because Fabric's built-in protections are a starting point, not a complete compliance answer across a growing estate of applications.

---

## The final target state

To close the loop: **Power BI and analytics consume Fabric only.** The Oracle database server is removed from the analytics path — not before, not tentatively — only after validation, report cutover, and formal decommission approval have all happened. Everything before that point is temporary scaffolding; everything after it is the platform your organization will run on for years. Keeping that distinction sharp, from the very first Copy job you build, is what separates a clean migration from a permanent mess dressed up as a migration.
