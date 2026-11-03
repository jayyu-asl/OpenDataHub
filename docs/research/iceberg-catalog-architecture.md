# Research: Iceberg catalog architecture for the required engine ecosystem

**Research date:** 2026-11-02 (environment clock).

## Summary

**Recommend HMS-backed Iceberg HiveCatalog as the reference candidate**, with one authoritative HMS entry per shared Iceberg table. Spark, Flink and Trino have documented HMS and REST paths. Released Hive code also provides a custom-catalog/REST route, so HiveServer2 does **not** make REST impossible; however, no inspected source certifies the complete REST/HS2 combination. REST adds an implementation, lifecycle and security boundary without an established requirement that offsets those costs.

The preference for **Hive-on-Tez creation, evolution, rewrite and snapshot-maintenance demonstrations** modestly strengthens HMS's lower-complexity case, not an absolute REST exclusion. Keep REST, especially a proxy over the **same HMS backend**, as an alternative/evolution candidate rather than a mandatory new component. This is a recommendation for review, not a product decision, BOM, security policy or certification.

## Scope and evidence standard

Context: [Research Iceberg catalog architecture for the required engine ecosystem](https://github.com/jayyu-asl/OpenDataHub/issues/14), its claim comment, existing core/services research, and mid-run human preference. The parent supplied issue/comments as a `gh` export. Only public primary sources were inspected; no deployment or interoperability tests occurred. Kafka→Flink→Iceberg→Trino remains the streaming mainline; this study does not change event-identity/duplicate-handling semantics or dataset-level lineage requirements.

**D = documentation; S = released-tag source; U = unverified combination; I = researcher inference.** Confidence concerns evidence, not deployment readiness. Anchors: Iceberg **1.10.0**, Hive **4.0.1** with Iceberg **1.4.3**, Trino **484**, Gravitino **1.0.0**, Polaris **1.0.1**, and frozen Doris 4.x-family documentation. These are comparison anchors, not latest-version assertions or a proposed shipping mixture. Search APIs and `source_check` could not retrieve passages because provider authentication was unavailable; original-source fetch/inspection supplied the evidence instead. Newer-release coverage is consequently bounded.

## Findings

### 1. Commit authority is not metadata visibility

**Claim:** HiveCatalog, REST protocol and a REST server are distinct layers. **Support: D/S; confidence: high.** Iceberg atomically advances a table's metadata state with optimistic-concurrency validation. HiveCatalog persists the metadata pointer through HMS; REST sends requirements/updates to a server, including conflict and unknown-outcome responses. The protocol is not itself a durable database or locking implementation. [1–3]

- **HMS baseline:** engines' HMS catalog implementations → one HMS table/pointer; files → selected storage.
- **REST baseline:** engines' REST clients → concrete REST service → authoritative backend; files → selected storage.

A REST service can use HiveCatalog/HMS, making REST a protocol boundary over existing authority. Conversely, registering the same metadata JSON in an independent HMS HiveCatalog and independent REST backend creates separate mutable pointers. **I:** identical UUIDs/files or periodic synchronization do not establish coherent commits. OpenMetadata remains discovery/governance metadata, not transaction authority. [1–3, 10]

### 2. Principal engine roles support either baseline

**Claim:** Catalog choice is not forced by Spark/Flink/Trino or optional Doris. **Support: D; confidence: high for configuration, not certification.** [4, 5, 9, 13]

| Role / anchor | HMS route | REST route / boundary |
| --- | --- | --- |
| Spark batch / Iceberg 1.10.0 | `SparkCatalog`, `type=hive`; `SparkSessionCatalog` can delegate non-Iceberg tables. | `SparkCatalog`, `type=rest`. Match engine/Scala/runtime artifacts; REST does not change SQL write semantics. |
| Flink streaming / Iceberg 1.10.0 | Iceberg catalog with `catalog-type=hive`; explicit Hive client dependencies. | `catalog-type=rest`. Commits still depend on checkpoints; recovery and retention constraints remain. |
| Trino queries / 484 | Iceberg connector defaults to HMS; independently implemented HMS commit path. | `iceberg.catalog.type=rest`. Server capabilities, identity and namespace configuration matter. |
| HiveServer2 / Hive 4.0.1 | Storage-handler tables directly visible with Hive integration enabled. | External HMS overlay plus named REST catalog is a released-source composition; see §3. |
| Doris optional / frozen 4.x guide | `iceberg.catalog.type=hms`. | `rest` documented for external lake access/writeback. Guide lists 4.1.5+ with SDK 1.11.0 but includes newer material: not exact-release certification. Native serving copies remain separate tables. |

HDFS and Ozone stay separately selectable/certified. **I:** every engine must understand canonical metadata/data URIs and have working FileIO/configuration/credentials. REST does not turn `hdfs://`, `ofs://`, `o3fs://` and S3 into interchangeable paths; the server may itself need storage access/libraries. [2, 4, 5, 10]

### 3. HiveServer2 can delegate to REST without a second pointer

**Claim:** Calling all Hive/REST support development-only is incorrect; calling it turnkey certified stable support is also incorrect. **Support: D/S plus U; confidence: high for code route, medium for operational feasibility.**

The released chain is explicit:

1. Iceberg 1.10.0's tagged Hive guide documents named custom catalogs and **external overlays**. With no `iceberg.catalog`, HiveCatalog is the default; `location_based_table` means HadoopTables, not REST. [6]
2. Hive **4.0.1** `Catalogs.loadCatalog` calls `CatalogUtil.buildIcebergCatalog` using `iceberg.catalog.<name>.*`. Its manifest pins Iceberg **1.4.3**, whose `CatalogUtil` supports `type=rest` and `org.apache.iceberg.rest.RESTCatalog`. This predates development HEAD. [7]
3. Hive's storage handler performs final job commits on the **HS2 side**. `HiveIcebergOutputCommitter` reloads through `Catalogs.loadTable` before append/row-delta/overwrite commits; Iceberg 1.4.3 `RESTTableOperations.commit` POSTs requirements/updates to REST. **S/I:** configured correctly, Hive's table commits therefore use REST, not an independent HMS HiveCatalog pointer. [8]

The illustrative shape is `iceberg.catalog.shared_rest.type=rest`, `iceberg.catalog.shared_rest.uri=<service>`, and an external storage-handler overlay with `iceberg.catalog=shared_rest`. This is **not a tested recipe**. Preserve compatible `db.table` identifiers; start with lower-case, single-level namespaces, not arbitrary aliases/nested namespaces. The metahook defaults identifiers from HMS database/table names; Trino disables nested namespace support by default. [7, 9]

HMS remains Hive's SQL-facing overlay catalog, but not a second authority when operations reload REST. REST tables do not automatically appear in Hive: provisioning, schema freshness and lifecycle need explicit handling. Do not replace delegation with location-based access or copied `metadata_location`. For a REST proxy over the same HMS entry, validate that entry's routing/Hive integration rather than creating duplicate entries. HiveCatalog has automatic HMS/Iceberg property synchronization that other catalogs lack; managed overlays risk accidental data deletion, and external/purge behavior still needs validation. DML requires **Tez**. Iceberg stopped publishing its Hive runtime connector from 1.8.0; swapping Hive's embedded Iceberg for 1.10.0 is not established support. [6–8]

### 3a. Requested Hive-on-Tez administrative demonstration

**Claim:** Released Hive 4.0.1 contains the requested operation surfaces, with a limited full-table rewrite rather than general Iceberg optimization. **Support: D/S; confidence: high for entry points, medium for catalog-specific feasibility.** All rows below remain **U** as complete OpenDataHub demonstrations. Use the actual Hive 4.0.1/embedded Iceberg 1.4.3 runtime, not the union of later documentation features. [6–8, 14–16]

| Operation | Real SQL / released implementation | HMS versus REST and limitations |
| --- | --- | --- |
| Create | `CREATE EXTERNAL TABLE db.t (id BIGINT) STORED BY ICEBERG`; metahook → `Catalogs.createTable`. | HMS directly registers the authoritative table. Named REST catalog can receive create through the same abstraction; custom-catalog creation requires a table location in this tag. Do not infer CTAS/CTLT parity: non-HiveCatalog restrictions exist. |
| Schema evolution | `ALTER TABLE db.t ADD COLUMNS (note STRING)`; `CHANGE COLUMN` for rename/safe promotion; metahook uses `UpdateSchema`/transaction. | Source loads selected catalog, so REST is a candidate, not categorically excluded. Validate overlay schema refresh across engines; arbitrary type changes are not supported. |
| Partition evolution | `ALTER TABLE db.t SET PARTITION SPEC (bucket(16,id))`; `IcebergTableUtil.updateSpec` → `UpdatePartitionSpec.commit`. | Catalog-routed metadata update, not rewriting old files or populating HMS partition records. Validate selected transform/type support and concurrent writers. |
| Rewrite / compaction | `OPTIMIZE TABLE db.t REWRITE DATA` maps to blocking **major** compaction; alternatively `ALTER TABLE db.t COMPACT 'major' AND WAIT`. `IcebergMajorQueryCompactor` runs `INSERT OVERWRITE TABLE db.t SELECT * FROM db.t` with `ALL_PARTITIONS` rewrite policy. | Full-table query rewrite; non-major Iceberg compaction is rejected. Requires actual compactor worker/service plus Tez/runtime/storage access. REST worker-side catalog/auth configuration and final commits are unverified. No evidence here of selective/binpack rewrite, target-size guarantees or Spark-action equivalence. |
| Snapshot expiry | `ALTER TABLE db.t EXECUTE expire_snapshots('2025-01-01 00:00:00')`; `... EXECUTE expire_snapshots RETAIN LAST 10`; handler uses Iceberg expiry APIs. | Loads selected catalog; REST candidate. Metadata maintenance is an HS2/API operation, not necessarily a Tez DAG. Retention must protect active Flink recovery snapshots. |
| Orphan cleanup | `ALTER TABLE db.t EXECUTE DELETE ORPHAN-FILES OLDER THAN ('2025-01-01 00:00:00')`; Hive-specific delete-orphan action. | Action lists through **Hadoop FileSystem**, then deletes through table FileIO: REST FileIO alone is insufficient. Verify URI/client compatibility and safe age against live jobs; deleting in-flight files can corrupt tables. |
| Rollback / current snapshot | `ALTER TABLE db.t EXECUTE ROLLBACK(123456789)`; `... EXECUTE SET_CURRENT_SNAPSHOT(123456789)`; `ManageSnapshots.commit`. | Selected-catalog APIs, not copying an HMS pointer. Snapshot/files must remain available; validate branch/ancestry rules and coordinate with streaming jobs. |

**Distinction:** the compaction request uses Hive's machinery, but its Iceberg implementation rewrites the Iceberg table via a query and snapshot commit; it is **not** native Hive ACID base/delta-file compaction, and does not itself expire old Iceberg snapshots/files. Queue acceptance or `AND WAIT` completion text is not proof of correct output: verify final compaction status, new snapshot/files and unchanged logical rows. The inspected `ALL_PARTITIONS` committer uses a delete-all plus replace-partitions transaction; this source route alone does not establish safety against concurrent streaming appends. Fence/quiesce writers for a baseline demonstration unless the exact concurrent operation has separate correctness evidence. This is a conservative proposed acceptance boundary, not a claim that a runtime data-loss test was performed. [8, 14, 15]

**I:** this preference strengthens HMS chiefly through visibility, synchronized metadata and fewer worker-side catalog/auth configurations. The source does not establish an HMS-only restriction on these Iceberg APIs. REST-over-HMS is therefore a credible experiment, but not equally evidenced end-to-end. If a desired operation fails certification, the smallest human choices are to narrow/defer it or approve a separately validated maintenance client against the **same** authority (e.g. Iceberg Java metadata APIs, or another engine's supported rewrite). Do not silently substitute Spark. No selective rewrite capability should be promised from this evidence.

### 4. Concrete REST services have different offline dependencies

**Claim:** REST implementation choice materially changes suitability and operational cost. **Support: D; cost interpretation: I; confidence: high for documented dependencies, medium for fit.**

**Gravitino 1.0.0:** dedicated REST service runs **standalone**, without mandating the full Gravitino server. Hive/JDBC are production backend candidates; memory is the default, not the durable baseline. A Hive backend proxies existing HMS authority; JDBC adds its own schema and separately supplied driver. It documents HDFS, OAuth2/HTTPS and Kerberos backend authentication. However, its protocol anchor is Iceberg **1.9** and missing endpoints include multi-table transactions, pagination, scan planning and load-table credentials. The HDFS section warns that its **Hadoop 2.10.x** build may have Hadoop 3.x compatibility problems, and HDFS writes use a specified user without proxying other HDFS users. Ozone is not established. These are certification gates, not proof every Hadoop 3 deployment fails. [10]

**Polaris 1.0.1:** distinct REST/catalog authority candidate, not an HMS facade by default. Overview lists S3/Azure/GCS and read-only external catalogs. Production guidance replaces in-memory persistence with durable JDBC/PostgreSQL and discourages FILE storage. These sources establish neither production HDFS/Ozone support nor writable HMS federation; do not assume a drop-in replacement. [11]

**I:** disconnected packaging must close server/JVM/library/driver, client, storage-client, trust and identity/token-service dependencies; operations add HA/recovery, backups, upgrades and observability. Neither candidate inherently requires public cloud access in every mode, but examples downloading images/JARs are not offline proof. HMS already needs its durable RDBMS/schema and clients. Since HMS stays in the catalog and Hive needs it, adding REST does not automatically remove those costs. A standalone proxy avoids mandating a wider governance platform, but remains another service to certify. [6, 10, 11]

### 5. Locking and authorization ownership remain explicit

**Claim:** Neither baseline supplies uniform end-user security or eliminates correctness prerequisites. **Support: D/S; confidence: high.** Iceberg 1.10.0 enables HMS locks by default; disabling them requires HIVE-26882, HIVE-28121 for MySQL/MariaDB, and coordinated compatible writers/locking behavior. Upstream warns of corruption otherwise. Trino 484 independently locks/checks the HMS pointer or uses conditional updates when locks are disabled. One engine's settings do not configure every other writer. [3, 12]

REST service/backend owns requirement validation and atomic publication; backend locks/database transactions still exist. A proxy over HMS retains HMS correctness requirements. Mixed REST/direct-HMS clients are only candidates if they reach the **same underlying authority** with compatible locking, and do not establish centralized REST authorization. [2, 3, 10, 12]

Separate client→catalog authentication, server→backend identity, engine/server→storage credentials, HS2/Trino SQL authorization, raw-file authorization and audit. OAuth/credential vending is implementation/client/storage-specific, not automatic Kerberos delegation or universal row/column protection. Direct HMS/filesystem access can bypass REST-only policy. No security baseline is selected. [9–11]

## Recommendation, alternative triggers and evolution

**I; confidence: medium:** use HMS as the lowest incremental-complexity reference candidate, including the requested Hive/Tez administration demo as a **certification target**, not a proven promise. This rests on dependency/visibility evidence, not blanket legacy compatibility. Choose REST when a named deployment requires a shared HTTP contract, specific service-side controls/credential capabilities or reduced HMS coupling for non-Hive clients **and** accepts the selected implementation's cost and validated Hive/storage path. “Future-proofing” alone is not sufficient. Gravitino-over-HMS merits an evolution experiment, not automatic selection.

Retain existing Iceberg HMS entries and validate Hive integration; ordinary Hive-format tables require separate format migration. A facade over unchanged HMS permits a staged client-conversion candidate without data copying. Moving to a different REST backend is a real authority transfer: fence writers/maintenance including Flink checkpoint commits, capture the final pointer, register only where supported, compare UUID/snapshots/locations, switch all writers and block old-catalog writes. Rollback must not leave both authorities writable. These are proposed controls, not a verified migration tool. [1–3, 6, 10]

## Contradictions and missing evidence

- Late product input makes bounded CDP Hive migration compatibility a formal requirement, including selected client/SQL, non-transactional table, and ACID update/delete paths. This Iceberg-catalog study does not establish CDP source-to-target compatibility, metastore-schema portability, or safe transfer of Hive transaction state. Preserving existing Hive formats and converting them to Iceberg are separate paths requiring their own evidence.
- Generic Hive custom-catalog documentation lists fewer types than released `CatalogUtil` implements. Explicit REST integration is a **released-source composition**, neither development-only nor a certified service pairing. [6–8]
- Gravitino's HDFS support coexists with Hadoop-version/user-proxying warnings; Polaris governance claims do not establish HDFS support. [10, 11]
- No concurrent-write, compactor, REST-authentication, overlay-refresh, offline-install or negative-authorization tests were run. Newer releases may improve alternatives. The Hive preference is feasible to investigate, not proven for the full selected stack.

## Next steps

Certify one named combination: requested Hive create/evolve/full-rewrite/maintenance, Kafka/Flink streaming writes and Trino reads against the same table UUID/snapshots. Test conflicting/ambiguous commits, checkpoint restart/fencing, retention, compactor worker configuration/status, overlay schema/drop/rename, canonical storage URIs and denied direct access. Demonstrate offline restart and catalog/files restore together; verify dataset-level lineage through the selected producers rather than infer it from catalog visibility. Add Doris only if selected. Parent owns publication and issue resolution.

## Sources

**Kept — primary evidence:**

1. Iceberg 1.10.0 [specification](https://github.com/apache/iceberg/blob/apache-iceberg-1.10.0/format/spec.md) — table state and commit model.
2. Iceberg 1.10.0 [REST OpenAPI](https://github.com/apache/iceberg/blob/apache-iceberg-1.10.0/open-api/rest-catalog-open-api.yaml) — requirements, updates, errors and protocol boundaries.
3. Iceberg 1.10.0 [HiveTableOperations](https://github.com/apache/iceberg/blob/apache-iceberg-1.10.0/hive-metastore/src/main/java/org/apache/iceberg/hive/HiveTableOperations.java) — HMS pointer publication.
4. Iceberg 1.10.0 [Spark configuration](https://github.com/apache/iceberg/blob/apache-iceberg-1.10.0/docs/docs/spark-configuration.md) — HMS/REST client paths.
5. Iceberg 1.10.0 [Flink integration](https://github.com/apache/iceberg/blob/apache-iceberg-1.10.0/docs/docs/flink.md) and [writes](https://github.com/apache/iceberg/blob/apache-iceberg-1.10.0/docs/docs/flink-writes.md) — catalogs and streaming boundary.
6. Iceberg 1.10.0 [Hive integration](https://github.com/apache/iceberg/blob/apache-iceberg-1.10.0/docs/docs/hive.md) — overlays, defaults, Tez and synchronization limits; not an exact Hive release matrix.
7. Hive 4.0.1 [Catalogs](https://github.com/apache/hive/blob/rel/release-4.0.1/iceberg/iceberg-handler/src/main/java/org/apache/iceberg/mr/Catalogs.java), [manifest](https://github.com/apache/hive/blob/rel/release-4.0.1/iceberg/pom.xml), [MetaHook](https://github.com/apache/hive/blob/rel/release-4.0.1/iceberg/iceberg-handler/src/main/java/org/apache/iceberg/mr/hive/HiveIcebergMetaHook.java); Iceberg 1.4.3 [CatalogUtil](https://github.com/apache/iceberg/blob/apache-iceberg-1.4.3/core/src/main/java/org/apache/iceberg/CatalogUtil.java) — released REST resolution, create/evolution and naming evidence.
8. Hive 4.0.1 [StorageHandler](https://github.com/apache/hive/blob/rel/release-4.0.1/iceberg/iceberg-handler/src/main/java/org/apache/iceberg/mr/hive/HiveIcebergStorageHandler.java), [OutputCommitter](https://github.com/apache/hive/blob/rel/release-4.0.1/iceberg/iceberg-handler/src/main/java/org/apache/iceberg/mr/hive/HiveIcebergOutputCommitter.java); Iceberg 1.4.3 [RESTTableOperations](https://github.com/apache/iceberg/blob/apache-iceberg-1.4.3/core/src/main/java/org/apache/iceberg/rest/RESTTableOperations.java) — HS2 commit delegation and maintenance, not certification.
9. Trino 484 [connector](https://github.com/trinodb/trino/blob/484/docs/src/main/sphinx/connector/iceberg.md) and [metastores](https://github.com/trinodb/trino/blob/484/docs/src/main/sphinx/object-storage/metastores.md) — HMS/REST configuration and identity/namespace boundaries.
10. Gravitino [1.0.0 REST service](https://gravitino.apache.org/docs/1.0.0/iceberg-rest-service/) — standalone service, backends, missing endpoints and HDFS limits.
11. Polaris [1.0.1 overview](https://polaris.apache.org/releases/1.0.1/) and [production configuration](https://polaris.apache.org/releases/1.0.1/configuring-polaris-for-production/) — dependencies and storage/external-catalog limits.
12. Iceberg 1.10.0 [configuration](https://github.com/apache/iceberg/blob/apache-iceberg-1.10.0/docs/docs/configuration.md); Trino 484 [HMS table operations](https://github.com/trinodb/trino/blob/484/plugin/trino-iceberg/src/main/java/io/trino/plugin/iceberg/catalog/hms/HiveMetastoreTableOperations.java) — locks/conditional updates and corruption warning.
13. Doris [Iceberg Catalog, website commit `12cfb3bd72049510d2e81003f71149c3abbafab2`](https://github.com/apache/doris-website/blob/12cfb3bd72049510d2e81003f71149c3abbafab2/versioned_docs/version-4.x/lakehouse/catalogs/iceberg-catalog.mdx) — optional HMS/REST surfaces; family documentation, not exact-release certification.
14. Hive 4.0.1 [AlterClauseParser grammar](https://github.com/apache/hive/blob/rel/release-4.0.1/parser/src/java/org/apache/hadoop/hive/ql/parse/AlterClauseParser.g), [execute analyzer](https://github.com/apache/hive/blob/rel/release-4.0.1/ql/src/java/org/apache/hadoop/hive/ql/ddl/table/execute/AlterTableExecuteAnalyzer.java), [compact operation](https://github.com/apache/hive/blob/rel/release-4.0.1/ql/src/java/org/apache/hadoop/hive/ql/ddl/table/storage/compact/AlterTableCompactOperation.java) — exact SQL entry points and queue/status behavior.
15. Hive 4.0.1 [IcebergCompactionService](https://github.com/apache/hive/blob/rel/release-4.0.1/iceberg/iceberg-handler/src/main/java/org/apache/iceberg/mr/hive/compaction/IcebergCompactionService.java), [IcebergMajorQueryCompactor](https://github.com/apache/hive/blob/rel/release-4.0.1/iceberg/iceberg-handler/src/main/java/org/apache/iceberg/mr/hive/compaction/IcebergMajorQueryCompactor.java), [CompactorFactory](https://github.com/apache/hive/blob/rel/release-4.0.1/ql/src/java/org/apache/hadoop/hive/ql/txn/compactor/CompactorFactory.java) — Iceberg-only major query rewrite versus native ACID paths.
16. Hive 4.0.1 [IcebergTableUtil](https://github.com/apache/hive/blob/rel/release-4.0.1/iceberg/iceberg-handler/src/main/java/org/apache/iceberg/mr/hive/IcebergTableUtil.java), [delete-orphan action](https://github.com/apache/hive/blob/rel/release-4.0.1/iceberg/iceberg-handler/src/main/java/org/apache/iceberg/mr/hive/actions/HiveIcebergDeleteOrphanFiles.java) — catalog-routed partition/snapshot APIs and Hadoop filesystem listing.

**Rejected/deprioritized:** Hive master/rolling guides as released REST proof; rendered Iceberg examples with moving variables; broad interoperability/governance slogans as commit-coherence evidence. No secondary sources used.
