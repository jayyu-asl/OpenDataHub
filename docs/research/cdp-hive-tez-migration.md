# Research: CDP Hive and Tez outbound migration compatibility constraints

**Research date:** 2026-11-02 (environment clock).

## Summary

Public primary evidence supports investigating **logical export/materialization into a new Hive service**, while preserving selected Hive formats and re-establishing target ACID behavior; it does **not** certify attaching an arbitrary Apache Hive to a CDP metastore or copying transactional directories. Apache Hive's released implementation explicitly materializes full-ACID exports because delete and aborted-insert events require their originating table/metastore context. Exact CDP patch/build → target artifacts, transaction behavior, clients and cutover must therefore pass combination-specific certification before any migration promise. [1–8]

## Scope and evidence standard

This addresses outbound **CDP Hive/Tez → OpenDataHub's future Apache-supported family**, pursuant to [Research CDP Hive and Tez migration compatibility constraints](https://github.com/jayyu-asl/OpenDataHub/issues/15) and the canonical resolution of [Define lakehouse data architecture and interoperability contracts](https://github.com/jayyu-asl/OpenDataHub/issues/8#issuecomment-6568627452). Selected clients/Hive SQL, non-transactional managed/external ORC/Parquet/partitioned tables, and ACID update/delete are formal long-term targets. Iceberg conversion and arbitrary UDF/SerDe/CDP-extension parity are not prerequisites. Source/target families, shipping BOM, scheduler and release allocation remain unselected.

**D** = direct official documentation; **S** = direct released-tag source; **I** = researcher interpretation/proposed control; **U** = unverified runtime behavior. Confidence describes evidence, not operational readiness. Representative anchors are CDP Private Cloud Base 7.1.7/7.1.9 and Apache Hive 3.1.3/4.0.1, not exhaustive inventory or a proposed BOM. Rolling manuals provide concepts only where tagged evidence is needed. No deployment, query, customer-data inspection or CLI validation occurred. The research skill and supplied issue, resolution and three prior reports were read. Web search and `source_check` were attempted but unavailable because provider authentication was missing; direct original-source fetch/inspection remained available. Some candidate documentation paths returned 404 and are not evidence.

## Findings

### 1. Capture the distribution build, not just “Hive 3”

**Claim:** Source-family identity must include vendor runtime/build and patch provenance. **Sources:** [1–4]. **Support:** D/S; patch-equivalence conclusion I. **Confidence:** high.

| Evidence anchor | Published/build dependency values | Meaning |
| --- | --- | --- |
| CDP 7.1.7 baseline | Hive `3.1.3000.7.1.7.0-551`; Tez `0.9.1.7.1.7.0-551`; Hadoop `3.1.1.7.1.7.0-551` | A distribution-specific source anchor. |
| CDP 7.1.9 baseline | Hive `3.1.3000.7.1.9.0-387`; Tez `0.9.1.7.1.9.0-387`; Hadoop `3.1.1.7.1.9.0-387` | Same nominal Hive/Tez lineage is not proof of identical patches. |
| Apache Hive 3.1.3 tag | Hadoop `3.1.0`, Tez `0.9.1` in POM | Upstream comparison, not a CDP substitute. |
| Apache Hive 4.0.1 tag | Hadoop `3.3.6`, Tez `0.10.4` in POM | Different target-family dependency closure. |

Cloudera explains its component strings as Apache component, runtime and build identifiers; service labels such as “Hive on Tez 1.0.0” are not the engine version. **I:** never normalize `3.1.3000` to upstream `3.1.3` or infer wire/schema/ACID equivalence from similar numbers. These baseline pages do not identify a customer's service pack, cumulative hotfix or private patch. Certification intake must collect exact installed artifacts/checksums, runtime/service-pack/hotfix identifiers, effective configurations, HMS schema/database version and source patch/change evidence. Public sources inspected here do not map every vendor fix to an upstream commit. [1–4]

### 2. New-service migration, Service Adoption and Iceberg conversion are different

**Claim:** No inspected primary source certifies outbound CDP→arbitrary Apache service adoption or physical ACID state transplantation. **Sources:** [5–10], approved resolution. **Support:** S/D plus I for product boundary. **Confidence:** high for separation; outbound support unverified.

- **New-service migration:** initialize a target-owned HMS/database; recreate approved metadata and import/materialize data through a certified path. Clients subsequently use target HS2/Tez.
- **Supported Service Adoption:** continue operating an existing, explicitly accepted service and its authoritative state under a separately defined management/support contract. It is not importing table definitions into another HMS, changing binaries in place, or granting redistribution permission. No such CDP adoption combination is established here.
- **Iceberg conversion:** explicitly change table representation/commit authority and independently validate semantics. Shared HMS/storage alone performs no conversion. This remains separate from preserving Hive tables.

Cloudera's inspected migration overview is titled **“Data migration to Apache Hive”** and links migration **into** its Hive environment. It is not an outbound certification contract. Likewise an upstream schema-upgrade chain describes that upstream chain, not reversal of Cloudera import/upgrade procedures. **I:** retain direction, exact versions and operation in every support claim. [10]

### 3. HMS schema portability is not table-metadata portability

**Claim:** Catalog definitions are only part of transactional state. **Sources:** [5–7]. **Support:** S/D; migration consequence I. **Confidence:** high.

Hive's schema tool records/verifies database schema compatibility and applies ordered database-specific upgrade SQL. Hive 4.0.1's schema-info implementation explicitly maps binary `4.0.1` to schema `4.0.0`; its tagged MySQL schema includes `TXNS`, `TXN_COMPONENTS`, `COMPLETED_TXN_COMPONENTS`, `TXN_TO_WRITE_ID`, `NEXT_WRITE_ID`, locks, write-set/history and compaction tables beyond ordinary table/partition records. These names illustrate coupled state, **not an export checklist sufficient for transplantation**. [5,6]

**I:** neither copying HMS table entries nor disabling schema verification establishes semantic compatibility. Even matching a schema-version string cannot demonstrate equivalent vendor columns, constraints, transaction allocation or cleaner behavior. Before any database-copy/adoption proposal, compare exact source DDL and patches with target scripts and transaction code; validate on a consistent clone, including database vendor/driver and recovery. No tested CDP→Apache schema-conversion or downgrade procedure was found. Do not hand-edit schema version markers or transaction counters as a migration recipe. [5–7]

### 4. ACID visibility depends on transaction context, not file presence

**Claim:** A directory copy can include events that must not become visible, and omit the state needed to interpret deletes. **Sources:** [7–9]. **Support:** S; safety consequence I. **Confidence:** high.

Hive 4.0.1's `AcidInputFormat` describes sequential **table write IDs**, base/delta directories, statement IDs and record identity. `AcidUtils.getAcidState` obtains valid transaction and write-ID lists, chooses an admissible base and delta ranges, handles `delete_delta`, and checks compaction visibility transaction IDs. Aborted/open writes and obsolete directories are not equivalent to live rows. Write IDs are not globally interchangeable transaction IDs. [7,8]

Consequently, `base_*`, `delta_*` and `delete_delta_*` are not ordinary independent ORC partitions. A major compaction creates a consolidated representation but is not evidence that target counters/visibility state can be discarded. The official transaction guide says old files are removed only after old readers finish. Its pre-Hive-3 upgrade warning requires appropriate major compaction and no intervening update/delete/merge before upgrade; that **specific inbound version transition is not a general CDP outbound recipe**. [7–9]

Separate **non-transactional**, **insert-only transactional**, and **full ACID** in inventory. Upstream 3.1.3 export code distinguishes insert-only/MM context from full-ACID export; do not extend a tested full-ACID path to insert-only Parquet, streaming ingestion or vice versa. Full-ACID update/delete is the ORC-based path under investigation; ordinary Parquet preservation does not imply Parquet full-ACID parity. [8,9,11]

### 5. Logical materialization is the strongest candidate, not an already supported recipe

**Claim:** Released Apache Hive explicitly implements full-ACID export through a logical read/rewrite. **Sources:** [8,11,12]. **Support:** S/D; outbound recommendation I, runtime U. **Confidence:** high for implementation; medium for candidate applicability.

Hive 3.1.3's `UpdateDeleteSemanticAnalyzer` states that delete events and aborted inserts require their source table/metastore context. It constructs `INSERT INTO temporary_table SELECT * FROM source`, writes non-transactional temporary data, then restores transactional properties in exported metadata so import can create an ACID table. Hive 4.0.1's dedicated `AcidExportSemanticAnalyzer` retains that rewrite approach. This is stronger evidence than treating “EXPORT copies data and metadata” as raw-ACID portability. [8,11]

The import/export manual documents movement between Hive/Hadoop instances independently of metastore DBMS, table/partition creation, schema/format checks, and restrictions on existing destination contents/partitions. It does not certify every vendor/version pair. [12]

**Proposed bounded candidate:** under a declared freeze, use the actual CDP engine's supported logical export/materialization to produce visible rows; validate archive properties and target import, or explicitly recreate target DDL and insert staged rows through the target transaction engine. Preserve ORC/Parquet and partition contracts where applicable; no mandatory Iceberg conversion. For full ACID, retain or explicitly rebuild target transactional semantics and prove subsequent updates/deletes and compaction. Logical rebuilding intentionally does not preserve source write-ID identities, open transactions or historical transaction lineage. CDP's exact export implementation, archive compatibility and temporary-space/authorization requirements remain **U** until inspected/tested. Do not advertise upstream behavior as vendor support. [8,11,12]

### 6. Non-ACID preservation still has lifecycle and location gates

**Claim:** File-format readability does not establish equivalent managed/external lifecycle. **Sources:** [12,13]. **Support:** D; proposed gates I. **Confidence:** high.

For selected non-transactional ORC/Parquet, candidate paths are approved DDL/partition recreation plus immutable file transfer, or certified export/import. Capture table **and individual partition** locations, column order/types, partition values, null/default partition encoding, formats/SerDes, compression, bucket/sort metadata, ownership and relevant properties. Inventory nested/non-default locations and views separately. Validate partition discovery rather than assume directory layout recreates every HMS property.

Hive's DDL guide distinguishes managed/external locations, notes that changing a database location does not relocate existing table/partition data, and documents `external.table.purge=true` deleting external data in Hive 4+. **I:** explicitly translate source defaults and purge/strict-managed behavior; test DROP/TRUNCATE/partition removal on disposable copies. “External” alone is not protection against deletion. Filesystem ACLs, encryption keys and canonical HDFS/Ozone URIs must work from every execution/maintenance process; changing URI strings is not data migration. Storage backend changes require separate certification. [13]

### 7. Clients, SQL and execution require separate regressions

**Claim:** HS2 protocol compatibility and a running endpoint do not prove workload compatibility. **Sources:** [3,4,14–16]. **Support:** S/D; fixture design I. **Confidence:** high.

Hive's tagged Thrift protocol includes negotiated client/server versions, asynchronous operations, typed result metadata and fetch representations. HS2 documentation separately configures authentication, transport and impersonation. Test named JDBC/Beeline and selected ODBC/BI clients with exact driver artifacts: connect/reconnect, metadata discovery, prepared parameters, fetch/null handling, decimal precision, timestamps/time zones, nested types, cancellation, errors and session settings. Vendor driver redistribution is a separate question. [14,15]

SQL fixtures should compare known rows/results for casts, CHAR/VARCHAR padding, decimal rounding/overflow, nulls, joins/aggregations, date functions, dynamic partitions, insert/overwrite and applicable update/delete/merge. Compare ordered results only when ordering is specified; otherwise compare multisets. Record source session settings and adapted target settings. Plans/runtime diagnostics explain differences but do not replace results. Classify each case **unchanged**, **configuration-adapted**, **query-adapted**, **format-converted**, or **unsupported**; format conversion must be explicit, not disguised as preservation. Custom extensions are inventory/remediation items, not mandatory parity.

Tez's installation guide requires Hadoop integration, distributed libraries, `tez.lib.uris`, submission-side classpaths and matching client/distributed Tez versions; examples run DAG ApplicationMasters/containers through YARN. Tagged Hive POMs show materially different dependency families. **I:** certify the actual JVM, Hadoop/storage clients, codecs/native libraries, credentials, scratch/staging permissions, execution resources and compactor initiator/worker/cleaner roles. An HS2 container or unrelated Spark service is insufficient. Scheduling remains undecided; this evidence does not certify an alternative execution backend. [3,4,9,16]

### 8. Certification must include freeze, failure and rollback

**Claim:** Outbound support requires reproducible, source-to-target gates. **Sources:** [5–16]. **Support:** I derived from direct constraints; runtime U. **Confidence:** medium pending prototypes.

1. **Identity/provenance gate:** name exact source patch/build, target BOM, schema/database, storage and clients; close artifact/license provenance. Do not bundle CDP binaries on an Apache-label assumption; redistribution/legal policy belongs to [Define redistribution, licensing, and source-delivery boundaries](https://github.com/jayyu-asl/OpenDataHub/issues/13), and support combinations to [Define distribution support contracts and certified component combinations](https://github.com/jayyu-asl/OpenDataHub/issues/6).
2. **State gate:** inventory every table category and unsupported feature; prove source committed-row oracle and absence/treatment of open/aborted work. For physical state transfer, require a separately evidenced complete state mapping; otherwise reject that route.
3. **Copy gate:** isolate destination; compare schema, locations, partitions, counts and deterministic business-key/value checks, including negative assertions for deleted/aborted rows. Rebuild statistics deliberately where needed. Verify target writes, restart and compaction without resurrection/duplication.
4. **Cutover gate:** proposed conservative baseline fences all source writers, streaming producers, DDL and maintenance; drain/resolve transactions and readers through a documented procedure; capture final consistent data/metadata boundary. Coordinate export and cleaner activity so required files cannot disappear. Switch endpoints/permissions only after reconciliation; block stale clients from writing the old authority. No zero-downtime or multi-table snapshot guarantee is inferred.
5. **Rollback gate:** retain a consistent source HMS/database-plus-files recovery point, configurations and required keys, protected from destructive target cleanup. Rehearse restore and fencing. Before target writes, controlled return to the frozen source is a candidate; **after target writes**, endpoint reversal loses divergent changes unless reconciliation/replay is separately designed and tested. Never leave both copies authoritative/writable.
6. **Security gate:** test effective HS2/HMS/Tez/storage identities, authorization denials, token renewal and audit correlation; do not presume CDP policy plugins transfer. Recovery and identity policy belong to [Define the security, governance, and recovery baseline](https://github.com/jayyu-asl/OpenDataHub/issues/9), not this report.

## Contradictions

No directly verified competing outbound support contracts were found. Important apparent conflicts remain: generic export documentation describes data/metadata movement, whereas tagged full-ACID code materializes visible rows; these are compatible only when the implementation detail is retained. The rolling ACID manual mixes first-release restrictions (including bucketing) with later-version notes, so it is not an exact Hive 3/4 feature matrix. Nominally similar CDP/upstream versions and inward-facing Cloudera migration guidance must not be used to resolve these uncertainties by assumption. [1–4,8,9,11,12]

## Missing evidence

No selected CDP patch family/target BOM, vendor-to-upstream patch map, verified outbound ACID archive pair, supported CDP metastore adoption/conversion procedure, post-cutover change-reconciliation mechanism, or runtime/client benchmark evidence exists in this study. Artifact redistribution permission is unverified. Provider authentication limited discovery/cross-checking; direct source inspection establishes implementation constraints, not absence of all possible vendor-supported paths.

## Next steps: bounded prototypes

Authorize synthetic-data experiments only after naming one source/target pair. **P0:** full-ACID ORC with insert/update/delete, aborted/open transactions, multiple deltas and compaction: compare source visible rows → logical export → target import/rebuild; then update/delete, compact and restart. Include insert-only ORC/Parquet separately, and deliberately test a raw-copy negative control in disposable isolation. **P1:** non-ACID managed/external ORC/Parquet with multiple/custom-location partitions and destructive lifecycle checks. **P1:** named-client SQL fixture suite exercising real Tez execution. **P0 before support:** interrupted export/import, stale-writer fencing and rollback before/after target writes. Store exact artifacts/configuration, expected results, final statuses and residual incompatibilities. None of these prototypes was run here.

## Sources

**Kept — fetched public primary sources:**

1. [Cloudera Runtime 7.1.7 component versions](https://docs.cloudera.com/cdp-private-cloud-base/7.1.7/runtime-release-notes/topics/rt-pvc-runtime-component-versions.html) — vendor build anchor.
2. [Cloudera Runtime 7.1.9 component versions](https://docs.cloudera.com/cdp-private-cloud-base/7.1.9/runtime-release-notes/topics/rt-pvc-runtime-component-versions.html) — Hive/Tez/Hadoop build mapping, not service-pack inventory.
3. [Apache Hive 3.1.3 POM](https://github.com/apache/hive/blob/rel/release-3.1.3/pom.xml) — upstream dependency anchor.
4. [Apache Hive 4.0.1 POM](https://github.com/apache/hive/blob/rel/release-4.0.1/pom.xml) — target-family dependency anchor.
5. [Hive Schema Tool, rolling manual](https://hive.apache.org/docs/latest/admin/hive-schema-tool/); [Hive 4.0.1 MetaStoreSchemaInfo](https://github.com/apache/hive/blob/rel/release-4.0.1/standalone-metastore/metastore-server/src/main/java/org/apache/hadoop/hive/metastore/MetaStoreSchemaInfo.java); [tagged MySQL upgrade order](https://github.com/apache/hive/blob/rel/release-4.0.1/standalone-metastore/metastore-server/src/main/sql/mysql/upgrade.order.mysql) — schema checks and ordered upgrades.
6. [Hive 4.0.1 MySQL schema](https://github.com/apache/hive/blob/rel/release-4.0.1/standalone-metastore/metastore-server/src/main/sql/mysql/hive-schema-4.0.0.mysql.sql) — catalog and transaction-state structures.
7. [Hive 4.0.1 AcidUtils](https://github.com/apache/hive/blob/rel/release-4.0.1/ql/src/java/org/apache/hadoop/hive/ql/io/AcidUtils.java) — base/delta selection and transaction/write-ID visibility.
8. [Hive 3.1.3 UpdateDeleteSemanticAnalyzer](https://github.com/apache/hive/blob/rel/release-3.1.3/ql/src/java/org/apache/hadoop/hive/ql/parse/UpdateDeleteSemanticAnalyzer.java) — explicit ACID export rationale and logical rewrite; [Hive 4.0.1 AcidExportSemanticAnalyzer](https://github.com/apache/hive/blob/rel/release-4.0.1/ql/src/java/org/apache/hadoop/hive/ql/parse/AcidExportSemanticAnalyzer.java) — released continuation of approach.
9. [Hive ACID Transactions, rolling manual](https://hive.apache.org/docs/latest/user/hive-transactions/) — snapshot/compaction configuration and historical upgrade warning; mixed-era restrictions flagged.
10. [Cloudera 7.1.9 Data migration to Apache Hive](https://docs.cloudera.com/cdp-private-cloud-base/7.1.9/migrating-data-into-hive/topics/hive_data_migration.html) — migration direction boundary, not outbound proof.
11. [Hive 3.1.3 ExportSemanticAnalyzer](https://github.com/apache/hive/blob/rel/release-3.1.3/ql/src/java/org/apache/hadoop/hive/ql/parse/ExportSemanticAnalyzer.java) — separate insert-only/MM versus full-ACID handling.
12. [Hive Import/Export, rolling manual](https://hive.apache.org/docs/latest/language/languagemanual-importexport/) — archive semantics, DBMS independence and destination restrictions.
13. [Hive DDL, rolling manual](https://hive.apache.org/docs/latest/language/languagemanual-ddl/) — locations, formats and deletion/purge lifecycle.
14. [Hive 4.0.1 TCLIService.thrift](https://github.com/apache/hive/blob/rel/release-4.0.1/service-rpc/if/TCLIService.thrift) — client/server protocol and result types.
15. [Setting up HiveServer2, rolling manual](https://hive.apache.org/docs/latest/admin/setting-up-hiveserver2/) — endpoint authentication/transport/impersonation seams.
16. [Apache Tez installation, rolling guide](https://tez.apache.org/install.html) — real distributed runtime/classpath dependencies; [Hive 4.0.1 AcidInputFormat](https://github.com/apache/hive/blob/rel/release-4.0.1/ql/src/java/org/apache/hadoop/hive/ql/io/AcidInputFormat.java) — write-ID/record identity and reader model.

**Rejected/deprioritized:** inward-only CDP procedures as outbound support proof; failed/404 candidate pages as evidence; rolling first-release limitations as an exact current matrix; nominal component-version equality as patch equivalence; secondary migration recipes and benchmarks. Supplied prior catalog/core/services reports were context, not independent CDP migration certification.
