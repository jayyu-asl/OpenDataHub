# Lakehouse core interoperability and security constraints

**Research date:** 2026-10-31 (environment clock). **Context:** [map #1](https://github.com/jayyu-asl/OpenDataHub/issues/1), [research #3](https://github.com/jayyu-asl/OpenDataHub/issues/3); planning baseline `d9f8e3b`.

## Scope and evidence standard

This is a public-primary-source, documentation/source-manifest study, **not a tested compatibility matrix**. No services were deployed, queries executed, or benchmarks run. **D** means officially documented capability or tagged implementation evidence; **L** means an explicit upstream limitation; **E** means a combination still requiring an experiment. D does not imply verified operation in OpenDataHub.

Version anchors below are examples for discovering constraints, not a release BOM or a recommendation to ship older releases. They include Iceberg 1.10.0, Hive 4.0.1, Ozone 2.0.0 (archived documentation), Hadoop 3.4.2, Spark 3.5.6, Flink 1.20, Trino 484, and Ranger 2.9.0. Hive `/latest` and Ranger's website are rolling material; Ranger's site identifies itself as `3.0.0-SNAPSHOT`. Importantly, Iceberg's nominal `/docs/1.10.0/` Spark/Flink pages displayed interpolated **1.12.0 / Flink 2.3** examples when retrieved. Version-sensitive findings therefore use the **1.10.0 source tag**, not those rendered examples. [S1, S2, S3, S15]

The Platform Distribution retains every named component in its long-term catalog; Customer Deployments may select capabilities. Only certified combinations should become support promises. Kerberos, transparent data encryption (TDE), client compatibility requirements, scheduler choice, and release boundaries remain human decisions. Offline packaging/management and services-lane integration specifics are outside this report.

## 1. Catalog and storage paths: evidence matrix

HMS is a metadata service with an RDBMS backend; it can run without the Hive query engine. Iceberg is a table format plus catalog/client implementations: **HiveCatalog uses HMS**, but HMS is not synonymous with every Iceberg catalog. Neither role should be collapsed into OpenMetadata's separately scoped metadata-management integration. HMS documentation calls for schema-version verification and explicit schema migration, and recommends an external RDBMS for durable multi-user use. [S4]

| Access path | Evidence and constraints | Planning consequence |
| --- | --- | --- |
| Spark/Flink -> Iceberg HiveCatalog -> HMS; files -> HDFS | **D:** Iceberg documents both engines using HiveCatalog and HDFS warehouse locations. HiveCatalog integration names Hive 2.3.10 or 3.1.3 or later. This is not a blanket Hive execution-engine compatibility statement. [S1, S2, S3] | A documented shared-catalog candidate, **E** for the selected client/server/security combination. |
| Trino Iceberg connector -> HMS; files -> HDFS | **D:** Trino lists HMS as its default catalog and supports HDFS 2.x/3.x when enabled. Coordinator/workers require storage connectivity. It does not route SQL through HS2. [S5, S6] | Certify Trino's own connector, identities and storage configuration independently of Hive. |
| Iceberg HadoopCatalog -> filesystem | **L:** Tagged source requires **atomic rename**; Trino's documented catalog choices do not include HadoopCatalog. [S5, S7] | A Spark/Flink path-based example is not an all-engine catalog contract. Do not mix independent catalog commit authorities over the same files. |
| Hive/Spark/Flink -> Ozone HCFS | **D:** Ozone provides `ofs://` across volumes/buckets and bucket-scoped `o3fs://`, with filesystem JAR/classpath configuration. Ozone's Hive guide specifically documents Hive 4.0+, `ofs`, `o3fs`, and `s3a`. **E:** complete Iceberg/engine combinations are not established by HCFS compatibility alone. [S8, S9] | Distribute/configure the matching Ozone client on every relevant process, including execution containers. |
| Ozone layout shared by HCFS and S3 | **L:** `OBJECT_STORE` buckets reject HCFS access. `FILE_SYSTEM_OPTIMIZED` supports both interfaces, but may reject/normalize irregular S3 keys; FSO documents atomic directory rename. [S10, S11] | Bucket layout is an interoperability boundary, not a transparent deployment detail. Validate Iceberg file-level commit behavior separately. |
| Trino -> native S3 -> Ozone S3 Gateway | **D:** Ozone exposes a separate S3 Gateway, configurable endpoints and path-style addressing. **L:** Trino says only AWS S3 and MinIO are upstream-tested. [S12, S13] | **E**, not certified Ozone support. Current HDFS documentation also does not establish direct `ofs`/`o3fs` support in Trino. |
| Iceberg REST catalog -> shared files | **D:** Flink and Trino document REST catalog options; Iceberg catalogs are distinct from file I/O. [S2, S5] | An alternative to evaluate, not an already selected additional service or a substitute for storage authorization. |

Iceberg metadata contains file locations. A deployment that exposes one bucket through both S3 and HCFS must still prove that **every engine can interpret and access the recorded URIs**, not merely reach the same physical bytes. Establish one catalog identity and canonical location convention before certifying mixed-engine writes. This is an architectural implication of the documented catalog and file-access separation, not a verified cross-protocol result. [S5, S8, S10]

## 2. Multi-engine correctness is operation-specific

Iceberg commits atomically replace a table's metadata pointer; readers retain a consistent snapshot, while concurrent writers use optimistic concurrency with retry validation. Incompatible changes may fail rather than merge automatically. These are **table-scoped** guarantees, not evidence for arbitrary multi-table transactions or identical engine SQL behavior. Iceberg identifier fields also do not guarantee row uniqueness. [S14]

**HMS locking is a correctness prerequisite, not a tuning toggle.** Iceberg 1.10.0 documents HMS locks enabled by default and heartbeat/check intervals constrained by metastore transaction timeout. Disabling locks requires HIVE-26882, HIVE-28121 when using MySQL/MariaDB, and coordinated Iceberg/version/locking behavior across the other HiveCatalog writers. Upstream explicitly warns of table corruption otherwise. These Iceberg settings must not be presumed to configure Trino's independently implemented connector. [S16]

The shared functionality envelope must be narrower than the union of engine features:

- **Spark:** row-level `MERGE`, `UPDATE`, and row-level `DELETE` require Iceberg Spark extensions. Static versus dynamic overwrite semantics matter; partition evolution can change what dynamic overwrite replaces. [S17]
- **Flink:** batch/streaming reads and writes are documented, but SQL column/partition alterations are not supported in the examined integration. Streaming `INSERT OVERWRITE` is unsupported. Upsert requires v2 tables and keys/identifier fields; partition source columns must be included in equality fields. The sink documents exactly-once semantics, but Iceberg commits follow successful checkpoints: successful checkpoints alone do not prove a successful table commit. Snapshot expiry/orphan deletion can corrupt active job state if retention ignores Flink's last committed snapshot and temporary files. Its streaming reader describes discovering **append-only** snapshots; do not advertise that as general cross-engine CDC. [S2, S18]
- **Hive:** the Iceberg integration documents storage-handler reads/writes, but **DML only with Tez**. Hive CTAS creates the table before the query's final data commit, allowing a transient empty table. [S1]
- **Trino 484:** supports v1/v2 tables; v3 is **experimental**. Row deletes on v2 produce position-delete files; encrypted Iceberg-table writes are unsupported in the examined connector. That last limitation concerns Iceberg table encryption, not a blanket prohibition on HDFS/Ozone TDE. [S5]

Consequently, append, overwrite, merge, equality/position deletes, compaction, schema/partition evolution, and snapshot maintenance require separate acceptance cases. Include timestamp/time-zone, decimal and nested-type round trips: engines have different type mappings, not an unconditional SQL type equivalence. [S1, S2, S5, S17]

## 3. HS2 execution and runtime coupling

**HiveServer2 is a Thrift SQL/session endpoint, not the complete execution stack.** Its JDBC/ODBC interface must be accompanied by Hive planning/execution libraries, HMS plus its database/schema, warehouse and scratch access, and an actual execution backend. For the documented Hive 4.x Iceberg DML path that backend is **Tez**. Hive's installation guide configures Tez libraries, `tez.lib.uris`, Hadoop classpaths and running YARN; Tez's own guide requires compatible Hadoop dependencies. Merely deploying an HS2 container, or having an unrelated Spark service, does not satisfy those prerequisites. This dependency must inform the still-open Kubernetes-versus-YARN decision without silently deciding it. [S1, S19, S20]

There is material embedded-version coupling:

| Version anchor | Actual constraint, not a full support promise |
| --- | --- |
| Hive 4.0.1 | Tagged manifests pin Hadoop 3.3.6, Tez 0.10.4 and Iceberg 1.4.3, including Hive-patched Iceberg modules. Iceberg says it stopped releasing the Hive runtime connector starting with 1.8.0; its Hive 2.x/3.x guidance points to the 1.6.1 runtime. Arbitrarily replacing Hive's bundled Iceberg with 1.10.0 is not established support. [S1, S21] |
| Iceberg 1.10.0 | Build anchors include Spark 3.4.4/3.5.6/4.0.0 and Flink 1.19.2/1.20.1/2.0.0. Runtime artifacts must match engine versions; Spark's Scala binary version also matters. Build declarations are evidence of coupling, not proof for every downstream mixture. [S3, S22] |
| Spark 3.5.6 / Flink 1.20 | Spark documents Java 8/11/17 and Scala 2.12/2.13, requiring matching Scala builds. Flink's version-line docs recommend Java 11, label Java 17 experimental, and list the Hive connector as untested on both. This does not itself prove Iceberg HiveCatalog fails, but prevents claiming the Hive-connector example validates it. [S22, S23] |
| Trino 484 / Hadoop 3.4.2 | Trino requires Java 25, minimum 25.0.1; Java 8/11/17/21/24 do not work. Hadoop 3.4.2 warns about certain JDK8/Protobuf combinations and recommends a newer JDK. A single platform-wide JVM assumption is unsafe. [S24, S25] |

Certify server runtime, client libraries, shaded/transitive dependencies, connector artifacts, HMS schema, and native/security libraries separately. A wire-compatible server does not establish classpath compatibility inside an engine.

## 4. Security enforcement boundaries

| Boundary | Documented controls and gaps |
| --- | --- |
| Authentication and delegation | Hadoop secure mode uses Kerberos; Ozone secure mode documents Kerberos plus delegation/block tokens, while S3 clients use SigV4 access IDs/secrets. HS2 separately offers Kerberos, LDAP and other modes; its documented default `NONE` is not authenticated enterprise identity. HS2 `doAs`, Trino HDFS impersonation, and Flink's process-wide Hadoop login determine the identity storage actually sees. Flink documents credential renewal differences for keytabs versus ticket caches. [S19, S26, S27, S28, S6] |
| Transport protection | Kerberos authentication is not automatically encryption. Hadoop distinguishes RPC privacy, DataNode transfer protection, and web HTTPS; KMS HTTPS is configured separately. HS2 TLS/SASL QOP covers its client connection, not every downstream connection. Spark RPC/local-disk encryption and web TLS are separate controls. Flink internal TLS and REST TLS are separate; REST TLS alone does not authenticate clients by default. Ozone web-console authentication is separate and disabled by default; its tagged configuration also defaults `hdds.grpc.tls.enabled` to false and separates internal certificates from web HTTPS certificates. HMS's administration guide leaves its security recipe incomplete: validate each engine-to-HMS authentication/TLS leg explicitly. [S4, S19, S26, S29, S30, S31, S32, S37] |
| Storage encryption and keys | HDFS TDE encrypts file content client-side using encryption zones, with independent KMS key permissions; filesystem permission alone is insufficient to decrypt. Renames across encryption zones are restricted. Ozone enables TDE when creating a bucket and documents a common KMS URI with HDFS. S3 cannot create those encrypted buckets; S3 Gateway requires KMS proxy-user/decrypt permissions for its mapped users. Ranger KMS implements Hadoop KeyProvider and has a separate KMS authorizer/audit handler. None of this proves every engine's TDE, credential propagation or key-renewal path. [S10, S33, S34] |
| Authorization: SQL versus raw files | Ranger's Hive plugin runs in **HS2**; its HDFS plugin runs in **NameNode**, with documented native-permission fallback when policies do not cover access. Ozone requires ACL enforcement plus `RangerOzoneAuthorizer` in OM, using volume/bucket/key resources. Trino has its own Ranger system access control, including column masking/row filtering; its Iceberg connector must use the corresponding `SYSTEM` security mode rather than assume `ALLOW_ALL` enforces policies. Ranger 2.5+ includes the required Trino service definition. [S5, S15, S35, S36] |
| Audit | Audit is generated at participating enforcement points, not globally by an HMS entry. Trino requires audit-store access/configuration; Ranger KMS has its own result processor, and storage plugins record storage operations. A file read under a shared engine principal is not automatically a named user's SQL audit. Validate policy-refresh outages, native fallback, audit buffering/delivery and identity correlation. [S15, S34, S36] |

**Critical implication:** Spark/Flink direct Iceberg or filesystem access is not routed through HS2 merely because they share HMS. This study establishes no universal Ranger SQL/row/column enforcement for arbitrary jobs in those engines. Table/column masks at HS2 or Trino do not protect a user who can independently read the underlying unmasked files. Storage authorization, engine controls, job-submission trust and key permissions need an explicitly reviewed boundary; neither Ranger nor Ranger KMS supplies that boundary automatically. [S1, S2, S10, S15, S36]

## 5. Bounded follow-up validation and decisions

Before a release promises a combination, specify a small certification envelope and obtain evidence for:

1. **Catalog correctness:** simultaneous Spark/Flink/Trino/Hive appends and conflicting updates through one catalog; metastore lock expiry, commit timeout/unknown outcome, retry, and crash recovery without lost/duplicate rows.
2. **Storage paths:** HDFS; Ozone HCFS/FSO; Ozone S3/Trino endpoint, addressing, multipart and signing behavior; canonical URI readability across engines. Do not infer cross-protocol compatibility from one successful write.
3. **Operation intersection:** read each writer's deletes, evolved schema/partitioning and timestamps; restart Flink around checkpoint/commit boundaries; run maintenance with live writers and retained snapshots.
4. **Negative security cases:** deny the same user through HS2, Trino, direct engine jobs, HCFS and S3; check effective identities, independent KMS denials, token renewal, certificate trust and audit records. Include raw data/metadata access and shared-service-principal bypass attempts.
5. **Execution completeness:** demonstrate HS2 Iceberg DML on the chosen real backend, with execution-container classpaths and credentials, not just endpoint health or `SELECT 1`.

These are future experiments, not tests performed here. The human must decide which security/client capabilities are mandatory and which operation/backend combinations enter each functionality-based release. Unsupported combinations may remain in the long-term catalog without becoming first-release promises.

## Primary-source bibliography

All sources were retrieved on the research date. Tags/version paths identify the evidence examined; rolling guides and development-status pages are explicitly marked.

- **S1:** [Iceberg 1.10.0 Hive integration](https://github.com/apache/iceberg/blob/apache-iceberg-1.10.0/docs/docs/hive.md).
- **S2:** [Iceberg 1.10.0 Flink integration](https://github.com/apache/iceberg/blob/apache-iceberg-1.10.0/docs/docs/flink.md).
- **S3:** [Iceberg 1.10.0 dependency versions](https://github.com/apache/iceberg/blob/apache-iceberg-1.10.0/gradle/libs.versions.toml); [Spark catalog configuration](https://github.com/apache/iceberg/blob/apache-iceberg-1.10.0/docs/docs/spark-configuration.md); rendered [Spark](https://iceberg.apache.org/docs/1.10.0/spark-getting-started/) and [Flink](https://iceberg.apache.org/docs/1.10.0/flink/) guides with moving variables.
- **S4:** [Hive Metastore 3.0+ administration, rolling guide](https://hive.apache.org/docs/latest/admin/adminmanual-metastore-3-0-administration/). Its security section is incomplete; it is not a sufficient security recipe.
- **S5:** [Trino 484 Iceberg connector](https://github.com/trinodb/trino/blob/484/docs/src/main/sphinx/connector/iceberg.md).
- **S6:** [Trino 484 HDFS configuration, identities and encryption](https://github.com/trinodb/trino/blob/484/docs/src/main/sphinx/object-storage/file-system-hdfs.md).
- **S7:** [Iceberg 1.10.0 HadoopCatalog definition](https://github.com/apache/iceberg/blob/apache-iceberg-1.10.0/core/src/main/java/org/apache/iceberg/hadoop/HadoopCatalog.java).
- **S8:** [Ozone 2.0.0 OFS](https://ozone.apache.org/docs/2.0.0/interface/ofs.html).
- **S9:** [Ozone 2.0.0 Hive integration](https://ozone.apache.org/docs/2.0.0/integration/hive.html).
- **S10:** [Ozone 2.0.0 TDE, layout and S3 Gateway KMS requirements](https://ozone.apache.org/docs/2.0.0/security/securingtde.html).
- **S11:** [Ozone 2.0.0 filesystem-optimized layout](https://ozone.apache.org/docs/2.0.0/feature/prefixfso.html).
- **S12:** [Ozone 2.0.0 S3 protocol](https://ozone.apache.org/docs/2.0.0/interface/s3.html).
- **S13:** [Trino 484 S3 filesystem and upstream testing boundary](https://github.com/trinodb/trino/blob/484/docs/src/main/sphinx/object-storage/file-system-s3.md).
- **S14:** [Iceberg 1.10.0 table specification: concurrency, commits and identifiers](https://github.com/apache/iceberg/blob/apache-iceberg-1.10.0/format/spec.md).
- **S15:** [Ranger FAQ, rolling 3.0.0-SNAPSHOT website: plugin placement and HDFS fallback](https://ranger.apache.org/faq.html). Its historical component list is not used as a current exhaustive support list.
- **S16:** [Iceberg 1.10.0 configuration: HMS locking prerequisites](https://github.com/apache/iceberg/blob/apache-iceberg-1.10.0/docs/docs/configuration.md).
- **S17:** [Iceberg 1.10.0 Spark writes](https://github.com/apache/iceberg/blob/apache-iceberg-1.10.0/docs/docs/spark-writes.md).
- **S18:** [Iceberg 1.10.0 Flink writes](https://github.com/apache/iceberg/blob/apache-iceberg-1.10.0/docs/docs/flink-writes.md); [Flink queries](https://github.com/apache/iceberg/blob/apache-iceberg-1.10.0/docs/docs/flink-queries.md).
- **S19:** [HiveServer2 setup/security, rolling guide](https://hive.apache.org/docs/latest/admin/setting-up-hiveserver2/).
- **S20:** [Hive manual installation, rolling guide](https://hive.apache.org/docs/latest/admin/manual-installation/); [Tez installation, unversioned guide with development examples](https://tez.apache.org/install.html).
- **S21:** [Hive 4.0.1 root manifest](https://github.com/apache/hive/blob/rel/release-4.0.1/pom.xml); [embedded/patched Iceberg manifest](https://github.com/apache/hive/blob/rel/release-4.0.1/iceberg/pom.xml).
- **S22:** [Spark 3.5.6 runtime prerequisites](https://spark.apache.org/docs/3.5.6/).
- **S23:** [Flink 1.20 Java compatibility, maintained release-line documentation](https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/deployment/java_compatibility/).
- **S24:** [Trino 484 deployment/runtime requirements](https://github.com/trinodb/trino/blob/484/docs/src/main/sphinx/installation/deployment.md).
- **S25:** [Hadoop 3.4.2 release overview and JDK8 warning](https://hadoop.apache.org/docs/r3.4.2/).
- **S26:** [Hadoop 3.4.2 secure mode and transport channels](https://hadoop.apache.org/docs/r3.4.2/hadoop-project-dist/hadoop-common/SecureMode.html).
- **S27:** [Ozone 2.0.0 security, tokens and SigV4](https://ozone.apache.org/docs/2.0.0/security/secureozone.html).
- **S28:** [Flink 1.20 Kerberos identity and renewal](https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/deployment/security/security-kerberos/).
- **S29:** [Hadoop 3.4.2 KMS API, SPNEGO and HTTPS](https://hadoop.apache.org/docs/r3.4.2/hadoop-kms/index.html).
- **S30:** [Spark 3.5.6 security](https://spark.apache.org/docs/3.5.6/security.html).
- **S31:** [Flink 1.20 TLS/internal and REST authentication](https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/deployment/security/security-ssl/).
- **S32:** [Ozone 2.0.0 HTTP-console authentication](https://ozone.apache.org/docs/2.0.0/security/securingozonehttp.html).
- **S33:** [Hadoop 3.4.2 transparent encryption](https://hadoop.apache.org/docs/r3.4.2/hadoop-project-dist/hadoop-hdfs/TransparentEncryption.html).
- **S34:** [Ranger 2.9.0 KeyProvider](https://github.com/apache/ranger/blob/release-ranger-2.9.0/kms/src/main/java/org/apache/hadoop/crypto/key/RangerKeyStoreProvider.java); [KMS authorizer/audit](https://github.com/apache/ranger/blob/release-ranger-2.9.0/plugin-kms/src/main/java/org/apache/ranger/authorization/kms/authorizer/RangerKmsAuthorizer.java).
- **S35:** [Ozone 2.0.0 Ranger authorizer configuration](https://ozone.apache.org/docs/2.0.0/security/securitywithranger.html); [Ranger 2.9.0 HS2 plugin configuration](https://github.com/apache/ranger/blob/release-ranger-2.9.0/hive-agent/conf/hiveserver2-site-changes.cfg).
- **S36:** [Trino 484 Ranger access control, service definitions and audit](https://github.com/trinodb/trino/blob/484/docs/src/main/sphinx/security/ranger-access-control.md).
- **S37:** [Ozone 2.0.0 tagged security and transport defaults](https://github.com/apache/ozone/blob/ozone-2.0.0/hadoop-hdds/common/src/main/resources/ozone-default.xml).
