---
name: mopheus-assets
description: "排查生产故障根因、分析服务依赖、评估爆炸半径、导入或程序化创建IT资产清单(manifest)、查询服务与基础设施部署关系时使用。/ Use when diagnosing production incidents, tracing dependencies, analyzing blast radius, creating or importing declarative IT asset manifests, or resolving infrastructure topology."
user-invocable: true
allowed-tools: Bash(mop asset *), Bash(mopheus asset *)
features:
  - asset
---

## Mopheus Workflow Return

This skill is a sub-procedure inside a larger Mopheus agent run. Its completion language ("done", "finished", "task complete") applies only to the skill's own steps.

After the skill's substantive work is complete, return to the runtime Workflow in the main instructions and continue with the next required step, including any final comment, metadata update, label update, status transition, or user-facing response.


# IT Asset Ontology & Topology Analysis (`mopheus-assets`)

Use this skill when investigating production alerts, tracing root causes across application architectures, identifying blast radius, inspecting CMDB infrastructure relationships, or declaring and importing IT asset manifests.

Mopheus models IT systems as an **ontology knowledge graph** governed by strong metamodels (TBox) and typed directed relations (ABox). Every asset is an explicit entity instance with a concept class, attributes, and canonical relationships.

## 1. Core Principles

1. **Entity Resolution First**: Do not guess hostnames, service names, or database instances. Always resolve human-friendly names or alert identifiers to canonical Asset IDs using `mop asset resolve`.
2. **Directional Topology Awareness**:
   - **Downstream** (`app -> service -> db -> host`): Use when determining architecture composition, deployments, and dependency propagation.
   - **Upstream** (`host -> db -> service -> app`): Use when diagnosing infrastructure failure impact (e.g. "If host-01 fails, which applications go down?").
   - **Both**: Comprehensive blast radius and boundary mapping.
3. **Deterministic Causal Path**: Use `mop asset path` to find the shortest causal chain between a symptom entity (e.g., failed service) and a suspect root cause entity (e.g., degraded physical node).
4. **Agent-Native Declarative Ingestion**: The `AssetManifest` YAML structure has strict schema validation and topology axioms. It is designed for AI agents to synthesize programmatically after inspecting concepts, NOT for human manual handcrafting. Always consult the bundled references before synthesizing:
   - Specification: `references/manifest-spec.md`
   - Canonical Template: `references/manifest-template.yaml`

## 2. Standard Operating Procedures (SOP)

### Step 1: Entity Resolution & Inspection
When given an entity name, alias, IP, or service slug:
```bash
# Resolve exact asset identity from name or alias
mop asset resolve "order-gateway"

# Inspect detailed attributes, labels, and concept schema
mop asset inspect "svc:order-gateway"
```

### Step 2: Topology Drill-Down & Dependency Tracing
To understand the environment around the asset:
```bash
# Downstream dependencies (what does this asset depend on or run on?)
mop asset topology "svc:order-gateway" --depth 3 --direction downstream

# Upstream blast radius (what depends on this infrastructure node?)
mop asset topology "host:db-node-01" --depth 3 --direction upstream

# Full bidirectional context
mop asset topology "db:settle-oracle-rac" --depth 2 --direction both
```

### Step 3: Causal Root Cause Path Finding
When an incident occurs between two correlated components:
```bash
# Find shortest causal path from symptom to root cause
mop asset path --from "svc:order-gateway" --to "host:db-node-01" --max-depth 5
```

### Step 4: Declarative Asset Manifest Creation & Ingestion
When registering, importing, or updating IT assets and relationships:
1. **Inspect Concept Metamodel**: Check required attributes and allowed relations:
   ```bash
   mop asset concept get Host
   mop asset concept get Service
   mop asset concept get Database
   ```
2. **Read Reference Template**: Consult `references/manifest-template.yaml` and `references/manifest-spec.md` for exact YAML syntax.
3. **Synthesize Manifest YAML**: Programmatically generate the YAML file (e.g. `/tmp/asset-manifest.yaml`). Ensure:
   - Physical nodes (`Host`, `DatabaseInstance`) specify `credential_ref: "ssh://user@ip:port?key=keyName"` when remote access is required.
   - Logical services (`Service`, `Database`) link to physical nodes via `runs_on` edges.
   - Relations only use the 7 canonical predicates (`contains`, `runs_on`, `routes_to`, `depends_on`, `replicates_to`, `clustered_with`, `manages`).
4. **Apply Manifest**:
   ```bash
   mop asset apply -f /tmp/asset-manifest.yaml
   ```
5. **Verify Ingested Assets**:
   ```bash
   mop asset list --env prod
   mop asset inspect host:k8s-worker-01
   ```

### Step 5: Architecture Diagram Visualization
When presenting or reviewing system architecture:
```bash
# View L1 workspace application architecture overview table
mop asset diagram

# Generate L2 detailed 4-tier architecture spec as Archify JSON
mop asset diagram --app "app:mopheus" -o json

# Render standalone interactive Archify HTML diagram
mop asset diagram --app "app:mopheus" --render -f mopheus-arch.html
```
