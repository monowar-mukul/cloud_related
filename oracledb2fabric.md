# Moving off Oracle: a solution architect's guide to migrating from a VM to Microsoft Fabric

Every migration project eventually runs into the same tension: the move itself is temporary, but the mistakes you make while moving can become permanent. That's the single most important idea behind the architecture in this article. There are two flows at play — a **migration/cutover flow** that exists only to get data from Oracle into Fabric safely, and a **steady-state target flow** that is what your analytics platform looks like forever after. Confusing the two, or building your permanent pipelines around temporary migration logic, is the most common way these projects go wrong.

This article walks through both flows: the six-step cutover sequence for taking Oracle out of the picture, and the permanent Bronze/Silver/Gold architecture that replaces it. It's written for three audiences at once — the solution architect who needs the full picture, the Oracle DBA or developer who's comfortable with databases but new to Fabric, and the non-technical stakeholder who just wants to know where their reports come from.

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

You know Oracle. You don't yet know Fabric. Here's each box in the migration band translated into concrete actions, building on the setup steps covered earlier (workspace, Lakehouse, gateway, Oracle connector, Copy job).

### Step 1 — Baseline source DB for migration
Before touching Fabric, document what you're moving: schemas, table row counts, data types (pay attention to Oracle `NUMBER`, `DATE`, `CLOB` — these are the usual type-mapping surprises), and which tables genuinely feed analytics versus pure OLTP noise you don't need to bring over. Capture baseline row counts and key aggregates (sums, min/max dates) per table — you'll compare against these at every later step. This baseline is your ground truth for the entire project.

### Step 2 — Initial full load to Fabric
This is the Copy job you'd run in **Full copy** mode: every run copies everything from Oracle into your Bronze Lakehouse landing tables. For large tables, use Oracle's built-in parallel/partitioned copy support so the load doesn't run overnight unnecessarily. Land the data as-is — don't transform anything yet. Bronze exists specifically so you have an unmodified, auditable copy of what Oracle actually contained.

### Step 3 — Incremental / final delta sync
Switch the same Copy job to **Incremental copy** mode and pick a watermark column per table (a last-modified timestamp, a sequence ID, or a `ROWVERSION`-equivalent). From here, every run only pulls rows changed since the last run — this is what lets you keep Oracle and Fabric in sync for weeks without re-copying everything each time. Run this on a schedule (hourly, every 15 minutes — whatever matches your freshness needs) throughout the parallel-run period, with Oracle continuing to take writes as normal.

This step ends with one specific run that matters more than the rest: the **final delta sync**. Once you're ready to cut over, trigger one last incremental run to pull every remaining change out of Oracle, right up to the cutover moment. This final sync is what step 4's reconciliation is checked against — Fabric should now match Oracle exactly, with nothing left behind. Remember: watermark-based incremental copy won't catch hard deletes in Oracle, so either use a soft-delete flag or schedule periodic full reconciliation for tables where deletes matter, and fold that check into the final sync too.

### Step 4 — Reconcile record counts + UAT
This is where the discipline pays off. Compare row counts, checksums, and key business metrics between Oracle and Fabric after every incremental run, ideally automated (a notebook or scheduled query that flags mismatches). In parallel, have business users run their existing Oracle-fed reports side by side with the equivalent new Fabric-fed reports (built on your Silver/Gold layers) and sign off that the numbers match. Don't skip UAT because the row counts look right — a right row count with wrong values is worse than an obvious failure.

### Step 5 — Switch reports to Fabric
Once reconciliation against the final sync and UAT both pass, repoint Power BI reports and any downstream consumers from Oracle to Fabric, one workspace or report group at a time rather than all at once. This is the actual cutover moment — it only happens after the final sync in step 3 has landed and been validated, never before. A staged cutover also limits blast radius if something was missed, and gives you a rollback path (Oracle is still running, untouched, and still holding the pre-cutover state) until you're fully confident.

### Step 6 — Decommission Azure DB server when approved
Only after every consumer has been repointed and a sign-off has been obtained do you turn Oracle off. Oracle is never stopped or made read-only before this point — it stays live and writable all the way through steps 1–5 so there's always a safe fallback. Because your permanent pipeline (Bronze → Silver → Gold) never depended on Oracle staying live — it was fed by the migration Copy jobs, not a live mirror — Fabric doesn't break when Oracle disappears. This is the payoff of keeping the two flows separate from day one.

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

None of this is optional scaffolding — it's what turns "we copied Oracle into Fabric" into "we have a governed, auditable analytics platform."

---

## The final target state

To close the loop: **Power BI and analytics consume Fabric only.** The Oracle database server is removed from the analytics path — not before, not tentatively — only after validation, report cutover, and formal decommission approval have all happened. Everything before that point is temporary scaffolding; everything after it is the platform your organization will run on for years. Keeping that distinction sharp, from the very first Copy job you build, is what separates a clean migration from a permanent mess dressed up as a migration.
