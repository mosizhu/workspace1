# Mopheus IT Asset Manifest Specification

This document provides technical specifications and axioms for AI agents to synthesize, validate, and ingest Mopheus declarative asset manifests (`AssetManifest`).

---

## 1. Architecture & Design Philosophy

Mopheus models enterprise IT systems as an **ontology knowledge graph** partitioned into **Concept Metamodels (TBox)** and **Asset Instances (ABox)**:
- **TBox Strong Schema Validation**: 17 built-in canonical concepts (`AssetConcept`). Each concept enforces a strict JSONSchema (`attributes_schema`) and an allowed-relation whitelist (`allowed_relations`).
- **ABox Strict Verification**: Every asset instance (`Asset`) and relation edge (`AssetRelation`) must strictly comply with its corresponding concept definition.
- **Agent-Native Synthesis**: The `AssetManifest` is designed with strict schema validation and topology axioms. It is **intended for programmatic synthesis by AI agents**, NOT for manual authoring by human engineers.

---

## 2. Top-Level Manifest Structure

Manifests can be presented in YAML or JSON format:

```yaml
version: mopheus.io/v1alpha1
kind: AssetManifest
metadata:
  workspace: <workspace-slug-or-uuid>  # Optional: target workspace identifier
  environment: prod                   # Default fallback environment (prod|staging|test|dev)
assets:
  - <Asset Object>
relations:
  - <AssetRelation Object>
```

### 2.1 Asset Object Specification

| Field | Type | Required | Description & Constraints |
| :--- | :--- | :---: | :--- |
| `id` | `string` | **Yes** | Globally unique identifier within the workspace. Recommended pattern: lowercase concept prefix, e.g. `host:node-1`, `svc:order-gw`. |
| `concept` / `conceptName` | `string` | **Yes** | Must match one of the 17 authoritative concept names (case-insensitive, PascalCase recommended, e.g. `Host`, `Service`). |
| `name` | `string` | No | Human-readable display name. Defaults to `id` if omitted. |
| `aliases` | `[]string` | No | Short names, abbreviations, or historical aliases for fuzzy resolution via `mop asset resolve`. |
| `status` | `string` | No | Lifecycle status: `active` (default), `draining`, `offline`, `unhealthy`. |
| `environment` | `string` | No | Deployment environment: `prod` (default), `staging`, `test`, `dev`. |
| `attributes` | `object` | No | **Objective technical facts**. Must strictly conform to the concept's JSONSchema. Undeclared properties are rejected if `additionalProperties: false`. |
| `labels` | `map[string]string` | No | **Organizational governance metadata**. Free-form key-value pairs, e.g. `owner_team`, `tier`, `oncall`, `department`. |

---

### 2.2 Relation Object Specification

| Field | Type | Required | Description & Constraints |
| :--- | :--- | :---: | :--- |
| `subject` / `subjectId` | `string` | **Yes** | Origin asset ID (Source entity). Must exist in the workspace or in the current manifest `assets`. |
| `predicate` | `string` | **Yes** | **One of the 7 canonical predicates**. Custom predicates are strictly prohibited. |
| `object` / `objectId` | `string` | **Yes** | Target asset ID (Destination entity). Must exist in the workspace or in the current manifest `assets`. |
| `attributes` | `object` | No | Edge properties (e.g. network port, replication mode). |

> [!CAUTION]
> **Topology Axioms**:
> 1. **No Self-Referencing**: `subject != object`. Self-loops are rejected by the validator.
> 2. **Metamodel Axiom Conformance**: The subject's concept `allowed_relations` rule must explicitly permit the given `predicate` to target the object's concept.

---

## 3. The 7 Canonical Topology Predicates

| Predicate | Direction (Subject $\to$ Object) | Meaning & Valid Combinations |
| :--- | :--- | :--- |
| **`contains`** | Whole $\to$ Part | Structural composition:<br>`Application -> Service/Database/LoadBalancer/DNSRecord`<br>`Database -> DatabaseInstance`<br>`Cluster -> Host/StoragePool`<br>`StoragePool -> StorageVolume`<br>`Host -> IPAddress/StorageVolume` |
| **`runs_on`** | Workload $\to$ Host | Hosting and execution:<br>`Service -> Host/Container/Cluster`<br>`Container -> Host/Cluster`<br>`DatabaseInstance -> Host/Container`<br>`DatabaseComponent -> Host/Container`<br>`VirtualIP -> Host/NetworkDevice`<br>`LoadBalancer -> Host/Container/NetworkDevice` |
| **`routes_to`** | Ingress $\to$ Destination | Traffic forwarding and routing:<br>`DNSRecord -> LoadBalancer/VirtualIP/IPAddress/Host`<br>`LoadBalancer -> Service/DatabaseInstance/VirtualIP/Host`<br>`VirtualIP -> DatabaseInstance/Service/Host`<br>`DatabaseComponent -> DatabaseInstance/DatabaseComponent` |
| **`depends_on`** | Caller $\to$ Dependency | Runtime functional or business dependency:<br>`Service -> Service/Database/MessageQueue/VirtualIP`<br>`Database -> Cluster/StoragePool`<br>`Host -> NetworkDevice/StoragePool` |
| **`replicates_to`** | Primary $\to$ Standby | Stateful data replication:<br>`DatabaseInstance -> DatabaseInstance` |
| **`clustered_with`** | Peer $\leftrightarrow$ Peer | High-availability peer-to-peer pairing:<br>`Host -> Host`<br>`DatabaseInstance -> DatabaseInstance`<br>`DatabaseComponent -> DatabaseComponent`<br>`NetworkDevice -> NetworkDevice` |
| **`manages`** | Control Plane $\to$ Target | Control plane orchestration and governance:<br>`Cluster -> Database/VirtualIP/Service` |

---

## 4. Concept Technical Attributes Schema Summary

Before synthesizing manifests, agents can query live schemas via `mop asset concept get <Concept>`. Below is a reference summary for core concepts:

### 4.1 Host (Compute)
- **Required attributes**:
  - `management_ip`: string (Primary management IP address)
- **Optional attributes**:
  - `role`: string enum `["worker", "control_plane", "gateway", "standalone"]`
  - `ips`: array of string (List of network interfaces)
  - `zone`: string (Datacenter / Availability zone)
  - `rack`: string (Physical rack location)
  - `credential_ref`: string (SSH credential URI, e.g. `ssh://user@ip:22?key=keyName`)
  - `machine_id`: string (UUID from `/etc/machine-id` for agent local machine short-circuit)

### 4.2 Database (Database)
- **Required attributes**:
  - `engine`: string (DBMS engine, e.g. `postgresql`, `mysql`, `oracle`, `tidb`, `redis`, `elasticsearch`, `mogdb`, `yashandb`, `oceanbase`, etc.)
  - `db_mode`: string enum `["single", "cluster", "distributed", "primary_standby", "rac"]` (Canonical architecture modes: `single`, `cluster`, `distributed`. Backward-compatible aliases: `primary_standby`, `rac`)
- **Optional attributes**:
  - `version`: string
  - `credential_ref`: string
  - `machine_id`: string

### 4.3 DatabaseInstance (Database)
- **Required attributes**:
  - `port`: integer `[1..65535]`
- **Optional attributes**:
  - `ip`: string
  - `role`: string enum `["primary", "standby", "read_replica", "witness"]`
  - `instance_name`: string
  - `credential_ref`: string (Connection or credential reference. For SwissQL integration, points to a SwissQL profile via `swissql://<profile-id>`, or SSH/Vault URI)
  - `machine_id`: string

### 4.4 Service (Business)
- **Required attributes**: None
- **Optional attributes**:
  - `codebase_repo`: string (Git repository URL)

### 4.5 Cluster (Platform)
- **Required attributes**:
  - `cluster_type`: string enum `["oracle_crs", "k8s_control_plane", "pacemaker", "patroni_etcd", "keepalived_vrrp"]`
- **Optional attributes**:
  - `api_endpoint`: string
  - `credential_ref`: string
  - `machine_id`: string

### 4.6 VirtualIP (Network)
- **Required attributes**:
  - `ip`: string
  - `role`: string enum `["vip", "scan", "floating"]`

### 4.7 LoadBalancer (Network)
- **Required attributes**:
  - `lb_type`: string enum `["nginx", "haproxy", "f5", "cloud_slb", "envoy", "apisix", "kong", "k8s_ingress"]`
- **Optional attributes**:
  - `ip`: string
  - `ports`: array of integer

### 4.8 DNSRecord (Network)
- **Required attributes**:
  - `domain`: string
  - `record_type`: string enum `["A", "CNAME", "AAAA", "PTR"]`

### 4.9 NetworkDevice (Network)
- **Required attributes**:
  - `device_type`: string enum `["switch_core", "switch_tor", "router", "firewall", "gateway"]`
  - `mgmt_ip`: string
- **Optional attributes**:
  - `credential_ref`: string
  - `machine_id`: string

---

## 5. Agent Ingestion Standard Operating Procedure (SOP)

1. **Verify Metamodel Constraints**:
   Run `mop asset concept get <Concept>` if uncertain about property types or allowed relations.
2. **Distinguish Facts from Governance**:
   - Place physical parameters (IP, port, engine, version, credentials) in `attributes`.
   - Place ownership and alerting metadata (owner_team, tier, oncall) in `labels`.
3. **Establish Hosting Relations**:
   Ensure logical services and databases point to physical `Host` or `Container` nodes via `runs_on` edges. This is essential for subsequent `mop ssh` target resolution.
4. **Apply Manifest**:
   ```bash
   mop asset apply -f <path-to-manifest.yaml>
   ```
5. **Verify Ingested Graph**:
   ```bash
   mop asset list
   mop asset inspect <id>
   ```
