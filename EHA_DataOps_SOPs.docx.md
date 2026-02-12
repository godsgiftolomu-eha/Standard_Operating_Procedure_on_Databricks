  
![][image1]

Standard Operating Procedures

Data Engineering & DataOps

|   |
| :---- |

Databricks Lakehouse Platform on Google Cloud

Version 1.0  |  February 2026  |  Confidential

# **Document Control**

| Field | Details |
| :---- | :---- |
| Document Title | EHA Data Engineering & DataOps Standard Operating Procedures |
| Document Owner | EHA Data Platform Team |
| Classification | Internal / Confidential |
| Version | 1.0 |
| Effective Date | March 2026 |
| Review Cycle | Quarterly (or after major platform changes) |
| Approved By | \[CTO / VP Engineering\] |

## **Revision History**

| Version | Date | Author | Changes |
| :---- | :---- | :---- | :---- |
| 1.0 | Feb 2026 | Data Platform Team | Initial release covering priority SOPs |
|  |  |  |  |

# **Table of Contents**

# **1\. Introduction**

## **1.1 Purpose**

This document defines the Standard Operating Procedures (SOPs) for all data engineering and DataOps activities across the EHA Group data platform. These procedures operationalize the Enterprise Data Strategy by providing step-by-step guidance for engineers, analysts, and platform administrators working within the Databricks Lakehouse environment on Google Cloud Platform.

## **1.2 Scope**

These SOPs apply to all five Databricks workspaces within the EHA Group data platform:

* eha-core-workspace (eHA Core production)

* eha-ghm-workspace (eHA-GHM production)

* eha-clinics-workspace (eHA Clinics production)

* eha-dev-workspace (Shared development and testing)

* eha-mart-workspace (Cross-subsidiary analytics)

All data engineers, data analysts, platform administrators, and team leads responsible for data pipelines and infrastructure are expected to follow these procedures.

## **1.3 Guiding Principles**

| Principle | What It Means in Practice |
| :---- | :---- |
| Code is the source of truth | All pipeline logic, infrastructure, and configuration lives in GitHub. Nothing manual in production. |
| Dev before prod, always | Every change is developed and tested in the shared dev workspace before promotion to production. |
| Least privilege access | Engineers get only the permissions they need, enforced via Terraform-managed Unity Catalog grants. |
| Automate the repeatable | If a task is done more than twice, it should be automated and version-controlled. |
| Measure, then optimize | Observe pipeline performance and cost metrics before making optimization changes. |
| Fail loudly, recover quickly | Pipelines should alert on failure immediately. Runbooks should enable recovery within SLA. |

# **2\. Pipeline Development Lifecycle**

This SOP governs the end-to-end process for developing, testing, reviewing, and deploying data pipelines. It is the foundational procedure that all other SOPs depend on.

## **2.1 Overview**

Every pipeline change, whether a new ingestion job, a transformation update, or a bug fix, follows a consistent lifecycle that ensures code quality, traceability, and safe deployment. The lifecycle has six stages:

| Stage | Environment | Actor | Output |
| :---- | :---- | :---- | :---- |
| 1\. Plan | Asana | Engineer \+ Lead | Ticket with acceptance criteria and design notes |
| 2\. Develop | eha-dev-workspace | Engineer | Working code against dev catalog with passing unit tests |
| 3\. Review | GitHub Pull Request | Peer \+ Lead | Approved PR with all CI checks green |
| 4\. Integration Test | eha-dev-workspace (develop branch) | CI Automation | Full pipeline run against dev catalog with data validation |
| 5\. Deploy to Production | Subsidiary production workspace | CI/CD (GitHub Actions) | Deployed pipeline running against production catalog |
| 6\. Monitor | Production workspace | Engineer \+ On-call | Healthy metrics, alerts configured, SLA tracking active |

## **2.2 Procedure: New Pipeline Development**

  **Stage 1: Planning and Design**

1. Create a ticket in the project tracker (Asana) with a clear title, description, and acceptance criteria.

2. Tag the ticket with the target workspace (e.g., eha-core, eha-ghm, eha-clinics) and the medallion layer (bronze, silver, or gold).

3. For new data sources, complete the Data Source Onboarding Checklist (Section 2.6) and attach it to the ticket.

4. If the pipeline involves PII/PHI data, flag the ticket for security review and reference the Data Classification Framework from the Enterprise Data Strategy.

5. Discuss the design with the team lead. For complex pipelines, create a brief design document covering: data source(s), expected schema, transformation logic, target table(s), SLA requirements, and dependencies.

  **Stage 2: Development in Dev Workspace**

1. Create a feature branch from develop in the appropriate pipeline repository (e.g., eha-core-pipelines).

2. Open the eha-dev-workspace in the Databricks UI and connect to the feature branch via Databricks Repos.

3. Develop pipeline code using your subsidiary's dev catalog (e.g., eha\_core\_dev.bronze, eha\_core\_dev.silver).

4. Follow the Medallion Layer Standards (Section 3\) for table naming, schema design, and transformation rules.

5. Write unit tests for all transformation functions. Unit tests must be runnable without a Spark cluster (pure Python logic tests).

6. Write integration tests that validate pipeline output against expected row counts, schema conformance, and data quality checks.

7. Test the full pipeline end-to-end in the dev workspace against sample data in the dev catalog.

| Important: Dev Workspace Conduct Never write to production catalogs from the dev workspace. Dev catalogs (eha\_core\_dev, eha\_ghm\_dev, eha\_clinics\_dev) are the only write targets during development. If you need production data for testing, request a masked sample refresh from the data platform team. |
| :---- |

  **Stage 3: Code Review**

1. Push your feature branch to GitHub and open a Pull Request targeting the develop branch.

2. Fill out the PR template completely, including: summary of changes, testing evidence (screenshots or logs from dev workspace), any schema changes, and the link to the original ticket.

3. CI automatically runs on PR creation: linting (ruff/black for Python, sqlfluff for SQL), unit tests, and static analysis.

4. Request review from at least one peer engineer and the team lead. For cross-team changes (e.g., shared libraries), request review from the data platform team.

5. Reviewers check for: correctness of transformation logic, adherence to medallion standards, test coverage, naming conventions, performance considerations (partition strategy, Z-ordering), and security (no hard-coded secrets, proper data classification tagging).

6. Address all review comments. Resolve each comment thread explicitly. Do not force-push over review comments.

7. Once approved, squash-merge into develop.

  **Stage 4: Integration Testing**

1. Merging to develop triggers an automated integration test workflow in GitHub Actions.

2. The workflow deploys the pipeline to the dev workspace using the develop branch configuration and runs it against the full dev catalog dataset.

3. Data quality checks run automatically: row counts within expected range, schema matches specification, no null values in required columns, referential integrity checks pass.

4. If integration tests fail, the engineer who merged is responsible for fixing or reverting within 4 hours.

5. Integration test results are posted as a comment on the original PR and linked ticket.

  **Stage 5: Production Deployment**

1. When the develop branch is ready for release, the team lead opens a PR from develop to main.

2. The release PR requires approval from the team lead and one data platform team member.

3. Merging to main triggers the production deployment workflow in GitHub Actions.

4. The workflow deploys the pipeline to the target production workspace (e.g., eha-core-workspace) using Databricks Asset Bundles or the Databricks API.

5. The deployment workflow runs a smoke test (limited data validation) against the production catalog.

6. If the smoke test fails, the deployment is automatically rolled back and the on-call engineer is alerted.

7. The original ticket is updated with the deployment timestamp and production job URL.

  **Stage 6: Monitoring and Operations**

1. Configure alerting for the new pipeline: failure alerts (immediate), SLA breach alerts (threshold-based), and data quality alerts.

2. Add the pipeline to the team's operations dashboard with key metrics: run duration, records processed, error rate, and cost.

3. Update the pipeline catalog (a shared Confluence page or internal wiki) with: pipeline name, schedule, owner, SLA, dependencies, and recovery procedure.

4. The deploying engineer remains responsible for the pipeline for the first 48 hours post-deployment (burn-in period).

5. After the burn-in period, ownership transfers to the team's normal on-call rotation.

## **2.3 Procedure: Pipeline Bug Fix (Hotfix)**

For urgent production bugs that cannot wait for the normal release cycle:

1. Create a hotfix branch from main (not develop): hotfix/TICKET-ID-brief-description.

2. Develop and test the fix in the dev workspace against the dev catalog.

3. Open a PR targeting main directly. Mark the PR as urgent and tag the team lead.

4. Minimum one reviewer (team lead or data platform team) must approve within 2 hours.

5. After merging to main, immediately cherry-pick the commit into develop to keep branches in sync.

6. Post a brief incident note in the team channel documenting the root cause and fix.

## **2.4 Branching Strategy**

| Branch | Purpose | Deploys To | Protection Rules |
| :---- | :---- | :---- | :---- |
| main | Production-ready code | Subsidiary prod workspaces | Require 2 approvals, no force push, CI must pass |
| develop | Integration branch | Dev workspace (automated) | Require 1 approval, no force push, CI must pass |
| feature/\* | New development | Dev workspace (manual) | None (personal branch) |
| hotfix/\* | Urgent production fixes | Prod workspace (fast-track) | Require 1 approval (team lead) |
| release/\* | Release candidates (optional) | Staging if available | Require 1 approval |

## **2.5 RACI Matrix: Pipeline Development**

| Activity | Engineer | Peer Reviewer | Team Lead | Data Platform Team |
| :---- | :---- | :---- | :---- | :---- |
| Create ticket and design | R | C | A | I |
| Write pipeline code | R | I | I | I |
| Write unit and integration tests | R | I | I | I |
| Test in dev workspace | R | I | I | I |
| Code review | I | R | A | C (if cross-team) |
| Approve release PR (develop to main) | I | I | A/R | R |
| Production deployment | I | I | A | R (automation) |
| Configure monitoring and alerts | R | I | A | C |
| Incident response (first 48h) | R | I | A | C |

R \= Responsible, A \= Accountable, C \= Consulted, I \= Informed

## **2.6 Data Source Onboarding Checklist**

Before developing a pipeline for a new data source, complete this checklist and attach it to the planning ticket:

| \# | Item | Details Required | Approved By |
| :---- | :---- | :---- | :---- |
| 1 | Source system identified | System name, API/DB type, authentication method | Team Lead |
| 2 | Data classification assigned | Public / Internal / Confidential / Restricted | Security Lead |
| 3 | PII/PHI columns identified | List of columns requiring masking or tagging | Security Lead |
| 4 | Expected volume and frequency | Records per batch, batch frequency, peak volumes | Engineer |
| 5 | Schema documented | Column names, types, nullability, sample data | Engineer |
| 6 | Credential provisioned | Secret created in GCP Secret Manager, scope assigned | Data Platform Team |
| 7 | Target catalog and schema confirmed | e.g., eha\_ghm.bronze.source\_name | Team Lead |
| 8 | SLA defined | Maximum acceptable latency from source to bronze layer | Team Lead |
| 9 | Monitoring requirements | Alert channels, escalation contacts | Team Lead |
| 10 | Compliance requirements | Regulatory requirements (e.g., GDPR, HIPAA), data retention | Compliance |

# **3\. Medallion Layer Standards**

The medallion architecture (bronze, silver, gold) provides a consistent framework for progressively refining data from raw ingestion to business-ready datasets. These standards define what each layer must contain, how tables must be named, and what transformations are permitted at each stage.

## **3.1 Layer Definitions and Rules**

  **Bronze Layer: Raw Ingestion**

| Attribute | Standard |
| :---- | :---- |
| Purpose | Exact copy of source data with ingestion metadata appended. This is the system of record. |
| Schema approach | Schema-on-read. Accept the source schema as-is. No column renames, type casts, or filtering at this stage. |
| Required metadata columns | \_ingested\_at (TIMESTAMP), \_source\_file (STRING), \_batch\_id (STRING), \_source\_system (STRING) |
| Write pattern | Append-only. Never overwrite or delete records. Use Delta Lake merge only for CDC-based sources with explicit audit trails. |
| Partitioning | Partition by \_ingested\_at (date) for all batch sources. Streaming sources may use event\_date if available. |
| Data retention | Minimum 12 months. Bronze tables are the fallback for reprocessing. |
| Naming convention | {catalog}.bronze.{source\_system}\_{entity} (e.g., eha\_ghm.bronze.iot\_trek\_readings) |
| Quality expectations | None enforced. Bronze accepts all data, including malformed records. |
| Table properties | delta.autoOptimize.optimizeWrite \= true, delta.autoOptimize.autoCompact \= true |

  **Silver Layer: Cleansed and Validated**

| Attribute | Standard |
| :---- | :---- |
| Purpose | Cleansed, deduplicated, and validated data with standardized schemas and consistent data types. |
| Schema approach | Explicit schema. All columns must have defined types. Column names standardized to snake\_case. |
| Required transformations | Deduplication, null handling (replace or flag), data type casting, column renaming to standard conventions, timestamp normalization to UTC. |
| Forbidden transformations | No business aggregations, no KPI calculations, no joins across entities (those belong in gold). |
| Quality checks (mandatory) | Non-null checks on primary keys, referential integrity to dimension tables, value range checks on numeric columns, format validation on dates and identifiers. |
| Quality check enforcement | Use Delta Live Tables expectations or Great Expectations. Quarantine rows that fail checks into a {table}\_quarantine table. |
| Partitioning | Inherit from bronze or re-partition by business date if more appropriate for query patterns. |
| Naming convention | {catalog}.silver.{domain}\_{entity} (e.g., eha\_ghm.silver.devices\_trek\_readings) |
| Data classification tags | All PII/PHI columns must be tagged using Unity Catalog tags at this layer. |
| Table properties | delta.autoOptimize.optimizeWrite \= true, delta.autoOptimize.autoCompact \= true, delta.enableChangeDataFeed \= true |

  **Gold Layer: Business-Ready**

| Attribute | Standard |
| :---- | :---- |
| Purpose | Business-ready datasets optimized for analytics, reporting, and machine learning. This is the primary consumption layer. |
| Schema approach | Star schema or wide denormalized tables. Business-friendly column names. |
| Permitted transformations | Aggregations, KPI calculations, cross-entity joins, slowly-changing dimension logic, feature engineering for ML. |
| Documentation requirement | Every gold table must have: a table-level comment describing its business purpose, column-level comments for all non-obvious columns, and an entry in the data catalog. |
| Z-Order optimization | For legacy partitioned tables only. New tables should use liquid clustering (CLUSTER BY) instead, which adapts automatically. |
| Naming convention | {catalog}.gold.{domain}\_{fact|dim|agg|feature}\_{entity} (e.g., eha\_ghm.gold.monitoring\_fact\_excursions) |
| Refresh strategy | Document whether the table is incremental or full refresh. Incremental is mandatory for large tables (\>1M rows). |
| SLA | Every gold table must have a documented freshness SLA (e.g., data available within 2 hours of source update). |
| Table properties | delta.autoOptimize.optimizeWrite \= true, delta.enableChangeDataFeed \= true. For new tables: use CLUSTER BY instead of PARTITIONED BY. |
| Access control | Gold layer data is readable by mart\_engineers (for Mart workspace pull/push). Analysts access via Mart only. |

## **3.2 Naming Conventions**

| Object Type | Convention | Example |
| :---- | :---- | :---- |
| Catalog | eha\_{subsidiary} | eha\_ghm, eha\_core, eha\_clinics |
| Dev catalog | eha\_{subsidiary}\_dev | eha\_ghm\_dev |
| Schema (layer) | {layer} | bronze, silver, gold, staging, sandbox |
| Bronze table | {source\_system}\_{entity} | iot\_trek\_readings, api\_facility\_list |
| Silver table | {domain}\_{entity} | devices\_trek\_readings, facilities\_master |
| Gold fact table | {domain}\_fact\_{entity} | monitoring\_fact\_excursions |
| Gold dimension table | {domain}\_dim\_{entity} | monitoring\_dim\_facilities |
| Gold aggregate table | {domain}\_agg\_{entity} | monitoring\_agg\_daily\_temps |
| Gold feature table | {domain}\_feature\_{entity} | monitoring\_feature\_device\_health |
| Pipeline job name | {subsidiary}-{layer}-{entity} | ghm-bronze-trek-readings |
| Quarantine table | {table}\_quarantine | devices\_trek\_readings\_quarantine |

## **3.3 Schema Evolution Rules**

Delta Lake supports schema evolution, but uncontrolled changes break downstream consumers. Follow these rules:

| Change Type | Impact | Approval Required | Procedure |
| :---- | :---- | :---- | :---- |
| Add new column (nullable) | Low | Team lead | Set mergeSchema=true. Update documentation. No consumer action needed. |
| Add new column (non-null with default) | Low | Team lead | Backfill existing data with default. Update documentation. |
| Rename column | High (breaking) | Team lead \+ Data Platform team | Create new column, migrate data, deprecate old column for 30 days, then drop. |
| Change column type (widening) | Medium | Team lead | e.g., INT to BIGINT. Test downstream. Merge-safe if widening. |
| Change column type (narrowing/incompatible) | High (breaking) | Team lead \+ Data Platform team | Create new table version. Run both in parallel for 30 days. |
| Drop column | High (breaking) | Team lead \+ Data Platform team | Deprecate for 30 days (rename to \_deprecated\_{col}). Notify all downstream consumers. |
| Change partition scheme | High | Data Platform team | Requires table rebuild. Schedule during maintenance window. |

# **4\. Data Quality Management**

Data quality is not a one-time activity. It is a continuous discipline embedded in every pipeline. This SOP defines how quality rules are written, where they are enforced, how failures are handled, and who is responsible for remediation.

## **4.1 Quality Framework**

| Dimension | Definition | Where Checked | Example |
| :---- | :---- | :---- | :---- |
| Completeness | Required fields are populated | Silver layer | patient\_id must not be null |
| Uniqueness | No duplicate records for the same entity | Silver layer | Deduplicate on (device\_id, timestamp) |
| Validity | Values fall within expected ranges/formats | Silver layer | temperature\_c between \-80 and 100 |
| Timeliness | Data arrives within SLA window | Bronze \+ monitoring | Trek readings arrive within 6 hours |
| Consistency | Related datasets agree with each other | Gold layer | Facility counts match master list |
| Accuracy | Values correctly represent the real world | Gold layer (spot checks) | Aggregated temps match manual readings |

## **4.2 Quality Rule Implementation**

Quality rules are implemented as code, version-controlled alongside pipeline definitions. The preferred implementation approach depends on the pipeline type:

| Pipeline Type | Quality Framework | Where Rules Live |
| :---- | :---- | :---- |
| Delta Live Tables (DLT) | DLT Expectations (EXPECT, EXPECT OR DROP, EXPECT OR FAIL) | Inline in DLT pipeline definition |
| Databricks Workflows (non-DLT) | Great Expectations or custom validation functions | Separate validation module in pipeline repository |
| Ad-hoc/exploratory | Assert statements in notebooks | Dev workspace only (not production) |

## **4.3 Failure Handling**

When a quality check fails, the response depends on the severity level:

| Severity | Response | Impact | Example |
| :---- | :---- | :---- | :---- |
| **WARN** | Log the violation, continue processing. Record in quality metrics dashboard. | None (pipeline completes) | 5% of records have missing optional field |
| **QUARANTINE** | Move failing rows to {table}\_quarantine. Continue with clean rows. | Partial data loss (recoverable) | Temperature reading outside valid range |
| **HALT** | Stop pipeline execution. Alert on-call engineer immediately. | Pipeline blocked until resolved | Primary key column is 100% null |
| **Quarantine Table Standard** Every silver-layer table that has quality checks must have a corresponding quarantine table in the same schema. Quarantine tables have the same schema as the source table plus: \_quarantine\_reason (STRING), \_quarantine\_timestamp (TIMESTAMP), and \_quarantine\_rule\_id (STRING). Quarantine tables are reviewed weekly by the responsible engineer. |  |  |  |

## **4.4 Quality Monitoring and Reporting**

1. Every pipeline must emit quality metrics to the quality monitoring dashboard after each run: total records processed, records passed, records quarantined, records failed.

2. Weekly quality review: each subsidiary team reviews their quarantine tables and quality trends. Action items are logged as tickets.

3. Monthly quality report: the data platform team produces a cross-subsidiary quality report showing trends, SLA compliance, and top issues. This report is shared with leadership.

4. Quality SLA: 99.5% of records must pass all quality checks over a rolling 30-day window. Breaches trigger a root cause analysis.

# **5\. Development Workspace Rules of Engagement**

The shared development workspace (eha-dev-workspace) is a shared environment where all subsidiary engineers develop and test pipelines before promoting code to production. Because it is shared, clear rules are essential to prevent interference, cost overruns, and accidental data exposure.

## **5.1 Access and Authentication**

| Attribute | Standard |
| :---- | :---- |
| Who has access | All data engineers across all subsidiaries. Analysts do not have dev workspace access. |
| Authentication | Google Workspace SSO with MFA. Same identity provider as production workspaces. |
| Catalog access | Engineers can only write to their subsidiary's dev catalog (e.g., eHA-GHM engineers write to eha\_ghm\_dev only). |
| Cross-catalog reads | Engineers may read from any dev catalog for testing cross-entity logic, but must not copy data between catalogs. |
| Production catalog access | Read-only access to production gold layers for comparison testing only. Never write to production from dev. |

## **5.2 Compute Strategy (Serverless-First)**

EHA adopts a serverless-first compute strategy across all workspaces. Serverless compute eliminates idle cluster costs, removes provisioning overhead, and auto-scales to workload demand. Classic clusters are used only when serverless is not supported for a specific workload type.

| Workload Type | Compute Type | Applies To | Rationale |
| :---- | :---- | :---- | :---- |
| Pipeline jobs (bronze/silver/gold) | Serverless Jobs Compute | All workspaces (dev \+ prod) | Auto-scales at task level, zero idle cost, no startup delay |
| Interactive development | Serverless Compute for Notebooks | Dev workspace | Instant-on, scales to zero when idle, no cluster management |
| Delta Live Tables pipelines | Serverless DLT | Production workspaces | Eliminates cluster sizing for DLT; ideal for variable-volume IoT ingestion |
| Analyst / leadership queries | Serverless SQL Warehouse | Mart workspace | Auto-scales on concurrency, zero cost when idle |
| ML model training (GPU) | Classic cluster (exception) | Production workspaces | Serverless does not yet support GPU instance types |
| Custom container workloads | Classic cluster (exception) | Production workspaces | Serverless does not yet support custom Docker images |

## **5.3 Dev Workspace Compute Rules**

| Rule | Setting | Rationale |
| :---- | :---- | :---- |
| Default compute | Serverless Compute for Notebooks (interactive) and Serverless Jobs Compute (pipeline testing) | Zero idle cost, instant startup, no cluster management burden |
| Classic clusters (if needed) | Maximum 4 workers, auto-terminate after 30 minutes, 100% spot instances | Fallback only for workloads requiring custom libraries not available on serverless |
| Photon | Enabled on serverless (included automatically) | Serverless compute includes Photon at no additional configuration cost |
| Unity Catalog mode | Required on all compute types | Ensures access control is tested as it will run in production |
| Compute policies | Enforced via Terraform; engineers cannot override serverless settings or create unrestricted classic clusters | Prevent cost overruns and ensure governance compliance |
| Resource naming | Classic clusters only: {engineer\_name}-{purpose} (e.g., jsmith-custom-lib-test) | Easy identification of exception-case classic clusters |

## **5.4 Data Management in Dev**

**Sample data provisioning:** The data platform team maintains a weekly automated job that refreshes dev catalogs with a masked, sampled subset of production data. Sample data is limited to the most recent 90 days and all PII/PHI columns are masked or anonymized. Engineers should not manually copy production data into dev catalogs.

**Temporary tables:** Engineers may create temporary tables in the sandbox schema of their dev catalog (e.g., eha\_ghm\_dev.sandbox.jsmith\_test\_output). Sandbox tables are automatically purged every 7 days. If you need data to persist longer, use the staging schema and document why.

**Cleanup responsibility:** Engineers are responsible for cleaning up their own sandbox tables and notebooks. With serverless compute, idle cluster costs are eliminated, but stale data and unused notebooks still consume storage. The data platform team runs a monthly audit and will notify engineers with stale resources.

## **5.5 What Constitutes 'Ready to Promote'**

Before opening a PR to promote code from a feature branch to develop, the engineer must confirm:

| \# | Checkpoint | Evidence Required |
| :---- | :---- | :---- |
| 1 | Pipeline runs end-to-end without errors in dev workspace | Screenshot or log link showing successful run |
| 2 | Output data matches expected schema | Schema comparison (assert or manual check) |
| 3 | All unit tests pass locally | Test output log |
| 4 | Quality checks pass on dev output | Quality check summary (row counts, pass rates) |
| 5 | No hard-coded credentials or file paths | Grep for common patterns (api\_key=, password=, /home/) |
| 6 | Code follows team style guide (linting passes) | Ruff/Black output clean |
| 7 | Notebook converted to production-ready script (if applicable) | No interactive cells, all parameters externalized |
| 8 | PR template filled out completely | PR description, testing evidence, ticket link |

# **6\. Mart Workspace Publishing Procedures**

The Mart workspace (eha-mart-workspace) is the central analytics layer where data from all subsidiaries is made available to analysts, leadership, and external partners. Publishing data to the Mart is a controlled process because downstream consumers depend on stable, documented, high-quality datasets.

## **6.1 Publishing Patterns**

| Pattern | Mechanism | When to Use | Trade-offs |
| :---- | :---- | :---- | :---- |
| Pull (primary) | Cross-catalog view in Mart referencing subsidiary gold tables | Internal dashboards, ad-hoc queries, real-time reporting | No data duplication, automatic freshness. Performance depends on gold layer query speed. |
| Push (secondary) | Scheduled Databricks Workflow that materializes curated data into Mart tables | External partner reports (UNICEF, GAVI, DOS), cross-entity aggregations, heavy analytical workloads | Data duplication, slight latency. Provides stable schemas decoupled from source changes. |

## **6.2 Procedure: Publishing a New Mart Dataset (Pull)**

1. Submit a Mart Publishing Request ticket with: business justification, source gold table(s), target schema in Mart (e.g., eha\_mart.ghm\_marts), intended consumers, and refresh requirements.

2. Data Platform team reviews the request for: data classification compliance (no Restricted data in pull views without masking), performance feasibility, and naming convention adherence.

3. Engineer creates the view definition in the eha-mart-pipelines repository. Pull views follow this naming convention: {source\_subsidiary}\_{entity}\_v (e.g., ghm\_excursions\_v).

4. The view definition must include a comment describing its business purpose and source lineage.

5. Code review and deployment follow the standard Pipeline Development Lifecycle (Section 2).

6. After deployment, the data platform team grants SELECT access to the appropriate consumer groups.

7. The dataset is added to the Mart Data Catalog with documentation: name, description, source, refresh frequency, owner, and consumer groups.

## **6.3 Procedure: Publishing a New Mart Dataset (Push)**

1. Submit a Mart Publishing Request ticket (same as pull, but specify push pattern and justification for materialization).

2. Data Platform team reviews and approves the materialization strategy: incremental vs. full refresh, schedule, and storage budget.

3. Engineer develops the push job in the eha-mart-pipelines repository. Push tables follow this naming: {source\_subsidiary}\_{entity} (e.g., ghm\_excursions, cross\_entity\_facility\_summary).

4. The push job must include quality checks on the output (row count validation, schema conformance, freshness check).

5. For external partner reports, the push job must include a validation step that compares key metrics against the previous run. Deviations exceeding 10% trigger a manual review before data is published.

6. Standard code review and deployment lifecycle applies.

7. After deployment, configure alerting for job failures and SLA breaches specific to the push schedule.

## **6.4 External Partner Reporting**

Reports delivered to external partners (UNICEF, GAVI, US Department of State) carry reputational risk and must follow additional controls:

| Control | Requirement |
| :---- | :---- |
| Data validation | All external reports must pass automated validation before delivery. No manual data manipulation. |
| Approval gate | External reports require sign-off from the subsidiary lead AND the data platform team before first delivery. |
| Format and delivery | Reports are generated as structured exports (CSV, Parquet, or API) from Mart tables. No ad-hoc notebook exports. |
| Audit trail | Every report delivery is logged: timestamp, recipient, data range, record count, and approver. |
| Schema stability | External report schemas are frozen. Any schema change requires 30-day notice to the partner. |
| PII/PHI review | Every external report is reviewed for PII/PHI leakage before the first delivery. Re-reviewed quarterly. |

## **6.5 SQL Warehouse Governance**

The Mart workspace provides Serverless SQL Warehouses for analyst and leadership queries. Serverless warehouses scale to zero when idle, eliminating baseline compute costs, and auto-scale based on query concurrency:

| Setting | Value | Rationale |
| :---- | :---- | :---- |
| Warehouse type | Serverless SQL Warehouse (mandatory) | Zero idle cost, auto-scaling, no provisioning overhead |
| Auto-stop | 5 minutes (serverless default) | Rapid scale-to-zero minimizes cost during inactivity |
| Max scaling | Auto-managed by Databricks with budget cap | Serverless scales clusters automatically; budget alerts enforce ceiling |
| Query timeout | 30 minutes | Prevent runaway queries from consuming resources |
| Photon | Enabled (included with serverless) | 2-8x faster query processing at no extra configuration cost |
| Access control | Analysts: USE \+ SELECT on eha\_mart and eha\_shared catalogs only | Analysts cannot access subsidiary catalogs directly |
| Cost tagging | All warehouse usage tagged with team and purpose | Enable cost allocation and chargeback reporting |
| Result caching | Enabled (serverless default) | Repeated dashboard queries served from cache at zero compute cost |

# **7\. Operational Runbooks**

Runbooks provide step-by-step guidance for handling common operational scenarios. They are designed to be used by on-call engineers and should enable resolution without requiring deep domain expertise.

## **7.1 Runbook: Pipeline Failure Triage**

| Step | Action | Details |
| :---- | :---- | :---- |
| 1 | Acknowledge the alert | Respond to the alert in the team channel within 15 minutes. Assign yourself as the responder. |
| 2 | Check the Databricks job run | Navigate to the failed job run in the Databricks UI. Read the error message and stack trace. |
| 3 | Classify the failure | Is it: (a) transient infrastructure error, (b) data issue (unexpected schema/volume), (c) code bug, or (d) dependency failure? |
| 4a | Transient error: retry | If the error is transient (timeout, serverless capacity throttle, transient cloud error), retry the job. If it fails again, escalate. |
| 4b | Data issue: investigate source | Check the source system. Has the schema changed? Is the data volume abnormal? Contact the source team if needed. |
| 4c | Code bug: hotfix | Follow the Hotfix procedure (Section 2.3). Fix in dev, fast-track review, deploy. |
| 4d | Dependency failure: wait or workaround | If an upstream pipeline failed, wait for it to recover. If a third-party service is down, implement fallback if available. |
| 5 | Validate recovery | After fixing, verify the pipeline produces correct output. Check row counts, schema, and quality checks. |
| 6 | Communicate resolution | Post a summary in the team channel: root cause, fix applied, data impact (if any), and whether downstream consumers were affected. |
| 7 | Create follow-up ticket | If the failure exposed a systemic issue, create a ticket for a longer-term fix (e.g., add retry logic, improve error handling). |

## **7.2 Runbook: Data Incident Response**

A data incident is any event where incorrect, incomplete, or unauthorized data was published to consumers. Examples include wrong aggregation results in gold tables, PII exposure in reports, or data loss during processing.

| Step | Action | Timeline |
| :---- | :---- | :---- |
| 1 | Declare the incident and assign a severity level (P1: data exposed to external partners or PII leaked, P2: incorrect data published to internal consumers, P3: data quality degradation below SLA) | Immediately |
| 2 | Notify affected consumers. For P1: notify subsidiary lead and CTO immediately. For P2: notify team leads. For P3: notify within 24 hours. | Within 30 minutes (P1/P2) |
| 3 | Contain the impact: pause downstream pipelines consuming the affected data, disable reports if data is externally visible, revoke access if PII is exposed. | Within 1 hour |
| 4 | Investigate root cause using Unity Catalog audit logs, pipeline logs, and Delta Lake time travel (DESCRIBE HISTORY, RESTORE). | Within 4 hours |
| 5 | Remediate: fix the pipeline, reprocess affected data using Delta Lake time travel to restore correct state, rerun downstream pipelines. | Within 8 hours (P1/P2) |
| 6 | Validate: confirm corrected data matches expectations. Have a second engineer verify. | Before closing |
| 7 | Communicate resolution to all affected parties with: root cause, impact scope, remediation steps, and preventive measures. | Within 24 hours |
| 8 | Conduct a blameless post-incident review within 5 business days. Document findings and action items. | Within 5 days |

## **7.3 Runbook: Cost Spike Investigation**

1. Alert triggered: daily Databricks spend exceeds 120% of the 30-day moving average for any workspace.

2. Check the Databricks account console for the top cost contributors: which workspace, which compute type (serverless jobs, SQL warehouse, classic clusters), and which jobs.

3. For serverless job costs: check for pipeline retries (a failing job retrying indefinitely), unexpectedly large data volumes triggering excessive auto-scaling, or misconfigured pipelines processing full tables instead of incremental.

4. For SQL Warehouse costs: check for runaway queries in query history. Terminate long-running queries if appropriate. Review if a dashboard is triggering excessive refreshes.

5. For classic cluster costs (exception workloads): check for idle clusters that were not terminated. Terminate immediately and notify the owner.

6. Implement the fix (kill runaway job, fix pipeline to incremental, terminate idle cluster, optimize query) and monitor for 24 hours.

7. If the cost spike is due to legitimate growth, update the baseline and cost forecast. If not, create a ticket for a preventive control (e.g., tighter budget alert, job timeout, compute policy).

## **7.4 Runbook: New Team Member Onboarding**

| Day | Task | Owner | Completed |
| :---- | :---- | :---- | :---- |
| Day 1 | Add to Google Workspace group for subsidiary team | Team Lead | ☐ |
| Day 1 | Provision Databricks account via SCIM (automatic with Google Workspace group) | Data Platform Team (auto) | ☐ |
| Day 1 | Add to GitHub team for pipeline repository access | Team Lead | ☐ |
| Day 1 | Grant Unity Catalog permissions via Terraform (add to engineer group) | Data Platform Team | ☐ |
| Day 1 | Share this SOP document and the Enterprise Data Strategy | Team Lead | ☐ |
| Week 1 | Walkthrough of dev workspace: connecting to repos, using serverless compute, navigating dev catalog | Buddy Engineer | ☐ |
| Week 1 | Assign first task: small bug fix or documentation update to practice the full lifecycle | Team Lead | ☐ |
| Week 1 | Complete Databricks Unity Catalog fundamentals training (Databricks Academy) | New Engineer | ☐ |
| Week 2 | Shadow an on-call shift to understand monitoring and incident response | On-call Engineer | ☐ |
| Week 2 | First independent pipeline PR (feature branch, dev testing, review, merge) | New Engineer | ☐ |
| Month 1 | Added to on-call rotation | Team Lead | ☐ |

## **7.5 Runbook: Team Member Offboarding**

| Step | Action | Owner | Timeline |
| :---- | :---- | :---- | :---- |
| 1 | Remove from Google Workspace group (cascades to Databricks via SCIM) | IT / Team Lead | Day of departure |
| 2 | Remove from GitHub teams and revoke personal access tokens | Team Lead | Day of departure |
| 3 | Transfer ownership of any pipelines or tables they owned | Team Lead | Before departure |
| 4 | Rotate any secrets they had access to (GCP Secret Manager) | Data Platform Team | Within 24 hours |
| 5 | Archive or delete their personal dev workspace resources (notebooks, sandbox tables, any classic cluster configs) | Data Platform Team | Within 1 week |
| 6 | Update on-call rotation and team contact lists | Team Lead | Day of departure |
| 7 | Conduct knowledge transfer session for any undocumented tribal knowledge | Departing Engineer | Before departure |

# **8\. Governance SOPs**

## **8.1 Access Request and Approval**

All access to the data platform is managed through group memberships in Unity Catalog, provisioned via Terraform. No ad-hoc GRANT statements are permitted in production.

| Access Type | Request Process | Approver | Provisioning |
| :---- | :---- | :---- | :---- |
| New engineer (subsidiary team) | Team lead submits onboarding ticket | Team Lead \+ Data Platform Team | Data Platform team adds to Terraform group config, applies via PR |
| Cross-catalog read access | Engineer submits access request ticket with justification | Both team leads \+ Data Platform Team | Terraform GRANT added via PR |
| Mart analyst access | Manager submits request for new analyst | Subsidiary Lead \+ Data Platform Team | Added to analyst group in Terraform |
| Elevated permissions (e.g., MANAGE) | Engineer submits exception request with time-bound justification | CTO \+ Data Platform Team Lead | Time-bound Terraform grant with expiry comment and review date |
| Service account credentials | Team lead submits request with scope and purpose | Data Platform Team Lead | Secret created in GCP Secret Manager, scope created in Databricks |

## **8.2 Data Classification and Tagging**

Every table containing PII or PHI data must be classified and tagged using Unity Catalog tags at the column level. This is enforced at the silver layer and verified through automated checks.

1. When creating a new silver-layer table with PII/PHI columns, add Unity Catalog tags using ALTER TABLE ... ALTER COLUMN ... SET TAGS.

2. Use the data classification framework from the Enterprise Data Strategy: classification:public, classification:internal, classification:confidential, classification:restricted.

3. PII tags must be specific: pii:name, pii:email, pii:phone, pii:address, pii:national\_id.

4. PHI tags: phi:diagnosis, phi:treatment, phi:medical\_record.

5. The CI pipeline includes an automated check that flags any new silver/gold table containing columns matching PII patterns (name, email, phone, address, ssn, dob) that are not tagged. This check blocks the PR until resolved.

6. Quarterly audit: the data platform team runs a scan of all tables to identify untagged PII/PHI columns. Findings are reported to the subsidiary leads.

## **8.3 Audit Log Review**

| Review | Frequency | Reviewer | What to Look For |
| :---- | :---- | :---- | :---- |
| Data access audit | Weekly | Data Platform Team | Unusual access patterns: bulk exports, access to restricted tables, access outside business hours |
| Permission changes | Weekly | Data Platform Team | Any GRANT or REVOKE not initiated through Terraform (indicates manual change) |
| Failed access attempts | Daily (automated) | Automated \+ Data Platform Team | Repeated failures may indicate compromised credentials or unauthorized access attempts |
| Compute usage review | Monthly | Data Platform Team \+ Leads | Cost allocation by team, idle resources, unusual compute patterns |
| External data access | Monthly | Subsidiary Leads | Review all data shared with or accessed by external partners |

## **8.4 Secret Rotation Schedule**

| Secret Type | Rotation Frequency | Owner | Procedure |
| :---- | :---- | :---- | :---- |
| Databricks service principal tokens | Every 90 days | Data Platform Team | Generate new token, update GCP Secret Manager, update Databricks secret scope |
| GCP service account keys | Every 90 days | Data Platform Team | Generate new key in GCP IAM, update Secret Manager, delete old key after 24h |
| Database connection strings | Every 180 days | Subsidiary Team | Coordinate with source system team, update Secret Manager, test connectivity |
| API keys (third-party services) | Every 180 days or per vendor policy | Subsidiary Team | Regenerate with vendor, update Secret Manager |
| SSH keys (if applicable) | Every 365 days | Data Platform Team | Generate new key pair, update authorized\_keys, revoke old key |

# **9\. Cost Optimization**

Cost optimization is a continuous practice, not a one-time exercise. This section defines the techniques, policies, and review cadence that ensure EHA maximizes the value of every Databricks DBU spent. The serverless-first compute strategy (Section 5.2) is the foundation; this section builds on it with data-layer and process-level optimizations.

## **9.1 Cost Optimization Techniques by Impact**

| Priority | Technique | Expected Impact | Applies To | Implementation |
| :---- | :---- | :---- | :---- | :---- |
| **1** | Serverless-first compute | 30-50% reduction in compute spend | All workspaces | Use Serverless Jobs Compute, Serverless Notebooks, and Serverless SQL Warehouses as the default. Classic clusters only for GPU or custom container workloads (Section 5.2). |
| **2** | Photon acceleration | 2-8x faster job completion (net lower cost) | All production workspaces | Photon is included with serverless compute at no additional configuration. For any classic cluster exceptions, enable Photon explicitly. Monitor Photon-eligible vs. non-eligible query ratios in SQL Warehouse query history. |
| **3** | Predictive Optimization | 10-20% storage cost reduction \+ query speedup | All catalogs (prod \+ dev) | Enable Databricks Predictive Optimization at the Unity Catalog metastore level. Automatically runs OPTIMIZE (compaction) and VACUUM on Delta tables based on usage patterns. Eliminates manual maintenance job scheduling. |
| **4** | Liquid Clustering | 15-30% faster queries, reduced manual tuning | All new gold tables | Use CLUSTER BY instead of static partitioning and Z-ORDER for new gold-layer tables. Liquid clustering adapts to query patterns automatically. Migrate existing high-query gold tables to liquid clustering during optimization phase. |
| **5** | Incremental processing | 50-90% reduction in per-run compute | All silver and gold pipelines | All silver and gold pipelines must use incremental processing (Delta Lake Change Data Feed or MERGE patterns) rather than full table rebuilds. Full rebuilds are permitted only for small dimension tables or during initial backfill. |
| **6** | Result caching and materialized views | Eliminate repeated compute for dashboards | Mart workspace | Enable result caching on Serverless SQL Warehouses (on by default). For frequently-queried Mart datasets with heavy aggregation, create materialized views that recompute only when underlying data changes. |
| **7** | Intelligent data tiering | 20-40% storage cost reduction | All catalogs | Set VACUUM retention to 30 days for bronze tables (12-month Delta log history retained). Use GCS Object Lifecycle Management to move data older than 12 months to Nearline storage. Enable deletion vectors for faster deletes without full file rewrites. |
| **8** | Compute policies and budget guardrails | Prevent cost overruns | All workspaces | Terraform-managed compute policies enforce serverless-first, maximum classic cluster sizes, and auto-termination. Set workspace-level budget alerts at 80% and 100% of monthly budget. Configure per-job DBU caps for safety. |

## **9.2 Production Compute Policies**

Compute policies are enforced via Terraform and cannot be overridden by individual engineers. These policies apply to production workspaces:

| Policy | Setting | Rationale |
| :---- | :---- | :---- |
| Default job compute | Serverless Jobs Compute with Photon | Optimal cost-performance for ETL workloads |
| DLT pipeline compute | Serverless DLT (Enhanced autoscaling) | Adapts to streaming volume fluctuations without over-provisioning |
| Classic cluster (exception only) | Max 8 workers, auto-terminate 30 min, spot instances where available | Hard ceiling for GPU/custom workloads that cannot run serverless |
| Job timeout | 4 hours maximum per run (configurable per pipeline) | Prevent infinite-retry cost spirals |
| Retry policy | Maximum 2 automatic retries with exponential backoff | Balance reliability with cost; alert after 2nd failure |
| SQL Warehouse | Serverless, auto-stop 5 min, Photon enabled, result caching on | Zero idle cost, optimal query performance |
| Scheduled pipeline windows | Stagger non-dependent pipelines to avoid concurrent peak | Smooth resource consumption and reduce burst costs |

## **9.3 Delta Table Optimization Standards**

| Optimization | When to Apply | How | Frequency |
| :---- | :---- | :---- | :---- |
| Predictive Optimization | All tables (automatic) | Enabled at metastore level; Databricks auto-runs OPTIMIZE and VACUUM | Automatic (managed by Databricks) |
| Liquid Clustering | New gold tables with varied query patterns | CREATE TABLE ... CLUSTER BY (col1, col2); choose 2-4 most-filtered columns | Applied at table creation; auto-maintained |
| VACUUM | All tables (if predictive optimization not available) | VACUUM table\_name RETAIN 720 HOURS (30 days) | Weekly (automated job) |
| OPTIMIZE | Tables with many small files (if predictive optimization not available) | OPTIMIZE table\_name | Daily for high-write tables, weekly for others |
| Z-ORDER (legacy tables) | Existing gold tables not yet migrated to liquid clustering | OPTIMIZE table\_name ZORDER BY (col1, col2) | Weekly (automated job) |
| Deletion Vectors | Tables with frequent updates or deletes | ALTER TABLE ... SET TBLPROPERTIES ('delta.enableDeletionVectors' \= true) | One-time configuration |
| Column Mapping | Tables requiring schema evolution (renames, drops) | ALTER TABLE ... SET TBLPROPERTIES ('delta.columnMapping.mode' \= 'name') | One-time configuration |

## **9.4 Storage Lifecycle Policies**

| Layer | Delta Log Retention | VACUUM Retention | GCS Storage Class | Archive Policy |
| :---- | :---- | :---- | :---- | :---- |
| Bronze | 365 days | 30 days | Standard (0-12 months), Nearline (12-24 months), Coldline (24+ months) | Retain raw data for 36 months for reprocessing capability. Archive to Coldline after 24 months. |
| Silver | 90 days | 14 days | Standard | No archival; silver tables are rebuilt from bronze if needed. Retention matches bronze source. |
| Gold | 90 days | 7 days | Standard | No archival; gold tables are rebuilt from silver if needed. Frequent VACUUM since data is actively queried. |
| Mart | 90 days | 7 days | Standard | Push tables follow gold retention. Pull views have no storage footprint. |
| Sandbox (dev) | 7 days | 1 day | Standard | Auto-purge after 7 days. No long-term retention. |

## **9.5 Cost Monitoring and Review Cadence**

| Review | Frequency | Owner | Actions |
| :---- | :---- | :---- | :---- |
| Daily cost dashboard check | Daily (automated alert at 120% of baseline) | On-call engineer | Investigate spikes per Cost Spike Runbook (Section 7.3) |
| Weekly compute review | Weekly | Team leads | Review top-10 most expensive jobs per workspace. Identify optimization candidates. |
| Monthly cost allocation report | Monthly | Data platform team | Produce per-subsidiary, per-workspace cost breakdown. Share with leadership. Compare against budget. |
| Quarterly optimization sprint | Quarterly | Data platform team \+ subsidiary leads | Review top cost drivers, migrate tables to liquid clustering, consolidate underused pipelines, right-size exception classic clusters. |
| Annual capacity planning | Annually | Data platform team \+ CTO | Forecast next year's DBU consumption based on data growth trends. Negotiate committed-use discounts with Databricks. |
| **Cost Optimization Target** EHA targets a 15% year-over-year reduction in cost-per-processed-record across all workspaces. This is measured as total monthly DBU spend divided by total records processed across all production pipelines. The metric is tracked on the monthly cost allocation report and reviewed quarterly. |  |  |  |

# **Appendix A: Quick Reference Card**

Post this at your workstation or pin it in your team channel.

| I want to... | Do this |
| :---- | :---- |
| Start a new pipeline | Create ticket \> branch from develop \> code in dev workspace \> PR to develop \> release to main |
| Fix a production bug urgently | Branch from main (hotfix/\*) \> fix in dev \> PR to main (1 approval) \> cherry-pick to develop |
| Publish data to Mart | Submit Mart Publishing Request \> data platform team review \> implement in eha-mart-pipelines \> standard deploy |
| Add a new data source | Complete Data Source Onboarding Checklist (Section 2.6) \> attach to ticket \> proceed with pipeline development |
| Request access to another catalog | Submit access request ticket \> both team leads \+ data platform team approve \> Terraform PR |
| Handle a pipeline failure | Acknowledge in 15 min \> classify (transient/data/code/dependency) \> fix \> validate \> communicate |
| Tag a PII column | ALTER TABLE ... ALTER COLUMN col SET TAGS ('pii:type', 'classification:confidential') |
| Create a temp table in dev | Write to {your\_catalog}\_dev.sandbox.{your\_name}\_{purpose}. Auto-purged in 7 days. |
| Choose compute type | Default: Serverless. Classic cluster only for GPU or custom containers. See Section 5.2. |
| Optimize a gold table | Use liquid clustering (CLUSTER BY) for new tables. Enable Predictive Optimization at metastore level. |
| Report a data incident | Declare severity (P1/P2/P3) \> notify per severity matrix \> contain \> investigate \> remediate \> review |
| Investigate a cost spike | Check account console \> identify top cost driver (serverless job / SQL warehouse / classic cluster) \> fix \> monitor 24h |
| Rotate a secret | Generate new credential \> update GCP Secret Manager \> update Databricks scope \> verify \> delete old |

[image1]: <data:image/png;base64,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>