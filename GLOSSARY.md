# OpenDataHub

A reusable data platform product for private enterprise environments, packaging component capabilities for customer-specific deployments.

## Language

**Data Hub**:
The reusable platform product delivered into different customers' private environments, rather than a single customer's dataset or application.
_Avoid_: Log Archive System (as a synonym for the entire platform)

**Platform Distribution**:
A versioned collection of component capabilities with an explicit support boundary, not all of which must be deployed by every customer.
_Avoid_: Full Installation Bundle (as a synonym for the distribution)

**Component Bundle**:
A separately versioned delivery unit for one platform component, selected as part of a Platform Distribution. Its activation is distinct from the version actually running and does not itself establish a safe data or metadata rollback.
_Avoid_: Cloudera Parcel (when implying Cloudera Manager compatibility)

**Customer Deployment**:
A selection of platform capabilities deployed to meet a particular customer's needs; a concrete use of a Platform Distribution.
_Avoid_: Platform Distribution (when referring to a single customer's deployment)

**Customer Organization**:
The organization that owns a Customer Deployment and its data, distinct from the people who deliver, operate, or use it.
_Avoid_: User (when referring to the customer organization)

**Platform Identity**:
The canonical identity of a person or workload within a Customer Organization, to which protocol-specific accounts and principals are explicitly mapped across a Customer Deployment. A shared identity does not imply identical permissions across environments or services.
_Avoid_: Protocol Account (when referring to the cross-service identity)

**Deployment Environment**:
A subdivision of a Customer Deployment, such as development, test or production, with its own data-plane services and workload resource pools. It may share the customer's management plane but is not merely a label on shared data-plane services.
_Avoid_: Environment Label (when implying an independent deployment boundary)

**Managed Cluster**:
A registered group of Managed Services belonging to one Deployment Environment and addressable for cluster-scoped management.
_Avoid_: YARN Queue (as a synonym for a Managed Cluster)

**Managed Service**:
A deployed platform component whose lifecycle is within the Data Hub's management responsibility, whether newly deployed or explicitly adopted from an existing environment.
_Avoid_: Connected Service (when implying lifecycle-management responsibility)

**Service Role**:
A named type of operational responsibility within a Managed Service, such as a NameNode or DataNode, distinct from an authorization role.
_Avoid_: Permission Role (when referring to a service's operational responsibility)

**Service Role Instance**:
A concrete deployment of a Service Role within one Managed Service on a host. It need not be a long-running process.
_Avoid_: Managed Service (when referring to only one deployed role)

**Role Configuration Group**:
A shared configuration baseline for instances of the same Service Role within a Managed Service.
_Avoid_: Host Group, Authorization Group

**Service Adoption**:
The explicit entry of an existing service into the Data Hub's lifecycle-management responsibility under supported compatibility conditions.
_Avoid_: Service Connection (when implying adoption rather than access)

**Service Offboarding**:
The explicit exit of a Managed Service from the Data Hub's lifecycle-management responsibility through an agreed handover of control and operating responsibility. It does not itself imply stopping the service or deleting its data.
_Avoid_: Service Deletion (when referring only to withdrawal from management)

**Management Operation**:
An authorized, scoped management action together with its attributable execution history, including partial or unresolved outcomes. It is not necessarily a single command or an atomic transaction.
_Avoid_: Shell Command (when referring to the complete platform-managed action)

**Shared Analytics Layer**:
The Data Hub's common persisted analytical datasets intended for reuse across supported engines, distinct from engine-specific serving copies.
_Avoid_: Unified Storage (when it could imply a single physical backend or prohibit serving copies)

**Serving Copy**:
An engine-specific persisted derivative maintained for a declared analytical workload, with a distinct lifecycle from the shared dataset or source events from which it is produced.
_Avoid_: Shared Dataset (when referring to a separately maintained serving copy)

**Data Migration**:
The controlled transfer of selected data and required metadata from a source into a Customer Deployment, under an explicit preservation boundary and cutover of accepted use. It does not itself adopt the source service into lifecycle management or require table-format conversion.
_Avoid_: Service Adoption, Replication (when implying that synchronization alone completes a migration)

**Hive Migration Compatibility**:
The Data Hub's bounded commitment to preserve selected existing Hive tables and workload behavior during migration, without mandatory table-format conversion.
_Avoid_: CDP Equivalence (when implying compatibility with every CDP component, version, or extension)

**Certified Path**:
A specific platform capability or lifecycle path whose declared component combination, operations, prerequisites, and boundaries have passed the required verification. Its certification does not extend to arbitrary uses or combinations of the same components.
_Avoid_: Certified Component (when implying certification of every integration involving that component)

**Supported Dependency**:
A prerequisite included in the support boundary of a selected platform capability, without thereby becoming an independently offered platform capability.
_Avoid_: Unmanaged Prerequisite (when referring to a dependency the platform has undertaken to support)

**Source Compatibility Profile**:
A bounded class of source services, table characteristics, and workload behaviors eligible for a declared migration path under common compatibility conditions. It does not imply compatibility with every build in a vendor release family.
_Avoid_: Supported CDP Version (when implying that a vendor version alone defines migration compatibility)
