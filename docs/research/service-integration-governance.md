# Optional service integration and governance coverage

**Research date:** 2026-10-31, Asia/Shanghai (+08:00), from the execution environment.

**Context:** [OpenDataHub — Product Definition, Architecture, and Feature Roadmap](https://github.com/jayyu-asl/OpenDataHub/issues/1) and [Research optional service integration and governance coverage](https://github.com/jayyu-asl/OpenDataHub/issues/4), including its context comment, were read through `gh`. At research time the research ticket had label `wayfinder:research` and assignee `jayyu-asl`.

## Scope and evidence standard

This is a primary-source, documentation-only study: **no services were deployed and no interoperability or governance combination was tested**. The matrix below is an evidence matrix, not a tested compatibility matrix. “Documented” means upstream describes a surface; it does not mean OpenDataHub has certified it.

Kafka, Doris, OpenMetadata, and Airflow remain in the long-term Platform Distribution catalog, with optional inclusion in a Customer Deployment. Offline private operation is the baseline. This report does not choose release features, Kubernetes versus YARN, deployment modes, identity policy, or support versions. Only certified combinations should eventually be promised. Full lakehouse compatibility and Ranger/KMS data paths belong to the core lane; installation/management comparisons belong to the operations lane.

Version scope is deliberately explicit: Kafka 4.4 documentation/tag, Doris 4.1.5 release plus rolling 4.x-family documentation, OpenMetadata 2.0.7 release plus 2.0.x documentation, and Airflow 3.3.2. These are research anchors, not recommendations. Doris website sources are frozen at commit `12cfb3bd72049510d2e81003f71149c3abbafab2`; even that versioned directory mentions Doris 4.2, so it is not an exact 4.1.5 support contract. OpenMetadata's `latest` redirected to 2.0.x; its 2.1.x-SNAPSHOT documentation was not used as stable evidence. [D1] [M1]

## Evidence matrix

| Surface | Officially documented evidence | Boundary requiring certification |
| --- | --- | --- |
| Kafka ingestion through Spark/Flink | Spark 4.3.0 uses `spark-sql-kafka-0-10_2.13:4.3.0`. Flink 2.2 documentation specifies Kafka connector `5.0.0-2.2`, separately packaged from Flink. [K1] [K2] | Broker protocol compatibility is not JVM/Scala/client compatibility. Kafka-to-Iceberg adds a separate sink, catalog and storage path; Kafka-to-Kafka delivery guarantees do not establish that whole pipeline. |
| Kafka directly into Iceberg | Iceberg 1.12.0's tagged Kafka Connect documentation describes coordinated commits/exactly-once delivery, requiring Kafka >=2.5 through KIP-447. It has a control topic and distributions with/without HMS client dependencies. [K3] | Pin Connect worker, sink archive, catalog/FileIO and security dependencies. Supply JDBC drivers separately when needed; validate recovery and control-topic permissions. |
| Doris external lake access | Iceberg Catalog documents direct lake reads and writeback, HMS/REST/JDBC/filesystem surfaces, Parquet/ORC, V1/V2 and delete-file handling. It lists Doris 4.1.5+ with Iceberg SDK 1.11.0. [D1] | Embedded SDK version does not guarantee every table-format feature. Check exact catalog/operation support, cache freshness, schema/partition evolution and files written by the selected engines. |
| Doris serving ingestion | Routine Load consumes Kafka CSV/JSON directly; the Doris Kafka Connect sink covers additional serialized/CDC paths. Spark/Flink connectors read Doris and write via Stream Load. [D2] [D3] [D4] | These populate Doris-managed tables, not automatically the original Iceberg table. Flink checkpoint-based and batch modes have different consistency conditions; a bounded Doris source is not CDC. |
| Airflow orchestration | Spark provider 6.3.4 documents `SparkSubmitOperator` and connection/classpath prerequisites. Flink provider 1.9.1's documented operator launches through Kubernetes and the Flink Kubernetes Operator. Both provider distributions require Airflow >=2.11.0. [A1] [A2] | A provider's minimum Airflow dependency is not an engine support matrix. The Flink surface does not prove a YARN orchestration path or select Kubernetes for the product. |
| OpenMetadata inventory | Kafka requires broker access and documented permissions; topic schemas additionally require Schema Registry. Doris metadata requires >=1.2.0 and profiling >=2.0.2. Iceberg tables are now surfaced through query-engine connectors, not a standalone Iceberg connector. [M4] [M5] [M6] | Inventory, schema extraction, profiling, usage and lineage are separate capabilities. Connector minimum versions do not certify current Doris or Kafka releases. |
| OpenMetadata pipeline/lineage | Airflow REST collection supports 2.x/3.x, but table lineage needs an OpenLineage producer. Flink collection uses REST; its guide explicitly names integration up to 1.19.0, followed by a forward-compatibility assertion. [M7] [M8] | Treat newer Flink support as unverified. DAG/job visibility alone proves neither dataset edges nor column lineage. |

## Dependency and runtime implications

**Kafka:** 4.4 runtime documentation lists Java 17/21/25 support and Java 11 only for a subset of modules. Kafka 4.x uses KRaft rather than ZooKeeper. Distributed Connect additionally maintains configuration, offset and status topics, plugins on workers, and a REST management surface; protecting brokers does not automatically protect that REST endpoint. Spark/Flink serializers, registry clients and sink plugins belong in the offline dependency inventory, not merely the broker package. [K4] [K5]

**Doris:** classic integrated storage-compute uses FE and BE processes, with embedded FE metadata storage. Storage-compute separation adds Meta Service, FoundationDB and shared storage; the examined guide specifies JDK 17 and FoundationDB 7.1.x. These are conditional dependencies, not a recommendation to support both modes. External Iceberg scans are distinct from storing a serving copy in Doris, even when both use shared physical infrastructure. A serving copy introduces its own freshness, deletion, retention, authorization and recovery obligations. [D5]

Connector versions move independently. Doris Flink connector **26.3.0** documents Flink 1.15–1.20 with JDK 8 and 2.0–2.3 with JDK 17; JDK 21 is experimental for listed newer 2.x versions. The Spark connector website table still ends at 26.0.0 and Spark 3.5, while a 26.1.0 release exists. Do not extrapolate Spark 4.x support from the release number. Tagged Flink 26.3.0 and Spark/Kafka 26.1.0 READMEs describe opt-in **one-way TLS**, disabled by default, without client-certificate/mutual-TLS support on their Doris connections. [D3] [D4] [D6]

**OpenMetadata:** 2.0.x bare-metal requirements name Java 21, MySQL >=8.0.42 **or** PostgreSQL >=15, and Elasticsearch 9.x **or** OpenSearch 3.x, with stated minima 9.0.0/3.4.0. Its Elasticsearch 9 client is not compatible with older servers. These alternatives remain packaging decisions. The guide retains historical Airflow examples; stronger release-specific evidence is 2.0.7's `ingestion/setup.py`, which pins the `airflow` extra to **3.3.2**. Ingestion packages must match the server release (`x.y.z.*`); managed UI ingestion also needs managed APIs, scheduler credentials and generated-DAG/config access. External Python execution and Kubernetes-native orchestration are documented alternatives, not reasons to silently mandate Airflow or Kubernetes for metadata collection. [M1] [M2] [M3]

**Airflow:** 3.3.2 documents Python 3.10–3.14, PostgreSQL 14–18 and MySQL 8.0/8.4/Innovation. SQLite is not for production; MariaDB is unsupported. CeleryExecutor conditionally adds a broker and worker dependencies. Spark binaries/JARs and provider dependencies must exist in the actual worker context. Pinning core Airflow alone does not freeze providers or their transitive libraries. Airflow schedules engine jobs; it does not supply their Iceberg catalog/storage credentials or table-transaction semantics. [A1] [A3] [A4]

## Lineage is a collection contract, not universal coverage

OpenMetadata's current Iceberg guide makes broad feature statements, but these must be read with connector-specific prerequisites. Query-log lineage requires an available log source, parseable dialect and resolvable entities; metadata ingestion must precede lineage ingestion. Profiling needs data-read permissions and can collect sensitive samples. The Kafka connector's Schema Registry prerequisite adds another service, authentication boundary and license review; broker metadata alone cannot reconstruct topic schemas or transformation lineage. [M4] [M6] [M9]

OpenMetadata's OpenLineage endpoint accepts `RunEvent` payloads. Its documented mapper needs both resolved inputs and outputs, namespace-to-service mapping and appropriate JWT/`EditLineage` permission. A successful HTTP 200 can create **zero edges**. Column lineage additionally needs matching input columns and supported facets; SQL text alone is insufficient. Missing parent catalog entities are not all auto-created. Neither protocol acceptance nor an “Include Lineage” toggle proves complete collection. [M10]

For Airflow, distinguish three roles: scheduling ingestion, being inventoried as pipelines, and producing runtime lineage. REST inventory alone does not provide table lineage; legacy backend/database extraction uses DAG inlets/outlets. The `Backend` connection reads the Airflow instance where ingestion executes, not necessarily the `hostPort` target. OpenMetadata explicitly warns that using it for an external instance can ingest the wrong Airflow. [M3] [M7]

OpenLineage's examined 1.55.0 documentation requires a Spark listener integration. Flink 1.x needs job-code changes and does **not** support Flink SQL through that implementation; Flink 2.x uses native lineage interfaces and supports SQL. This does not certify every Kafka/Iceberg/Doris connector's dataset naming or column facets against OpenMetadata. UDFs, custom operators and opaque Python tasks remain explicit coverage questions. [L1] [M10]

## Identity, authorization and audit boundaries

- **Kafka:** TLS and SASL mechanisms include Kerberos, SCRAM and OAuth bearer; authorization is pluggable and the KRaft default implementation is `StandardAuthorizer`. Topic/group/transactional-ID permissions and principal mapping must cover engine and Connect identities. This surface does not certify a particular Ranger plugin/Kafka pairing. The tagged configuration includes `kafka-authorizer.log`; required allowed/denied events, log levels and retention still need verification. [K6]
- **Doris:** LDAP supports group-to-role mapping; LDAPS requires >=4.0.5. Client-to-FE encryption is a separate configuration: LDAP support is not proof of end-to-end TLS. Ranger-Doris and per-catalog Ranger-Hive have different resource scopes; policies are not automatically translated between them. The Hive guide limits its controller to Hive/HMS catalogs and excludes underlying storage authorization. Global privilege bypass and privileged-user masking exclusions make ordinary-user negative tests essential; do not extend that guide's claims to every Iceberg Catalog. The Ranger-Doris guide links a branch-matched SNAPSHOT plugin and service definition, leaving reproducible artifact provenance/version matching to resolve. [D7]
- **OpenMetadata:** OIDC/LDAP and catalog RBAC/ABAC protect metadata/API actions, not automatically the source engine's data. Its security guide disallows simultaneous authentication mechanisms such as SSO plus Basic Auth. Ingestion bots and source credentials form additional identities; shared SSO does not establish delegated end-user enforcement. The examined “Audit Logs” page explicitly describes **Collate**: it is not sufficient proof that the same audit UI/export capability ships in the selected open-source bundle. [M11]
- **Airflow:** authentication/authorization uses one configured auth manager; FAB provides LDAP/OAuth integration. DAG authors execute arbitrary Python and must be trusted; upstream explicitly does not promise full task-level isolation. Airflow audit records in the metadata database are distinct from worker/event logs and downstream data-access audits. [A5]

Doris separately documents an audit table and per-FE audit files, with version-dependent defaults, SQL-length limits, exclusions and retention. Consequently, unified operations must not equate collected service logs, lineage and durable security audit evidence. End-user attribution, denied requests, policy changes, collector outages and tamper-resistant retention require an explicit acceptance contract. [D8]

## Licensing implications

The examined upstream Kafka 4.4.0, Doris 4.1.5, OpenMetadata 2.0.7 and Airflow 3.3.2 license files identify **Apache License 2.0**. That is not blanket clearance for a binary distribution: preserve applicable LICENSE/NOTICE/attribution, inspect bundled dependencies and keep trademark permissions separate. [P1]

Supporting services differ: OpenSearch 3.4.0 and FoundationDB 7.1.63 identify Apache 2.0; PostgreSQL uses the PostgreSQL License; Oracle describes MySQL GPLv2/commercial dual licensing. Elastic distinguishes ELv2 release distributions from AGPLv3/SSPL/ELv2 options for applicable source portions. Confluent Schema Registry 8.0.0 identifies Confluent Community licensing except client libraries. Do not infer a registry server's license from its client. Exact artifacts and redistribution model require a bill-of-materials/legal review; no alternative is selected here. [P2]

## Precise validation needs and decision implications

Future certification should collect evidence for a **named combination**, not all upstream mixtures:

1. **Artifact closure:** record component/provider/plugin versions, JVM/Python/Scala, checksums, native libraries and licenses; demonstrate installation and restart with outbound networking blocked, including certificates, identity discovery and dependency resolution.
2. **Data-path correctness:** exercise Kafka-to-selected-engine-to-Iceberg and selected Doris load paths with restart/rebalance/checkpoint failure, replay, schema change and deletes; reconcile offsets, snapshots and rows. Define exactly which claimed delivery guarantee is being accepted.
3. **Lake versus serving copy:** query the same evolved/deleted Iceberg data through Doris and selected core engines; test refresh/time travel, then separately verify loaded-copy freshness, delete propagation and recovery. Confirm actual writeback/catalog/format boundaries.
4. **Lineage fixtures:** include Kafka topic, engine job, Iceberg table and Doris copy; assert expected and deliberately unsupported table/column edges through the API. Test duplicate table names, namespace mapping, missing columns, custom tasks and HTTP-success/zero-edge cases.
5. **Security negatives:** test ordinary users and service principals for allowed/denied operations, masking, token/keytab rotation, TLS validation, policy-cache outages and unavailable identity services. Correlate source, engine, scheduler and metadata identities; test connector mTLS requirements before promising them.
6. **Audit/upgrade:** prove which events each selected open-source artifact captures and exports, with attribution, retention, redaction and outage recovery; repeat data/lineage/security fixtures after provider or connector upgrades.

These questions may change the cost and acceptance boundary of a release. They do not determine first-release scope, mandate a deployment technology, or justify removing any named component from the long-term catalog.

## Linked primary-source bibliography

All sources were retrieved on the research date. Versioned release/tag references are distinguished from rolling family documentation; no search snippets or private assessments are evidence.

- **K1:** [Spark 4.3.0 Kafka integration][K1].
- **K2:** [Flink 2.2 Kafka SQL connector][K2] (release documentation, currently marked older than latest).
- **K3:** [Iceberg 1.12.0 Kafka Connect documentation, tagged source][K3].
- **K4:** [Kafka 4.4 JVM support][K4]; [4.4 compatibility](https://kafka.apache.org/44/getting-started/compatibility/).
- **K5:** [Kafka 4.4 Connect user guide][K5].
- **K6:** [Kafka 4.4 security][K6], [ACLs](https://kafka.apache.org/44/security/authorization-and-acls/), [4.4.0 logging configuration](https://github.com/apache/kafka/blob/4.4.0/config/log4j2.yaml).
- **D1:** [Doris 4.x Iceberg Catalog, frozen website source][D1]; [4.1.5 release](https://github.com/apache/doris/releases/tag/4.1.5-release).
- **D2:** [Doris 4.x Kafka ingestion][D2].
- **D3:** [Doris Flink connector 26.3.0 README][D3]; [4.x connector guide](https://github.com/apache/doris-website/blob/12cfb3bd72049510d2e81003f71149c3abbafab2/versioned_docs/version-4.x/connection-integration/data-integration/flink-doris-connector.md).
- **D4:** [Doris 4.x Spark connector guide][D4].
- **D5:** [Doris architecture][D5]; [separated-mode dependencies](https://github.com/apache/doris-website/blob/12cfb3bd72049510d2e81003f71149c3abbafab2/versioned_docs/version-4.x/install/deploy-manually/separating-storage-compute-deploy-manually.md).
- **D6:** [Spark connector 26.1.0 README][D6]; [Kafka connector 26.1.0 README](https://github.com/apache/doris-kafka-connector/blob/26.1.0/README.md).
- **D7:** [Doris LDAP][D7], [Ranger-Doris](https://github.com/apache/doris-website/blob/12cfb3bd72049510d2e81003f71149c3abbafab2/versioned_docs/version-4.x/admin-manual/auth/authorization/ranger.md), [Ranger-Hive boundaries](https://github.com/apache/doris-website/blob/12cfb3bd72049510d2e81003f71149c3abbafab2/versioned_docs/version-4.x/admin-manual/auth/authorization/ranger-hive.md).
- **D8:** [Doris audit documentation][D8].
- **M1:** [OpenMetadata 2.0.x bare-metal requirements][M1]; [2.0.7 release](https://github.com/open-metadata/OpenMetadata/releases/tag/2.0.7-release).
- **M2:** [OpenMetadata 2.0.7 ingestion dependency manifest][M2].
- **M3:** [OpenMetadata 2.0.x UI-managed ingestion][M3].
- **M4–M8:** OpenMetadata 2.0.x [Kafka][M4], [Doris][M5], [Iceberg design][M6], [Airflow][M7], [Flink][M8].
- **M9:** [OpenMetadata 2.0.x lineage workflow][M9].
- **M10:** [OpenMetadata 2.0.x OpenLineage compatibility and limits][M10].
- **M11:** [OpenMetadata security][M11], [roles/policies](https://docs.open-metadata.org/v2.0.x/how-to-guides/admin-guide/roles-policies), [Collate-named audit guide](https://docs.open-metadata.org/v2.0.x/how-to-guides/admin-guide/audit-logs).
- **A1:** [Airflow Spark provider 6.3.4 operators][A1]; [package requirements](https://airflow.apache.org/docs/apache-airflow-providers-apache-spark/6.3.4/index.html).
- **A2:** [Airflow Flink provider 1.9.1 operators, tagged source][A2]; [package requirements](https://airflow.apache.org/docs/apache-airflow-providers-apache-flink/1.9.1/index.html).
- **A3:** [Airflow 3.3.2 prerequisites][A3].
- **A4:** [Airflow CeleryExecutor][A4] (rolling provider documentation).
- **A5:** [Airflow 3.3.2 security model][A5], [auth manager](https://airflow.apache.org/docs/apache-airflow/3.3.2/core-concepts/auth-manager/index.html), [FAB identity integration](https://airflow.apache.org/docs/apache-airflow-providers-fab/stable/auth-manager/webserver-authentication.html) (rolling provider documentation), [audit](https://airflow.apache.org/docs/apache-airflow/3.3.2/security/audit_logs.html).
- **L1:** [OpenLineage Spark integration][L1]; [Flink integration](https://openlineage.io/docs/integrations/flink/) (rolling documentation displaying 1.55.0, not development “Next”).
- **P1:** Tagged licenses: [Kafka](https://github.com/apache/kafka/blob/4.4.0/LICENSE), [Doris](https://github.com/apache/doris/blob/4.1.5-release/LICENSE.txt), [OpenMetadata](https://github.com/open-metadata/OpenMetadata/blob/2.0.7-release/LICENSE), [Airflow](https://github.com/apache/airflow/blob/3.3.2/LICENSE).
- **P2:** Owner license sources: [OpenSearch 3.4.0](https://github.com/opensearch-project/OpenSearch/blob/3.4.0/LICENSE.txt), [FoundationDB 7.1.63](https://github.com/apple/foundationdb/blob/7.1.63/LICENSE), [PostgreSQL](https://www.postgresql.org/about/licence/), [MySQL](https://www.mysql.com/about/legal/licensing/oem/), [Elastic licensing FAQ](https://www.elastic.co/pricing/faq/licensing), [Schema Registry 8.0.0](https://github.com/confluentinc/schema-registry/blob/v8.0.0/LICENSE).

[K1]: https://spark.apache.org/docs/4.3.0/streaming/structured-streaming-kafka-integration.html
[K2]: https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/connectors/table/kafka/
[K3]: https://github.com/apache/iceberg/blob/apache-iceberg-1.12.0/docs/docs/kafka-connect.md
[K4]: https://kafka.apache.org/44/operations/java-version/
[K5]: https://kafka.apache.org/44/kafka-connect/user-guide/
[K6]: https://kafka.apache.org/44/security/security-overview/
[D1]: https://github.com/apache/doris-website/blob/12cfb3bd72049510d2e81003f71149c3abbafab2/versioned_docs/version-4.x/lakehouse/catalogs/iceberg-catalog.mdx
[D2]: https://github.com/apache/doris-website/blob/12cfb3bd72049510d2e81003f71149c3abbafab2/versioned_docs/version-4.x/data-operate/import/data-source/kafka.md
[D3]: https://github.com/apache/doris-flink-connector/blob/26.3.0/README.md
[D4]: https://github.com/apache/doris-website/blob/12cfb3bd72049510d2e81003f71149c3abbafab2/versioned_docs/version-4.x/connection-integration/data-integration/spark-doris-connector.md
[D5]: https://github.com/apache/doris-website/blob/12cfb3bd72049510d2e81003f71149c3abbafab2/versioned_docs/version-4.x/features-architecture/system-architecture.md
[D6]: https://github.com/apache/doris-spark-connector/blob/26.1.0/README.md
[D7]: https://github.com/apache/doris-website/blob/12cfb3bd72049510d2e81003f71149c3abbafab2/versioned_docs/version-4.x/admin-manual/auth/authentication/ldap.md
[D8]: https://github.com/apache/doris-website/blob/12cfb3bd72049510d2e81003f71149c3abbafab2/versioned_docs/version-4.x/admin-manual/audit-plugin.md
[M1]: https://docs.open-metadata.org/v2.0.x/deployment/bare-metal
[M2]: https://github.com/open-metadata/OpenMetadata/blob/2.0.7-release/ingestion/setup.py
[M3]: https://docs.open-metadata.org/v2.0.x/deployment/ingestion/openmetadata
[M4]: https://docs.open-metadata.org/v2.0.x/connectors/messaging/kafka
[M5]: https://docs.open-metadata.org/v2.0.x/connectors/database/doris
[M6]: https://docs.open-metadata.org/v2.0.x/connectors/database/iceberg
[M7]: https://docs.open-metadata.org/v2.0.x/connectors/pipeline/airflow
[M8]: https://docs.open-metadata.org/v2.0.x/connectors/pipeline/flink
[M9]: https://docs.open-metadata.org/v2.0.x/connectors/ingestion/workflows/lineage
[M10]: https://docs.open-metadata.org/v2.0.x/how-to-guides/data-lineage/openlineage
[M11]: https://docs.open-metadata.org/v2.0.x/deployment/security
[A1]: https://airflow.apache.org/docs/apache-airflow-providers-apache-spark/6.3.4/operators.html
[A2]: https://github.com/apache/airflow/blob/providers-apache-flink/1.9.1/providers/apache/flink/docs/operators.rst
[A3]: https://airflow.apache.org/docs/apache-airflow/3.3.2/installation/prerequisites.html
[A4]: https://airflow.apache.org/docs/apache-airflow-providers-celery/stable/celery_executor.html
[A5]: https://airflow.apache.org/docs/apache-airflow/3.3.2/security/security_model.html
[L1]: https://openlineage.io/docs/integrations/spark/
[P1]: https://github.com/open-metadata/OpenMetadata/blob/2.0.7-release/LICENSE
[P2]: https://www.elastic.co/pricing/faq/licensing
