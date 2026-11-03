# OpenDataHub Product Requirements Document

## Document status and use

**Purpose:** the consolidated PRD-level product baseline for OpenDataHub and the required guide for release/version maps.

**Status:** consolidates accepted human product decisions; implementation, actual release artifacts and operational certification remain pending. A planning decision or research report is not evidence that a capability has shipped.

**Revision:** identify this document by its Git commit when planning a release. The document revision is independent of Platform Distribution and upstream component versions. This initial consolidation introduces stable requirement identifiers; later edits preserve their identity.

**Authority:** this PRD states the consolidated requirements. The [source decisions](#source-decisions) retain their rationale, discussion and evidence. The implementation decision's scoped security/identity amendment takes precedence over conflicting original security language. The later [PRD handoff correction][handoff] replaces the earlier assumption that the total map could finish without producing this document. Other settled product choices are unchanged.

Read [GLOSSARY.md](GLOSSARY.md) for domain terms. For release planning, read this PRD, pin its revision, select the applicable requirement identifiers and use the relevant linked decisions for detail. Report a contradiction explicitly and obtain an approved amendment; do not silently choose a different product boundary. Follow [Version maps and change control](#version-maps-and-change-control) for handoff and publication gates.

### Contents

- [Product scope and users](#product-scope-and-users)
- [Required component catalog](#required-component-catalog)
- [Data architecture and interoperability](#data-architecture-and-interoperability)
- [Deployment and resource topology](#deployment-and-resource-topology)
- [Security and identity](#security-and-identity)
- [Governance and discovery](#governance-and-discovery)
- [Management and operator workflows](#management-and-operator-workflows)
- [Availability, backup and disaster recovery](#availability-backup-and-disaster-recovery)
- [Managed CDP data migration](#managed-cdp-data-migration)
- [Implementation and repository foundations](#implementation-and-repository-foundations)
- [Distribution and licensing](#distribution-and-licensing)
- [Support contract](#support-contract)
- [Reference acceptance and evidence](#reference-acceptance-and-evidence)
- [Feature-release roadmap](#feature-release-roadmap)
- [Version maps and change control](#version-maps-and-change-control)
- [Requirement allocation index](#requirement-allocation-index)
- [Source decisions](#source-decisions)

## Product scope and users

Sources: [product decision][product], [support decision][support], [release roadmap][roadmap].

**ODH-PROD-001 — Product purpose.** Deliver a reusable, privately deployed Platform Distribution combining verified component packaging with unified delivery and operations. It is not a single customer's dataset or application. Customers select dependency-complete capabilities instead of installing the entire catalog.

**ODH-PROD-002 — Customer-operated product and equal-priority personas.** The Customer Organization owns its data and daily operations. Delivery engineers establish, verify and hand over deployments; ongoing vendor-operated service is not the default. Support all four personas with equal product priority, without requiring four custom workbenches or four different people:

| Persona | Required user value |
| --- | --- |
| Platform administrator | Supported configuration, health, lifecycle, capacity and recovery operations. |
| Delivery engineer | Select capabilities, validate prerequisites, deploy/adopt services, demonstrate accepted paths and hand over an operable deployment. |
| Data engineer | Use integrated professional tools to build, schedule, observe and recover supported pipelines. |
| Analyst | Discover and understand relevant data, reach an appropriate query tool and query within the actual authorization contract. |

**ODH-PROD-003 — First-party and customer ownership.** Own the unified entry point and coherent managed experience; integrate professional development, query, orchestration, discovery and policy tools rather than recreating their editors. Customers own business rules, production pipelines and reports. Reproducible product reference jobs demonstrate integration, not arbitrary business-data quality or ownership of customer applications.

**ODH-PROD-004 — Management scope.** Manage one Customer Organization with multiple clusters and Deployment Environments. A Managed Cluster belongs to one environment. Distinguish Managed Services, connection-only resources, adoption in progress and offboarded services. Cross-customer central management and automatic environment promotion are not selected product requirements.

**ODH-PROD-005 — Selection, quality and status.** Optional describes customer activation, not experimental quality. Supported capabilities must meet production-grade contracts when selected. Keep supported/certified, experimental, deprecated, unverified and out-of-scope states explicit. Selection never omits a required dependency or implies every combination of individually supported features is certified.

### Explicit product non-goals

- A complete Cloudera Manager clone, proprietary implementation/assets/branding reuse, or Cloudera API/CSD/Parcel compatibility.
- Reimplementing upstream storage, compute, transaction, catalog, scheduling or recovery engines, or creating custom TLS/snapshot/transaction engines to hide an upstream integration gap.
- Arbitrary CDP/component/version/plugin equivalence or untested cross-release mixtures.
- Customer hardware procurement, OS installation, VM/network/disk provisioning, or execution of the background customer's log-archive project.
- Universal zero downtime, zero data loss, global transactions, automatic rollback or active-active multi-writer operation.
- Fine-grained calendar commitments, staffing estimates or a commercial service-hours/response-time SLA.

No background customer's capacity, retention, identity infrastructure, client constraints or no-DR assumption is inherited.

## Required component catalog

Source: [support decision][support], refined by [architecture][architecture], [topology][topology] and [roadmap][roadmap].

**ODH-CAT-001 — Long-term catalog and dependency completeness.** Preserve the following required capabilities in the long-term product. Catalog membership neither forces first-release delivery nor requires every customer to install it. Supported Dependencies are maintained prerequisites, not automatically independently offered products.

| Component/capability | Product role | Initial feature stage |
| --- | --- | --- |
| HDFS | Distributed filesystem and separately certified analytical/native storage paths. | R1 |
| Apache Ozone | Separately certified object/filesystem analytical storage paths. | R6 |
| Apache Iceberg | Default format for new shared persisted analytical datasets. | R1 |
| Hive Metastore (HMS) | Reference Iceberg HiveCatalog authority and selected legacy Hive metadata/transaction contracts. | R1; legacy scope R4 |
| HiveServer2/Hive and actual Tez dependencies | Legacy Hive SQL execution and bounded Hive SQL administration. | R4 |
| Apache Spark | Reference batch transformation and selected ingestion paths. | R1 |
| Apache Flink | Reference continuous processing and checkpoint/recovery integration. | R2 |
| Apache Kafka | Event transport and bounded replay input. | R2 |
| Trino | Reference interactive SQL over shared Iceberg; legacy Hive access certified separately by table type. | R1 |
| Apache Airflow | Batch/maintenance orchestration, completion and failure integration. | R1 |
| OpenMetadata | Discovery, agreed metadata and evidenced lineage. | R2 |
| Apache Ranger | Principal policy-management/audit integration surface where supported. | R1 protected profile |
| Ranger KMS | Separately certified encryption/key-provider integrations. | R3 |
| Apache Doris | Separately certified external lake queries and native Serving Copies using disaggregated deployment. | R5 |
| Resource scheduling | Actual YARN execution for selected Hive/Tez, Spark and Flink paths plus host-service resource controls. | R1; expands with paths |
| Metrics, alerting and logging | Offline-usable health, alert, log and diagnosis integration. | R1; expands with paths |
| Deployment/lifecycle management | Management foundations and complete operator loops for offered capabilities. | R1; expands with paths |

Stage labels are roadmap identifiers, not formal release numbers or dates. Refer to [Feature-release roadmap](#feature-release-roadmap) for their boundaries. Auxiliary filesystems, databases, search services, runtimes, native clients, connectors and Doris FDB/Recycler dependencies belong in each selected path's actual inventory and support responsibility.

## Data architecture and interoperability

Source: [architecture decision][architecture].

**ODH-DATA-001 — Dual analytical tracks.** Preserve selected existing Hive tables and HiveServer2/Hive-on-Tez behavior without mandatory format conversion. Use Iceberg for new datasets in the Shared Analytics Layer. Allow justified engine-specific Serving Copies, including Doris native tables; do not force every dataset through the lake or copy every dataset into Doris. Hive and Iceberg remain distinct table/operation contracts even when they share storage or HMS.

**ODH-DATA-002 — Catalog authority.** Use HMS-backed Iceberg HiveCatalog as the reference architecture. Each table has one authoritative current-state/commit authority. Different engine connectors must agree on table identity, locations and compatible commit/locking behavior. A named REST catalog/facade may be considered later for a concrete requirement, not installed for hypothetical future-proofing. Authority transfer must fence old writers and maintenance; independently writable metadata pointers are not a supported default.

**ODH-DATA-003 — Bounded interoperability.** Certify actual component/client/connector/runtime versions, storage URI/access mode, types and operation combinations. Append, overwrite, update/merge, equality/position deletes, evolution, concurrency and maintenance need distinct evidence where claimed. A common endpoint, name, HMS or bucket is not interoperability proof; the union of engine features is not a shared guarantee.

**ODH-DATA-004 — Reference data paths and ownership.** Support the following long-term paths, allocated progressively. Identify source/table/namespace owners, authorized writers, evolution/maintenance responsibility and retention policy.

```text
Exported business files -> Spark -> Iceberg/HMS -> Trino
                            ^
                         Airflow

Reproducible events -> Kafka -> Flink -> Iceberg/HMS -> Trino

Selected legacy Hive data -> HiveServer2 / Hive-on-Tez
Shared Iceberg -> Doris external queries
Declared source, including selected Spark lake loads -> Doris native Serving Copy
Reference jobs/datasets -> OpenMetadata discovery and evidenced lineage
```

Doris external/native paths are separate contracts. Direct Kafka/Flink-to-Doris may be selected separately, not required as a second copy of the streaming mainline. Airflow does not supply table transactions or Flink checkpoint semantics; OpenMetadata does not own table commits.

**ODH-DATA-005 — Consistency and recovery semantics.** The baseline is single-table commit consistency. A committed detail table with a failed aggregate remains an incomplete pipeline. No global multi-table or Iceberg/Doris transaction is promised. A successful job or Flink checkpoint does not prove a committed/queryable result. Validate ambiguous/conflicting commits, retry/restart and maintenance interaction. Retention/cleanup must protect active checkpoints, required snapshots, in-flight files and writers.

**ODH-DATA-006 — Bounded Hive SQL administration.** Provide supported table creation/evolution, selected full-table rewrite, snapshot retention/expiration, orphan cleanup and retained-snapshot recovery through admitted Hive SQL entry points, with real Tez execution when computation requires it. This is not selective/binpack optimization or native Hive ACID base/delta compaction by implication. Inspect final native status and reconcile rows/snapshots/files; coordinate writers or a validated maintenance outage. Expose unsupported target-version operations and obtain explicit deferral/alternative approval rather than calling a silent Spark substitution completion of the Hive requirement. R1 has an explicit limited-operation exception; the full offered contract begins in R4.

**ODH-DATA-007 — Storage interface contracts.** HDFS and Ozone are separately selectable and certified. Bind canonical URI/authority, namespace/layout, actual protocol/client and credentials to each path. Sharing physical storage neither creates one logical authority nor makes OFS/O3FS, S3 and HDFS interchangeable. No transparent backend switch, automatic copy/replication or mandatory dual installation is implied.

## Deployment and resource topology

Source: [topology decision][topology].

**ODH-TOPO-001 — Principal deployment route.** Run management and component services on prepared Linux bare-metal/VM hosts; use actual YARN execution for selected Hive/Tez, Spark and Flink applications. Management hosts and required foundations bootstrap/recover independently of managed HDFS/Ozone/YARN. Host-service lifecycle, management hosting and engine application scheduling are distinct responsibilities. A subset without YARN-dependent capabilities need not install YARN just to show a UI. Kubernetes-led deployment/execution is deferred.

**ODH-TOPO-002 — Environment and hardware boundaries.** Each Deployment Environment owns separate data-plane service instances and workload pools; a queue, label or Doris compute group is not an environment. Management may be shared within the Customer Organization, with its true wider impact disclosed. Production hosts/backing disks are separate from nonproduction. Development/test may use independent resource-bounded VMs on shared hardware; this does not create independent hardware failure domains.

**ODH-TOPO-003 — Placement and mixed workloads.** Default production placement separates storage and compute. Co-location is a named independently validated variation. Sharing compute hosts requires non-overlapping OS/service/YARN CPU and memory budgets plus disk/network/cache/spill controls. YARN does not automatically govern external host services. Specify queue/capacity/reservation/admission policies, streaming recovery headroom and maintenance conditions. Compare isolated and concurrent/burst/fault loads for batch completion, stream backlog/freshness/checkpoints, interactive tail latency/timeouts and utilization; correctness precedes performance.

**ODH-TOPO-004 — Real execution and auxiliary filesystems.** Include RM/NodeManager/AM/tasks, planning, HMS/database, Tez/selected compactor, shuffle/spill, worker libraries/native clients, accounts/credentials/renewal and staging/scratch/recovery/history/log paths. Endpoint/submission success is insufficient. Ozone table data compatibility does not prove localization or recovery filesystem semantics. An auxiliary HDFS Supported Dependency is allowed where justified and disclosed, with its own lifecycle/security/recovery; it is not automatically a second analytical backend or adequate Doris native storage capacity.

**ODH-TOPO-005 — Disaggregated Doris.** Target host/VM storage-compute-separated Doris, including FE, compute BE, Meta Service, actual FDB server/native/external-client pairings, shared storage, Recycler functionality and local state/cache/spill. Multiple compute groups share authorities and do not create independent recovery environments. Isolate native-file ownership/prefixes from lake maintenance. Certify HDFS vaults and any Ozone-native candidate separately; external-lake/S3 overlap or a TLS/snapshot extension setting does not prove native write, recycling, security or recovery support.

**ODH-TOPO-006 — Infrastructure and offline envelope.** Customers provide hardware/VMs, OS, networking, disks and declared foundations. The platform preflights actual OS/architecture/runtime, capacity/mounts/permissions, endpoints, name/time/trust/identity, failure domains and artifact readiness, and performs only approved software/account/directory/configuration preparation. Prove disconnected clean bootstrap, actual execution, restart, replacement, supported change and recovery. Connected mode only changes controlled acquisition of the same approved artifacts: no moving versions, mandatory public runtime/control service, default telemetry or automatic support upload.

## Security and identity

Sources: [security decision][security] **as amended by** [implementation/security decision][stack]. Apply guarantees to the actual selected configuration. The original blanket non-disableable production data-plane protection and always-combined directory/Kerberos interpretation are superseded; other governance, correctness, recovery and support obligations are not silently waived.

**ODH-SEC-001 — Product quality versus activation.** Customers select component capabilities, HA, ongoing backup/DR, component/data-plane SSL/TLS, TDE and Kerberos/LDAP within admitted configurations. Unselected data-plane protection does not automatically make a deployment demonstration-only. State real coverage, prerequisites, risks and exclusions; do not show disabled/unavailable capabilities as enabled or promise arbitrary toggle combinations. A selected protected path must satisfy its genuine native security and worker/bypass tests.

**ODH-SEC-002 — Non-disableable management minimum.** Require management authentication/scoped RBAC, authenticated agents and encrypted control transport through gRPC/mTLS, protection of management credentials/private keys and durable attributable critical management audit or its bounded durable-buffer/refusal behavior. Applicable management connections need actual native or other admitted protection, not a private-network assumption. Separate management, data, policy, audit and key authority; one person may explicitly hold multiple grants, without a universal two-person-approval requirement. Declared host/identity/key root authorities are trusted privileged actors; do not claim resistance to their complete infrastructure control.

**ODH-SEC-003 — Identity modes.** Maintain canonical Platform Identity mappings across actual protocol accounts/principals, not matching short usernames. Provide bounded support for optional integrated FreeIPA, external MIT KDC-only, external LDAP-only and external combined directory/Kerberos foundations. Prefer admitted customer foundations where supported. A KDC is not automatically a directory; LDAP-only does not supply secure Hadoop/YARN Kerberos or universal SSO. State actual user/group/OS-account sources and per-service/worker limitations; obtain explicit selection/remediation or decline a path needing missing capabilities. OIDC remains optional, not another required identity service.

**ODH-SEC-004 — Workload identities and credentials.** Record initiating and effective identities; reject ambiguous mappings/collisions. Use distinct least-privilege service/workload identities and bounded delegated credentials. Validate worker distribution, isolation, renewal, expiry and revocation on actual AM/task/executor/connector paths. Arbitrary jobs must not obtain a shared exportable storage-superuser credential or other users' authority. State authentication strength honestly in base configurations.

**ODH-SEC-005 — Enforcement and failure semantics.** Use Ranger as the principal policy/audit surface where supported, complemented by named native engine/storage enforcement. Identify each permission's authoritative source, synchronization and enforcement points. In protected claims, test SQL, direct engine/catalog/raw-file access, alternate interfaces, impersonation/delegation and credential reuse. Fine-grained row/column/masking guarantees require separately restricted paths and cannot coexist with an unrestricted raw-file bypass. Bound cached authorization and revocation; unknown, ambiguous or expired authority fails closed for affected operations rather than expanding privileges during an outage.

**ODH-SEC-006 — Actual protection coverage.** For selected transport/persistence protection, inventory and verify real protocol links and persistence boundaries, including workers, databases/catalogs/storage, Doris FE/BE/MS/FDB/connectors and applicable cache/spill/checkpoints/logs/audit/backups. One HTTPS endpoint, TLS-named setting or table-encryption feature is insufficient. Use actual peer verification and admitted native/protocol protection; publish unprotected boundaries and unsupported combinations. Data-plane TDE selection never permits plaintext management secrets.

**ODH-SEC-007 — Secrets, trust and keys.** Use a scoped protected local-keystore/customer-CA route built on maintained implementations for management secrets/PKI, with trusted bootstrap, issuance/import, access control, rotation/revocation, audit and independent protected recovery. OpenBao is neither default/required nor a committed adapter. Ranger KMS owns only admitted native key-provider/master-key integrations, not universal secrets/CA or presumed OpenBao compatibility. Protect master passwords, delegated material and historical keys separately from encrypted data/repositories.

**ODH-SEC-008 — Lifecycle and reversibility.** Enable/disable selected protection through approved preflight, impact confirmation, native sequencing and validation. Publish restart/outage/migration and reversibility constraints rather than lossless-toggle promises. Preserve old keys needed by existing data/backups even after future encryption is disabled. Test expired/revoked trust, unauthorized key access, rotation, renewal failures and required old-key recovery. Restore current ownership/policy/identity state without reviving retired writers or revoked authority.

**ODH-SEC-009 — Critical audit.** Retain durable, access-controlled, tamper-detectable records for applicable successful/denied access, changes, identities/policies/keys and management/lifecycle/recovery actions under the selected configuration. Record initiating/effective identity, actual target/environment, action, result, timestamp and correlation without secrets/unnecessary raw data. Management critical audit is always required. Validate real action/enforcement points, offline persistence, protected export and retention/deletion. A reliable durable buffer may cover bounded backend outages; exhaustible capacity is not permission to run affected operations unaudited. Quantify collection delay, retention/capacity and applicable revocation/cache bounds before publication.

## Governance and discovery

Sources: [architecture][architecture], [security][security], [roadmap][roadmap].

**ODH-GOV-001 — Metadata authorization.** Metadata-read permission is separate from source-data permission and default-deny. Explicitly authorized users may inspect metadata without data-read permission, but discovery must not grant query/sample/file access. Apply controls to search, APIs, descriptions, lineage neighbours and query links, including hidden names/adjacent entities. Sampling, previews and profiling are off by default and require separate source access/content-sensitive controls. Collection uses least-privilege identities.

**ODH-GOV-002 — Evidenced metadata and lineage.** Supply agreed table/field structure, descriptions, owners and dataset-level input/job/output lineage on declared reference paths. Map actual catalog/table/location/engine identities and query entry points. Expose delay, unresolved names and missing/unsupported edges; successful ingestion HTTP responses, pipeline inventory or a governance installation are not lineage evidence. Scope column-level coverage explicitly. OpenMetadata entities are not HMS/Iceberg commit authority or native Doris state; broad inventory does not imply certified lineage/security/recovery for every entity.

## Management and operator workflows

Source: [management-workflow decision][management], with [implementation foundations][stack].

**ODH-OPS-001 — Common managed operation loop.** Own target/ownership identification, prerequisite and permission checks, actionable plan and impact, required authorization/confirmation, native sequencing, progress/results, outcome verification and supported continuation/recovery/handover. Each manual step has an actor, prerequisites and evidence. Use Web UI and automation-capable API for regular management; core CLI covers essential management and independently usable offline bootstrap/diagnosis/recovery, not every UI or specialist-tool feature. API absence is not an authority/audit bypass.

**ODH-OPS-002 — Inventory and scope.** Expose selected capabilities, minimal dependencies, actual artifacts/runtimes, shipped/customer prerequisites, support/admission/approval state, ownership and real protection readiness. Distinguish new/adopted, connection-only, partial/failed admission and offboarded resources. Each action declares environment/cluster/service and true shared-foundation impact; a narrow action cannot silently mutate another environment. Keep one configuration/lifecycle control owner per resource.

**ODH-OPS-003 — Configuration and operator model.** Use environment/cluster/service/role/host navigation, Role Configuration Groups, versioned diffs and distinct desired/distributed/effective/pending-restart state. Show drift and its consequences; operators choose validated baseline adoption or restoration, rather than automatic overwriting. Provide hierarchical/per-target operation progress/history and contextual professional-tool entry with verified identity/permissions, correlation and return navigation. Do not expose credentials in links or require universal seamless SSO.

**ODH-OPS-004 — Approved host preparation.** Prefer DNS; a bounded validated hosts-file fallback must preserve actual discovery/identity/certificate requirements. Check deployment sudo capabilities and non-interactive/temporary-credential conditions; report missing rights rather than broadening sudoers or weakening host verification. Validate/create dedicated component users/groups and approved directories/packages/runtimes. UID/GID/ownership conflicts need explicit remediation, not recursive takeover. OS accounts do not themselves establish Platform Identity or Kerberos. Management may be privileged; components use their respective service identities.

**ODH-OPS-005 — Durable partial outcomes and safe retries.** Persist approved intent and attributable per-target completed/failed/unresolved/not-run evidence. After timeout, lost contact or restart, observe native effects before retrying. Retry only proven idempotent/safely recoverable steps; cancellation is not proof native work stopped. Stop unsafe dependents and expose continuation/repair/controlled compensation. No universal exactly-once effect, all-or-nothing transaction or automatic data rollback is promised.

**ODH-OPS-006 — Changes and upgrades.** Validate versions/dependencies, rights/provenance, drift, impact and operation-specific recovery conditions before supported staged/batched or outage-based change. Recheck actual running state and affected data/security paths. Changed premises or wider-than-approved scope require reassessment. Rolling upgrade, downgrade and schema/data reversal need distinct evidence. Emergency patches retain mandatory integrity/security/licensing gates.

**ODH-OPS-007 — Capacity, retirement and maintenance.** Customers supply infrastructure; the platform coordinates supported join/drain/rebalance/replacement/removal within budgets. Include safe shrink/retirement or a declared outage/replacement alternative/refusal, not arbitrary autoscaling. Service removal, storage reclamation and data destruction are distinct decisions. A maintenance window authorizes timing but does not prove quiescence. Coordinate native writer/DDL/maintenance fencing and customer-owned workloads with explicit stop/resume confirmation; retain active recovery and in-flight state, and refuse unsafe work.

**ODH-OPS-008 — Adoption and offboarding.** Adoption performs read-only assessment, remediation/admission checks, recovery/identity/provenance validation, explicit single-controller transfer and actual data/service acceptance before declaring management. A failed transfer remains partial with its real responsible owner; do not automatically re-enable a competing old manager. Offboarding hands over required operating/configuration/access/certificate material and ceases platform control without implying stop or deletion. Migration eligibility or a connection does not establish adoption; preserve accepted services/data and validate the receiving owner.

**ODH-OPS-009 — Health and diagnosis.** Distinguish host/process, dependency/service, application/job and declared end-to-end data-path health. Unknown/stale evidence is not green; maintenance suppression preserves underlying evidence. Integrate metrics/logs, acknowledgment, notifications and correlated actionable alerts using Prometheus/Alertmanager, Grafana and Loki/Alloy. Scope thresholds/storage/retention/access/offline dependencies. Diagnostic bundles have authorized target/time/version/action context, preview/redaction, protected export and traceable access; exclude secrets/raw business data by default. Customers authorize support exchange; no automatic upload is required.

**ODH-OPS-010 — Acceptance and handover.** Delivery ends with meaningful selected data/security/operations evidence and explicit receiving-administrator acceptance, not installer exit status. Hand over inventory, entry points/permissions, ownership, conditional prerequisites, support/maintenance terms, procedures, protection status/gaps and protected recovery material. Show the administrator can operate the accepted scope, including supported change, failure diagnosis/recovery and two-environment targeting/isolation.

## Availability, backup and disaster recovery

Sources: [security/recovery decision][security], [activation amendment][stack], [management workflows][management], [Doris recovery evidence][doris-research].

**ODH-RES-001 — Separate selectable offerings.** Ultimately offer HA, managed backup/restore and cold/warm/hot cross-site DR through complete Certified Paths. Customers may omit these with real protection/risk explicit; an individual operation may still require a usable recovery point. Optional protection meets production quality when selected. Different paths may offer different tiers; neither every combination/every tier nor completion of the DR target by cold alone is promised.

**ODH-RES-002 — HA contract.** Declare actual fault domains, automatic takeover and task interruption/resumption, dependency conditions, partitions and stale-leader fencing. Include shared management/identity/policy/database/storage effects. Replica counts and VMs sharing a host are not independent HA proof. HA neither preserves historical recovery points nor establishes site-loss protection.

**ODH-RES-003 — Managed backup.** Provide scheduled/on-demand execution, consistency/quiescence coordination, compatible manifests, independent protected/versioned repository, integrity/access checks, retention/cleanup, failure visibility and restore/drill workflows. Reuse admitted native mechanisms. A dump collection, live mirror, same-domain snapshot or successful capture without restoration evidence is insufficient. Protect required keys and historical materials independently; partial/incompatible/missing-key captures are not complete recovery sets. Backup encryption/security is explicit, not inferred from optional runtime TDE.

**ODH-RES-004 — Compatible recovery units.** Recover compatible native authorities and all required state, not just component processes:

| Path/state | Required compatible state and validation |
| --- | --- |
| Legacy Hive | Files, HMS/database schema and applicable transaction/write-visibility state, configuration and security; reconciled rows and future transactional behavior. |
| Shared Iceberg | Catalog authority/current metadata, metadata files and every referenced data/delete file for the claimed state. Retained snapshots alone are not independent backup. |
| Kafka/Flink/Iceberg | Source events/offsets/retention, applicable checkpoints/savepoints, commit state and retained files with declared duplicate/replay semantics. |
| Disaggregated Doris | Compatible FE control metadata, FDB transaction/data-layer state, retained native files and applicable source/progress/WAL/security state, plus all-writer/Recycler fencing. |
| Management/governance/foundations | Required databases/configuration, policy/identity mappings, human-maintained governance, job definitions and key/trust recovery, with explicit customer-foundation ownership. Distinguish rebuildable indexes/cache from irreplaceable state. |

A Serving Copy may be rebuilt only from protected adequate source/replay material with tested correctness and claimed RPO/RTO. Do not infer native Doris recovery from FDB-only protection, live file replication, SQL BACKUP for another deployment mode or extension/API names. Its exact FE capture/restore, FE/FDB alignment, file retention, vault-address portability, dead-source activation, native client/security, writer/Recycler fencing and failback remain release gates. Accepted research documents gaps; it is neither operational certification nor proof that no solution exists.

**ODH-RES-005 — DR tiers.** Certify each offered tier on a complete declared path:

| Tier | Required behavior |
| --- | --- |
| Cold | Restore at another site from independently available recovery materials/dependencies without relying on the lost source; manual initiation and rebuilding under declared prerequisites are allowed. |
| Warm | Preposition standby services/resources and synchronize compatible state periodically/continuously; support controlled manual takeover. Retrieving backups only after loss is not warm. |
| Hot | Keep full-path services ready with low-lag synchronization and safe automatic takeover within a preauthorized certified envelope. Hot does not mean synchronous, zero-RPO or active-active. |

Separate historical restore, planned switchover, unplanned site loss, partitions, takeover and failback evidence. Specify acknowledged/committed/loss/replay boundaries, lag, detection/takeover and RPO/RTO against exact artifacts/workload/topology. DR materials remain independently accessible/decryptable and offline dependency-complete. A live mirror is not historical protection.

**ODH-RES-006 — Activation versus failback.** Open current-target writes only after consistency, exclusive authority/fencing, security and applicable approval pass. Cold/warm activation is operator-approved; hot automation has a separately validated/preauthorized envelope. Do not wait for old-site rebuilding or future failback readiness to activate a safe current target. Stop affected writes if exclusive authority is unknown. Later failback is a distinct synchronization/reconciliation/approval/validation operation, never endpoint reversal alone; published claims still require the promised return-path evidence.

**ODH-RES-007 — Independent management recovery.** Existing data services continue during management outage while actual runtime/security dependencies hold. Freeze unsafe changes and reconcile live state, ownership, operations, credentials/policies and audit before resuming; do not blindly replay or restore stale controllers. Provide independent offline bootstrap/CLI and protected database/key/state recovery materials. R1 has recoverable single-active management, not automatic management HA. Ensure an unavailable manager does not secretly invalidate an advertised data-path HA dependency.

## Managed CDP data migration

Sources: [architecture][architecture], [support][support] and the added migration requirement in the [release roadmap][roadmap].

**ODH-MIG-001 — First migration scope and targets.** Productize bounded CDP-origin HDFS file/directory and Hive table/partition migration, including required metadata, into admitted OpenDataHub HDFS/Hive targets. Preserve the selected logical data/structure/behavior contract without mandatory Iceberg conversion. Ozone targets and format conversion require distinct explicit certification. Migration transfers selected data/use; it does not adopt the source service.

**ODH-MIG-002 — Managed full and incremental workflow.** Provide bounded source/target/object selection, policies, preflight, plan/authorization, initial full transfer, periodic or on-demand differential synchronization, per-object progress/results, safe retry/resume and reconciliation reports. Coordinate planned stop-write/DDL/maintenance, accepted consistency capture, verification, endpoint cutover, stale-writer fencing and administrator handover. Specify actual source/target ownership/cleanup and controlled single-writer target staging. File deltas do not prove Hive ACID incremental visibility or delete semantics; establish mechanisms/evidence separately for each claimed object/table class.

**ODH-MIG-003 — Destructive changes and mapping.** Default to no implicit destructive target changes. Overwrite, mirroring and deletion propagation require explicit policy, impact preview, authorization and object-specific correctness/recovery tests. Address renames/deletes and existing-target conflicts. Preserve file bytes and declared filesystem attributes; map target locations, identities, permissions and applicable encryption/key material deliberately, without copying privileges blindly. Protect and admit source credentials/native clients and actual rights.

**ODH-MIG-004 — Hive preservation categories.** Preserve selected common HS2 clients/SQL/Hive-on-Tez behavior, non-transactional managed/external ORC/Parquet tables/partitions, and selected ACID update/delete behavior. Certify claimed non-transactional, insert-only and full-ACID classes separately; one does not prove the others. Reconcile committed-visible content, schema/partition/lifecycle, clients/results and correct future transactional updates/deletes/maintenance/restart. Report unchanged, configuration-adapted, query/job-adapted and unsupported cases. Open transactions, all historical write IDs/history and arbitrary custom UDF/SerDe/JAR/CDP parity are not default preservation promises.

**ODH-MIG-005 — Cutover and recovery boundary.** Allow a planned source write outage. Fence writers, DDL and maintenance, handle transactions/readers under a declared procedure, obtain consistent state and reconcile before switching access. Before target writes, return to a frozen source only via a validated procedure. After divergence, prefer target recovery; source return needs separately designed/tested reconciliation/replay. Copying HMS rows/ACID directories, matching markers, counts alone or reversing endpoints is not safe migration/recovery proof. Cover partial export/import, interruptions, ambiguous completion and failed cutover.

**ODH-MIG-006 — Reference, exclusions and evidence.** Reference Cloudera Manager replication tooling's operator experience, not proprietary code or mandatory API/policy compatibility. Use first-party orchestration of admitted mature native mechanisms. Customer-supplied restricted tools/clients are conditional on rights/provenance/technical admission, not automatically bundled. Initially exclude all-CDP/HBase/Kudu/Kafka migration, generic near-real-time CDC, indefinite replication service, implicit backup/DR and zero downtime. Set actual object/data/partition sizes, synchronization lag/interval, retention/replay and stop-write/cutover/recovery limits in the version map; validate changes/deletes, permissions, source profiles/conflicts and failed recovery, not just copies completing.

## Implementation and repository foundations

Source: [implementation decision][stack]. These are selected foundations, not claims of built applications or admitted shipping versions.

**ODH-IMPL-001 — First-party management.** Build bounded first-party lifecycle orchestration over mature host/native mechanisms, not an Ambari-centered product or a script collection. Use Go for controller, lightweight agent and core CLI, with a modular monolith and independently runnable workers where useful. Reference Cloudera Manager's operator model independently of language; suitable Bigtop/native/library reuse requires actual integration/provenance/license admission. No universal automation engine is selected.

**ODH-IMPL-002 — UI and management interfaces.** Use React/TypeScript with fixed compiled assets served by the controller by default, REST/OpenAPI for UI/core CLI/automation and explicitly versioned interfaces. Serving UI alone does not require production Node.js or public CDNs. An admitted customer reverse proxy/LB preserves real authentication/trust. Specialist tools retain their scoped native responsibilities.

**ODH-IMPL-003 — Agent protocol.** Agents initiate versioned Protobuf/gRPC connections using mutual TLS. Define/test enrollment, reconnect, acknowledgments, redelivery, stale authority and version compatibility. Transport reliability alone supplies none of those effect/authorization guarantees.

**ODH-IMPL-004 — Dual management SQL backends.** Support bounded tested PostgreSQL and MySQL. Use one authoritative backend per deployment; connect admitted customer-provided databases, or deploy independently recoverable managed PostgreSQL by default. Declare lifecycle/protection ownership and preflight version, permissions, protected connectivity, durability/capacity and recovery. MySQL does not imply MariaDB/arbitrary variants, cross-engine dual writing or automatic migration. This does not select all service databases.

**ODH-IMPL-005 — Durable operation module.** Implement the bounded first-party Go operation module in the management database. Validate actual PostgreSQL/MySQL transactions/isolation/locking/claims/deadlocks/schema migration and ambiguous commits separately. Controller-runtime, River and Temporal supply reference patterns, not adopted operation runtimes. No independent Redis/Raft/Kubernetes/Temporal service or memory-FSM durability claim is selected. Exact tables/leases/helper libraries remain version decisions.

**ODH-IMPL-006 — Agent-local durability.** Use bbolt for bounded execution records, pending result/acknowledgment state and reliable buffers. It is neither central service authority nor an encryption/audit solution by itself. Validate filesystem/sync/crash/disk-full/corruption behavior, retention/migration and protected recovery. Resending a recorded result is not rerunning a command; reconnecting does not renew stale controller/attempt authority.

**ODH-IMPL-007 — Bootstrap and process control.** Use first-party Go SSH bootstrap wrapping maintained SSH, approved preparation and authenticated agent enrollment; use agents for routine management. Ansible is not required. Use systemd with correct native foreground/daemon/PID/user/process-group/limit/stop semantics, not a new process supervisor. systemd is not distributed drain, membership, migration, transaction or recovery. Actual Hive/Tez work is not a fictional daemon probe.

**ODH-IMPL-008 — Definitions and adapters.** Use schema-validated declarative component definitions with typed Go Adapters for native install/configure/observe/drain/change/adoption/offboarding/recovery prerequisites and failure semantics. Scripts/CLIs use admitted runtimes. Do not start with a dynamic Go plugin ABI, arbitrary uploaded plugins, marketplace or general workflow DSL. An executable adapter change may require an application rebuild; independently versioned definitions do not imply arbitrary hot loading.

**ODH-IMPL-009 — Repository and build ownership.** Use one first-party monorepo for applications/UI/contracts, component build/integration definitions, distribution manifests, fixtures and documentation; reference pinned upstream sources/forks rather than vendor every tree. Large binaries/source collections/run evidence live in artifact storage. Use GitHub Actions with isolated runners and controlled self-hosted heavy-build runners; isolate untrusted builds from signing credentials. Assign actual management/host/UI/component/release maintenance responsibility before publication. Build-time containers/connectivity do not select containerized runtime or a complete offline/bit-reproducible rebuild guarantee.

The selected logical organization below is a design boundary, not a claim that these implementation directories have been scaffolded:

```text
cmd/{controller,agent,odh}/  # first-party executables
internal/                  # management/operation/host/security modules
web/                       # management UI
contracts/                 # API, agent and integration schemas
components/<name>/         # build inputs, patches, adapters, templates, fixtures
distributions/             # exact combinations, source/notices, evidence references
tests/                     # contract, integration and certification fixtures
docs/                      # guidance, operator/design material and qualifying ADRs
```

## Distribution and licensing

Sources: [licensing decision][licensing], [support decision][support], [implementation decision][stack].

**ODH-DIST-001 — Immutable versions and activation states.** Produce separately versioned immutable Component Bundles through controlled pinned-source compilation or reviewed binary repackaging. Every Platform Distribution release/update pins the approved application/agent/definition/executable-adapter/component/runtime combination. Keep downloaded/verified/distributed-staged/selected-active/actually-running/validated state distinct. Deployed data/configuration/secrets live outside immutable payloads. Old active/recovery-required bundles cannot be removed merely because a new one is selected. Bundle switching/retained binaries do not establish native schema/data rollback.

**ODH-DIST-002 — Delivery/dependency closure.** Use static HTTPS artifact storage and offline media; no mandatory OCI registry/Harbor/Nexus is selected. Name actual OS/native/runtime/client/plugin/trust prerequisites and complete lifecycle/recovery closure. Record self-contained shipped subsets separately from customer-completed conditional paths. Changed inventories get identifiable revisions and affected-path validation, not silent substitution, moving latest versions or unrestricted cross-release mixing. Declare/test actual upgrade compatibility windows.

**ODH-DIST-003 — Integrity and release aids.** Use Syft for SPDX JSON inventory, Trivy as a vulnerability-check aid and Cosign for signed manifests/disconnected verification with fixed trusted materials. Bind actual payload digests, transitive/native contents, source/patch/build records and certification evidence. Validate scanner coverage/freshness and offline signer trust; scans/checksums are not legal clearance or path certification. Keep private signing keys in controlled release infrastructure; customer verification must not require public identity/transparency services.

**ODH-LIC-001 — First-party and third-party rights.** Original platform code/UI/scripts intend Apache-2.0; third-party code retains its actual license and applicable notices/obligations. Publish applicable modifications and required source. Commercial support terms must not negate granted rights; first-party licensing does not clear a derived/combined work or establish ownership of others' code. This PRD does not itself install a license file or grant legal clearance.

**ODH-LIC-002 — Activity/artifact-specific admission.** Review connection, unmodified redistribution, modification, plugins/drivers, linking/embedding/combined works and changed hosted/OEM/business models separately. Verify actual community/enterprise/server/client/transitive artifacts, not project names. Prefer permissive artifacts; MPL/LGPL/GPL/AGPL are not categorically excluded or commercially forbidden. Restricted/source-available/proprietary artifacts require documented bounded rights/obligations and authorized exceptions. A process/network boundary is not automatic legal shelter; AGPL obligations may matter in private operation. Missing rights cannot be manufactured by an internal exception.

**ODH-LIC-003 — Binary and source product delivery.** Deliver built binaries as the primary product plus exact matching source archives/collections: first-party code, modifications and any required third-party corresponding source/build/install materials. Use immutable binary/source/patch mappings, not moving branches or unrelated first-party archives. Source need not be installed as runtime payload. No blanket full upstream offline rebuild closure or bit-for-bit reproducibility promise is added; actual license obligations still apply.

**ODH-LIC-004 — Delivery-specific fulfillment.** Select and fulfill an applicable source mechanism for each network/offline/physical delivery and license. A matching public download can be sufficient for a qualifying delivery; offline delivery may need accompanying source or another permitted mechanism, such as a valid written offer with access/duration/fulfillment. Neither an arbitrary repository link nor mandatory source-in-installer is universally correct. Preserve applicable network source offers and required availability.

**ODH-LIC-005 — Recipient changes, notices and brands.** Preserve LICENSE/NOTICE/copyright/attribution/modification material and applicable trademark rules. Customer modification/redistribution rights do not imply official branding, upstream endorsement or certification. Validate affected paths of modified artifacts without automatically voiding unrelated support. Legal rights and technical support are distinct.

**ODH-LIC-006 — Inventory, exceptions and accountability.** Maintain actual build/version/digest, origin/license/terms, modifications/source mapping, activity/delivery scope, notices/source fulfillment, shipped/customer-supplied status, accountable approval and exception/re-review triggers. Customer-supplied vendor/CDP artifacts require actual entitlement/provenance/use and technical admission, not assumed redistribution rights. Technical and licensing/source acceptance are separate mandatory gates. Name release owners and authorized human approval; ambiguity/restricted or strong-copyleft integration risk requires appropriate qualified review. Keep confidential legal/entitlement records protected. Missing rights/materials block affected combinations; lawful complete unaffected paths may proceed only with accurate scope. Do not discard a required long-term capability merely because one candidate fails admission.

## Support contract

Source: [support decision][support].

**ODH-SUP-001 — Path-scoped support.** Bind each Certified Path to a Platform Distribution release, actual artifacts, operations, prerequisites, configuration/topology/workload and evidence/exclusions. Installing a component is not certifying its integrations. New deployment, adoption, migration and conversion need distinct evidence. Composition of individually certified paths requires declared interaction conditions. Experimental is not a production-support promise; deprecation retains obligations through its announced window.

**ODH-SUP-002 — Source compatibility profiles and admission.** Use finite Source Compatibility Profiles with a justified set of fully validated representative source versions/builds, publishing samples separately from profile coverage. Non-sample customer sources receive actual-environment prechecks and isolated representative small-scale trials. Material transaction/export/metadata/security differences trigger targeted investigation, remediation or rejection. Neither a shared CDP family label nor admission proves full testing of that exact build; being non-sample does not force full regression of every unrelated service/build.

**ODH-SUP-003 — Maintenance, defects and EOL.** Before stable-line publication, announce maintenance windows, deprecation notice periods, supported updates/upgrades and withdrawal arrangements. No universal multi-year term/concurrent-line count is invented here. Own unified intake, reproduction/diagnosis, impact assessment and mitigation within supported scope, including upstream issues; possible remedies are reviewed fixes/backports/upgrades/workarounds/restrictions. Do not make customers coordinate every upstream project, guarantee every repair or promise indefinite forks. Upstream EOL requires an explicit maintainable disposition and communication. Product support is not automatically 24x7 staffing, response/repair time or customer business SLA.

## Reference acceptance and evidence

Sources: [product scenarios][product], [architecture][architecture], [support][support], [management][management], [roadmap][roadmap].

**ODH-ACC-001 — Four required long-term reference journeys.** Each offered subset has known synthetic inputs, expected outcomes and reproducible evidence, progressively allocated below. These journeys are not exhaustive customer-workload promises or an obligation to ship every component first.

| Journey | Observable acceptance | Initial allocation |
| --- | --- | --- |
| Batch lakehouse analysis | Exported business tables → cleaning/joining → persisted detail/aggregates → analyst query. Reconcile known results, rerun without double-counting, schedule and recover interruption through a declared procedure; expose partial completion. | R1 |
| Streaming business-event analysis | Reproducible events → continuous ingest/processing → queryable results. Include duplicate event identities and restart/recovery; reconcile actual committed results under declared replay/dedup/recomputation semantics and query-visible freshness. No universal end-to-end exactly-once or required CDC. | R2 |
| Discovery to authorized use | Find reference data, inspect agreed metadata/provenance and real input/job/output lineage, reach a working authorized query. Deny unauthorized source-data reads under the protected path; metadata visibility remains separate. | R2 |
| Offline delivery and continuous operations | Outbound-denied establish/accept/handover; supported change and failure handling; inject declared fault, detect/diagnose/recover and recheck data. Two environments/clusters prove targeting/isolation. Demonstrate bounded adoption and rejection/partial handling, not connection-only success. | R1 subset; expands with all paths |

**ODH-ACC-002 — Quantitative envelopes.** Before applicable supported publication/admission, set finite numeric limits, endpoints/clocks, hardware/runtime/topology and procedure for data/row/table/partition/file/object sizes; event rate/size/bursts and concurrency; resource budgets/mixed-load degradation/admission; batch duration/query-visible freshness; retention/replay/dedup/checkpoints; identity/credential/cache/revocation; audit capacity/retention/delay; backup/synchronization windows; fault detection/takeover/recovery and RPO/RTO. Migration additionally needs increment/lag/stop-write/cutover bounds. No inherited background-customer figures or arbitrary global numbers are supplied by this PRD. An out-of-envelope path needs validation before expanding support, not hidden data loss.

**ODH-ACC-003 — Real layered validation.** Plan Go module/interface tests; actual PostgreSQL/MySQL transactions and ambiguity; agent protocol/enrollment/bbolt/credential/audit failures; UI/browser, REST/core CLI, actual static resources, package/signature/dependency/bootstrap/drift/partial-operation tests; and actual native distributed/service/path change/adoption/offboarding/selected recovery tests. Run relevant full lifecycle operations outbound-denied. Include permissions/sudo/account conflicts, missing artifacts/keys, stale controllers/writers/Recycler, denied cross-scope actions, duplicates/deletes and partial/incompatible materials as applicable. A process, HTTP, job, checkpoint, scan or backup success is not reconciled data/security/recovery evidence. No such certification run is claimed here.

**ODH-ACC-004 — Requirement-to-evidence record.** Maintain a release ledger linking each applicable PRD requirement and source decision to mandatory/optional/deferred/excluded scope; exact path/configuration/artifact/runtime/topology and relevant source sample/profile/admission; dependency/delivery/ownership conditions; reproducible fixture and expected result; workload/numeric gate; actual evidence/result and accountable acceptance owner. Keep public safe fixtures/manifests/support terms separate from protected customer/entitlement/legal evidence. Distinguish representative certification, reusable profile coverage and customer admission. Record technical approval separately from licensing/source fulfillment, with immutable/revisioned manifests and affected-path validation.

## Feature-release roadmap

Source: [release-roadmap decision][roadmap].

**ODH-PLAN-001 — Ordered feature stages.** Deliver batch first, streaming next; resilience is HA first, then backup/cold DR, then warm and hot. The labels below are feature stages, not dates, formal product version numbers or exactly one release each. Later stages may split into coherent releases as their maps mature; explicit changes carry unmet obligations forward rather than deleting them. Customers select dependency-complete subsets; optional offered capabilities remain production-grade.

### R1 — Batch Lakehouse & HA

**ODH-PLAN-002 — First supported release boundary.** The first external release is narrow and formally supported, not a technical preview. Its mandatory offered capabilities are:

- HDFS as the sole certified analytical backend; actual YARN/Spark → Iceberg/HMS → Trino with Airflow, known batch fixtures and real dependencies.
- A selectable end-to-end HA offering for the main batch path, including runtime dependencies, declared task interruption/resumption, fault domains and exclusive authority. A recoverable single-active manager is not management HA; its outage must not conceal a failed dependency of the offered data-path HA claim.
- Base and protected finite data-plane profiles. The protected reference includes optional integrated FreeIPA, explicit identity mapping, Kerberos where applicable, Ranger/native authorization and actual-link transport protection. Base mode does not silently install FreeIPA or claim unselected protection. No arbitrary toggle matrix is implied.
- Both admitted customer PostgreSQL/MySQL management backends and independently recoverable platform-managed PostgreSQL by default when none is provided. Service databases remain separately admitted.
- The selected management/UI/API/core-CLI/agent, durable-operation, SSH/systemd, definition/adapter and signed-bundle foundations, with the non-disableable management minimum, scoped keystore/customer CA and independent key/state/database recovery.
- Offline-usable Prometheus/Alertmanager, Grafana and Loki/Alloy integration, contextual professional tools, meaningful health/diagnostics and the complete selected operator loop/handover.
- Full selected batch new deployment, plus separately bounded Trino adoption/offboarding, admission/remediation/rejection and partial control-transfer recovery. This is not whole-cluster/CDP adoption.
- Limited explicitly verified table operations and safe writer/maintenance/retention procedures through named entry points. The full Hive SQL contract is deferred to R4, not treated as fulfilled by Spark alternatives.

**Envelope:** independent Linux host/VM management, host services plus actual YARN, separate storage/compute production reference, independent environments/pools and production/nonproduction hardware boundaries. Validate both SQL backend semantics, the admitted base/protected/HA configurations, two-environment isolation, actual data-path HA faults and meaningful change/fault recovery. Exact immutable proposal identities, finite configuration matrix and numbers are version-map decisions, not supplied by this PRD.

**Excluded/deferred:** streaming/discovery scenario completion, CDP/general Hive migration, full Hive SQL administration, Doris/Ozone, data-plane TDE/Ranger KMS, external identity modes, ongoing managed data backup/DR and automatic management HA. Necessary pre-operation recovery points, safe limited maintenance, independent management/key/state recovery and declared service-fault recovery still apply; they are not historical-backup product claims. Unproved mandatory first-release scope, especially offered end-to-end HA, blocks its promised publication unless explicitly reapproved.

### Subsequent stages

The recurring delivery/security/support/acceptance obligations continue for every new subset. Existing supported paths follow their published maintenance/update terms. New components do not inherit security, HA, backup/DR, migration or lineage certification.

| Stage | User value and offered increment | Prerequisites, workflow and evidence | Explicit boundary |
| --- | --- | --- | --- |
| **R2 — Streaming & Evidenced Discovery** | Kafka → Flink → shared Iceberg → Trino; OpenMetadata discovery and dataset-level reference batch/stream lineage followed by authorized queries. | Build on R1's admitted shared-lake foundation. Reconcile duplicate/replay/restart/commit and measured freshness; verify real lineage, metadata visibility and denied source reads under the protected profile. Complete offline/operator loops for new paths. | No generic CDC, universal exactly-once or all-engine/column-level lineage. Any streaming HA claim is separate from R1 batch HA. |
| **R3 — Managed Backup, Cold DR & Security Expansion** | Scheduled/on-demand managed backup, historical restore and independently complete cold DR on selected full batch/stream paths; bounded TDE/Ranger KMS and the remaining external identity modes. | Follows R2 and precedes Hive/Doris expansion. Reconcile actual compatible source/checkpoint/catalog/files/security state; separately test historical deletion/corruption and complete primary-site loss. Prove independent offline materials, key recovery, retention and measured RPO/RTO. | No warm/hot or management HA claim. Backup protection does not wait for runtime TDE; identity modes have honest service-specific limits, not all toggle combinations. |
| **R4 — CDP Data Migration & Hive Administration** | Managed HDFS/Hive full plus periodic/on-demand incremental migration and controlled cutover; real HS2/Hive/Tez and bounded Hive SQL maintenance. Begin non-transactional cases and separately expand selected transaction classes. | Follow R3 protection and actual target admission. Use source profiles, finite representative samples and customer trials; reconcile content/SQL/transaction behavior, destructive policy, partial failure, cutover and recovery. | No universal CDP build/plugin parity, automatic service adoption, zero downtime or automatic post-divergence rollback. Ozone targets/conversion are separate. |
| **R5 — Doris Analytical Expansion** | Separately offer external Iceberg queries and disaggregated native Serving Copies, initially on the HDFS foundation; selected Spark/lake-native ingestion. | Follow R4; admit real FE/BE/MS/FDB/Recycler/storage/native clients. Reconcile source/native correctness, freshness, retry/reload, updates/deletes and claimed restore/rebuild. Native coordination/security gaps gate the affected path/tier. | External queries do not certify native tables; no integrated-mode substitution or automatic Ozone vault/DR support. Explicitly defer any unproved separable claim instead of advertising all Doris as delivered. |
| **R6 — Ozone Storage Expansion** | Separately validated Ozone interfaces/backends for selected existing analytical paths and their lifecycle/recovery. | Follow R5; prove actual worker/auxiliary filesystem semantics, URI/layout/credentials, mixed-load budgets and faults. Name/maintain auxiliary HDFS where needed. | No implicit backend coexistence/co-location, transparent switching/replication or native Doris Ozone support from lake/S3 overlap. |
| **R7 — Warm DR & Management HA** | Selected complete-path prepositioned standby, compatible-state synchronization and controlled manual takeover; actual automatic management HA. | Follow cold/historical protection and preceding stages. Prove lag, source independence, partitions, writer/controller fencing, takeover, security and separate failback. Include database/key/audit dependencies. | Rebuild-after-loss is not warm; management HA does not protect all data paths or replace historical backup. Coverage is path-specific, not every combination. |
| **R8 — Hot DR** | Ready full-path standby, low-lag synchronization and safe automatic takeover in a preauthorized certified envelope. | Follow R7. Exercise complete primary-site loss, partitions, dead-source activation, old writers/Recycler, acknowledged/loss/replay boundary and approved later failback against numeric bounds. | No zero-RPO, synchronous replication, active-active or universal path coverage. Stop writes if exclusive authority is uncertain; cold-only is not the complete DR product. |

## Version maps and change control

Sources: [roadmap][roadmap], [PRD handoff correction][handoff].

**ODH-PLAN-003 — Version-map opening gate.** Every version map must name this PRD's exact accepted Git revision and applicable requirement identifiers before working release decisions. Its Notes must point to `PRD.md`, relevant sections and decision sources; scope records identify mandatory offerings, optional activation, inherited/deferred/excluded requirements with reasons and carry-forward destination. Open a map when user value, path/component/topology boundary, prerequisites and precise unresolved decisions are visible. Choosing versions/numbers is valid map work; they need not already be decided. Use native blockers for missing prerequisites. Open later maps progressively rather than pre-creating all version maps or implementation backlogs.

A release map lacking the PRD handoff or an accessible pinned revision must stop at that prerequisite and obtain it. The predecessor issue index alone is not a substitute for the requested PRD artifact. The current [Batch Lakehouse & HA — Release Planning][first-map] is the first consumer.

**ODH-PLAN-004 — Planning-completion gate.** A version map resolves its human decisions and required investigations into a coherent implementable release specification: proposed immutable artifacts/OS/architecture/runtimes and owners; actual SQL/identity/security/topology/configuration boundaries; protocols/state/agent compatibility; lifecycle/UI/API/core-CLI journeys; independent bootstrap/key/state recovery; finite numeric acceptance targets; dependency/offline/source fulfillment; fixtures/evidence plan and support terms. Bind choices to PRD identifiers. Use focused further planning where still too large. Closing a planning map is specification handoff, not implementation, test pass or support publication.

**ODH-PLAN-005 — Supported-publication gate.** Execution work must supply actual artifacts, reproducible positive/negative/fault/recovery evidence meeting frozen scope/numbers, handover and separate human licensing/provenance/source approval. Actual payload identities, compatibility/maintenance windows, deprecation notice, updates/withdrawal and accountable people cannot remain blank. No version label waives missing dependencies, unproved selected security/recovery, rights or source materials. Separable unaffected paths may proceed only with accurate approved scope and complete dependencies. Mandatory core scope changes require explicit reapproval. Revise manifests/evidence immutably with affected-path validation.

**ODH-PLAN-006 — PRD amendments and total-map completion.** The total map's output includes this actual PRD and its accepted handoff, not only enough issues to write it later. During the initial PRD handoff review, keep [Produce the consolidated PRD and bind release maps to its requirements][handoff] and the total map open; after acceptance/merge, record the accepted commit, complete handoff and close them. Requirement IDs remain stable: do not renumber/reuse them; an approved retirement records replacement and rationale. A new approved product decision must update this PRD and affected map baselines/traceability together. A release map may refine an allocated contract but cannot silently weaken or delete it. Log changed assumptions and human approval; preserve provenance in the relevant decision ticket rather than maintaining competing PRD copies in issues.

### Minimum version-map handoff record

| Field | Required content |
| --- | --- |
| PRD baseline | Repository, accepted commit and `PRD.md`; relevant requirement IDs and source decisions. |
| Scope | User value, mandatory/optional/deferred/excluded capabilities, dependency-complete paths and remaining obligations. |
| Concrete envelope | Actual proposed application/agent/definition/adapter/component/runtime/OS/architecture versions, topology, identities, SQL and customer/platform ownership. |
| Acceptance | Reproducible fixtures, expected data/security/operational outcomes, workload/numeric pass/fail bounds and required negative/fault cases. |
| Evidence lifecycle | Planned versus implemented/tested/certified state, actual evidence/manifest identities, owners and public/private boundaries. |
| Delivery and support | Artifact/source/notices/rights fulfillment, approvals, maintenance/deprecation/update/withdrawal and handover conditions. |

## Requirement allocation index

This index gives the initial product allocation for every requirement family. A family/range includes each defined identifier in that range. Cross-cutting requirements begin with the offered R1 subset and recur for each applicable later capability; this is not a claim that the entire long-term contract ships in R1. Version maps must expand the applicable rows into their requirement-level scope/evidence ledger.

| Requirements | Initial allocation and continuing obligation |
| --- | --- |
| ODH-PROD-001–005 | R1 product/operating model; preserve all personas and boundaries throughout. |
| ODH-CAT-001 | Catalog allocated component-by-component above across R1–R6; required long-term capabilities remain present in the roadmap. |
| ODH-DATA-001–005, ODH-DATA-007 | R1 batch/Iceberg/HDFS subset; streaming R2, legacy Hive R4, Doris R5 and Ozone R6 extend distinct contracts. |
| ODH-DATA-006 | R4 full bounded Hive SQL offering; R1 only its explicitly named safe limited operations, not completion of the full requirement. |
| ODH-TOPO-001–004, ODH-TOPO-006 | R1 host/YARN/environment/budget/offline subset; actual Hive/Tez closure R4 and Ozone auxiliary variants R6. |
| ODH-TOPO-005 | R5 disaggregated Doris; Ozone-native candidates remain separately conditional, not a required cross-product promise. |
| ODH-SEC-001–009 | R1 management minimum plus base/FreeIPA-protected scope; R3 adds external modes/TDE/KMS; each later path declares actual coverage and evidence. |
| ODH-GOV-001–002 | R2 declared reference discovery/lineage; later engines/backends require separate coverage/evidence. |
| ODH-OPS-001–010 | R1 coherent batch/new-deployment/limited-Trino-adoption loop; expand and reverify for each later offered path/action. |
| ODH-RES-001–002, ODH-RES-007 | R1 optional batch-path HA and independently recoverable single-active management. Management automatic HA R7. |
| ODH-RES-003–004 | R3 selected managed historical recovery; operation-specific recovery prerequisites and independent management/key/state recovery apply from R1. Later native paths need their own compatible sets. |
| ODH-RES-005–006 | Cold R3, warm R7, hot R8 on declared complete paths; current-target activation and later failback gates apply whenever recovery is offered. |
| ODH-MIG-001–006 | R4 bounded CDP HDFS/Hive migration. No implicit Ozone target, conversion, all-CDP service or generic CDC commitment. |
| ODH-IMPL-001–009 | R1 selected management/build foundations; exact versions/contracts/testing decided per version. |
| ODH-DIST-001–003 | R1 and every release/update/conditional delivery path. |
| ODH-LIC-001–006 | R1 and every actual artifact/activity/delivery mode; independent mandatory publication acceptance. |
| ODH-SUP-001, ODH-SUP-003 | R1 onward; concrete maintenance/compatibility/support terms before each applicable publication. |
| ODH-SUP-002 | R4 source-profile/sampling/admission product; relevant admission discipline also applies to R1 limited Trino adoption. |
| ODH-ACC-001–004 | R1 batch/offline subsets; streaming/discovery R2; each later capability extends actual measurable evidence. |
| ODH-PLAN-001–006 | This PRD handoff and every version map/release; planning and publication remain separate. |

## Source decisions

These are the accepted PRD-level inputs and their canonical resolution comments. Their detailed rationale/evidence remain there; this document is the consolidated requirements artifact, not a copy of every discussion checkpoint.

| Input | Canonical decision |
| --- | --- |
| Product/personas/reference journeys | [Define product personas, boundaries, and reference acceptance scenarios][product] |
| Data architecture/interoperability | [Define lakehouse data architecture and interoperability contracts][architecture] |
| Distribution support/source profiles | [Define distribution support contracts and certified component combinations][support] |
| Deployment/resources | [Define deployment topology and resource-scheduling boundaries][topology] |
| Security/governance/recovery | [Define the security, governance, and recovery baseline][security], read with the scoped amendment below. |
| Management responsibilities | [Define management-plane responsibilities and lifecycle workflows][management] |
| Implementation/repository and scoped security amendment | [Choose the implementation stack and repository organization][stack] |
| Licensing/source fulfillment | [Define redistribution, licensing, and source-delivery boundaries][licensing] |
| Feature stages/CDP migration/handoff gates | [Define feature releases and version-map handoff criteria][roadmap] |
| Actual PRD output and future-map guidance | [Produce the consolidated PRD and bind release maps to its requirements][handoff] |

The [total map][total-map] indexes the underlying accepted research assets. In particular, [CDP Hive/Tez migration evidence][hive-research] constrains source/target claims, and [disaggregated Doris recovery research][doris-research] records native capture/fencing/security gaps. Research acceptance is documentation acceptance, not tested support or proof of impossibility. Actual versions, compatibility matrices, protocols, schemas, topology variants, numeric envelopes and certification results belong to applicable version maps and execution records; they are intentionally not invented here.

[product]: https://github.com/jayyu-asl/OpenDataHub/issues/2#issuecomment-6567485286
[architecture]: https://github.com/jayyu-asl/OpenDataHub/issues/8#issuecomment-6568627452
[support]: https://github.com/jayyu-asl/OpenDataHub/issues/6#issuecomment-6569630098
[topology]: https://github.com/jayyu-asl/OpenDataHub/issues/7#issuecomment-6571238244
[security]: https://github.com/jayyu-asl/OpenDataHub/issues/9#issuecomment-6572177820
[management]: https://github.com/jayyu-asl/OpenDataHub/issues/10#issuecomment-6573412824
[stack]: https://github.com/jayyu-asl/OpenDataHub/issues/11#issuecomment-6594220127
[licensing]: https://github.com/jayyu-asl/OpenDataHub/issues/13#issuecomment-6569196014
[roadmap]: https://github.com/jayyu-asl/OpenDataHub/issues/12#issuecomment-6595175158
[handoff]: https://github.com/jayyu-asl/OpenDataHub/issues/32
[total-map]: https://github.com/jayyu-asl/OpenDataHub/issues/1
[first-map]: https://github.com/jayyu-asl/OpenDataHub/issues/23
[hive-research]: https://github.com/jayyu-asl/OpenDataHub/issues/15#issuecomment-6568697528
[doris-research]: https://github.com/jayyu-asl/OpenDataHub/issues/18#issuecomment-6572428671
