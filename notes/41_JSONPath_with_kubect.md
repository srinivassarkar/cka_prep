# 🔍 Kubernetes JSONPath 

> *Think in JSON. Author in YAML. Query with a scalpel.*

---

## 0. First Principles — The Mental Model (What Never Changes)

These truths hold regardless of Kubernetes version, cloud provider, or cluster size.

1. **The API server speaks JSON.** Every Kubernetes object — Pod, Deployment, Node — is ultimately a JSON document stored in etcd (as protobuf, but deserialized to JSON for you).

2. **kubectl is a JSON client wearing a pretty table.** The default `kubectl get` output hides 95% of the actual object. `-o json` shows everything; `-o jsonpath` shows exactly what you ask for.

3. **YAML is a writing convenience, not a wire format.** You write YAML because it's readable. `kubectl` serializes it to JSON before sending it over HTTPS. The API never sees your YAML.

4. **JSONPath walks a tree.** Every path starts at `$` (root), descends with `.`, indexes arrays with `[n]` or `[*]`, and filters with `?(@.field=="value")`. That's the whole grammar you need for CKA.

5. **List responses wrap results in `.items`.** A `kubectl get pods` response is not an array — it's a `PodList` object with an `items` field. Always start with `.items[*]` when querying multiple resources.

6. **Selectors reduce noise before JSONPath extracts signal.** `-l app=web` or `--field-selector status.phase=Running` narrows the dataset *first*, making your JSONPath expression simpler and faster.

---

## 1. Reality Constraints — What Kubernetes Actually Does

### What `kubectl` does under the hood

| Phase | What Happens |
|---|---|
| **Auth** | Reads `~/.kube/config` → extracts cluster URL, CA cert, credentials |
| **Request build** | For GET: HTTPS GET to `/api/v1/namespaces/<ns>/pods`, no body. For writes: YAML → JSON, sent as HTTP body |
| **Wire format** | Always HTTPS. Reads = no body. Writes = JSON body (`Content-Type: application/json`) |
| **API server** | Auth → RBAC → Admission (bypassed for GETs) → serve from watch cache or etcd |
| **etcd storage** | Protobuf internally — but you never interact with this directly |
| **Response** | JSON or protobuf via content negotiation |
| **kubectl rendering** | Decodes to in-memory object, applies your `-o` format, prints to terminal |

### The watch cache matters

- `kube-apiserver` keeps an **in-memory watch cache** for most resources.
- `GET/LIST` calls are usually served from cache, *not* etcd — this is why your cluster doesn't fall over under heavy `kubectl get` usage.
- JSONPath runs on the **decoded in-memory object** inside kubectl. Wire format (protobuf vs JSON) is irrelevant to your query.

### What JSONPath can and cannot do

| Can ✅ | Cannot ❌ |
|---|---|
| Extract specific fields | Modify or write data |
| Wildcard over arrays `[*]` | Cross-resource joins (e.g., Pods + Services in one query) |
| Filter by field value `?(@.x=="y")` | Arithmetic or string operations |
| Loop with `{range}{end}` | Aggregations (count, sum) — use `jq` for those |
| Emit valid JSON with `-o jsonpath-as-json` | Replace `jq` for complex pipelines |

---

## 2. Decision Logic — When to Use What

### Output format decision tree

```
Need cluster data?
│
├─ Just checking one field on one object?
│   └─ kubectl get <kind> <name> -o jsonpath='{.field}'
│
├─ Same field across many objects?
│   └─ kubectl get <kind> -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
│
├─ Need to filter by field value first?
│   └─ Use --field-selector or -l BEFORE jsonpath (cheaper)
│
├─ Complex transformation, aggregation, conditional?
│   └─ kubectl get <kind> -o json | jq '...'
│
└─ Need the output as valid JSON array (e.g., pipe to jq)?
    └─ kubectl get <kind> -o jsonpath-as-json='{.items[*].metadata.name}' | jq length
```

### Format comparison table

| Format | Use When | Verbosity |
|---|---|---|
| *(default)* | Human glance, table view | Low |
| `-o wide` | Add IP/node columns | Low+ |
| `-o yaml` | Full object, authoring/debugging | High |
| `-o json` | Full object, piping to `jq` | High |
| `-o jsonpath=` | Targeted field extraction, scripting | Exact |
| `-o jsonpath-as-json=` | JSONPath result must be valid JSON | Exact |
| `-o custom-columns=` | Tabular output, multiple fields | Medium |

### JSONPath operator cheat-decision

| You want... | Operator | Example |
|---|---|---|
| A specific field | `.field` | `.metadata.name` |
| All elements of an array | `[*]` | `.spec.containers[*].image` |
| Specific array element | `[n]` | `.spec.containers[0].image` |
| Elements matching a condition | `?(@.field=="val")` | `?(@.name=="web")` |
| Iterate and format output | `{range X}{...}{end}` | `{range .items[*]}{.metadata.name}{"\n"}{end}` |
| Embed newline/tab in output | `{"\n"}` / `{"\t"}` | See examples below |
| Key containing dots/slashes | Quote it | `.metadata.annotations."checksum/config"` |

---

## 3. Internal Working — How It Actually Happens

### Full journey: `kubectl get pods -o jsonpath='{.items[0].metadata.name}'`

```
Step 1: kubectl reads ~/.kube/config
        → cluster: https://192.168.1.100:6443
        → CA: /etc/kubernetes/pki/ca.crt
        → credentials: client cert + key (or token)

Step 2: kubectl constructs the HTTPS request
        GET /api/v1/namespaces/default/pods
        Accept: application/json
        Authorization: Bearer <token>  (or client cert in TLS handshake)

Step 3: kube-apiserver receives the request
        → Authenticates (validates token / cert)
        → Authorizes (RBAC: can this user LIST pods in this namespace?)
        → Admission: skipped for GETs
        → Serves from watch cache (avoids etcd round-trip for most requests)

Step 4: Response comes back
        HTTP 200 OK
        Content-Type: application/json
        Body: { "kind": "PodList", "items": [ {...}, {...} ] }

Step 5: kubectl decodes the JSON into an in-memory Go struct

Step 6: kubectl applies your JSONPath template
        '.items[0].metadata.name'
        → navigates items array → index 0 → metadata → name
        → emits: "nginx-pod-abc123"

Step 7: kubectl writes output to stdout
```

### YAML → JSON serialization (for writes)

```
You write:                    kubectl sends:
─────────────────────────     ─────────────────────────────────────
apiVersion: v1            →   {"apiVersion":"v1",
kind: Pod                 →    "kind":"Pod",
metadata:                 →    "metadata":{
  name: my-pod            →      "name":"my-pod"
spec:                     →    },"spec":{
  containers:             →      "containers":[
  - name: web             →        {"name":"web",
    image: nginx          →         "image":"nginx"}
                                ]}}
```

The mapping is 1:1 and lossless. YAML indentation = JSON nesting. YAML `-` list items = JSON array elements.

### etcd storage reality

Objects in etcd are stored as **Protocol Buffers**, not JSON. The flow is:

```
kubectl (JSON) → API server → decode JSON → validate → encode protobuf → etcd
etcd → protobuf → API server → decode → encode JSON → kubectl
```

You never touch protobuf. This detail matters for interviews — it explains why etcd backups must be restored to the *same* Kubernetes version (protobuf schema is version-specific).

---

## 4. Hands-On — Production-Quality YAML + Commands

### The demo Pod (used throughout this guide)

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers-demo
  namespace: default
  labels:
    app: two-containers
spec:
  containers:
    - name: web
      image: nginx:1.27
      ports:
        - containerPort: 80
    - name: helper
      image: busybox:1.36
      args:
        - sh
        - -c
        - "while true; do echo two-containers running; sleep 10; done"
```

### Core JSONPath patterns — single object

```bash
# Basic field extraction
kubectl get pod two-containers-demo -o jsonpath='{.metadata.name}{"\n"}'
kubectl get pod two-containers-demo -o jsonpath='{.metadata.namespace}{"\n"}'
kubectl get pod two-containers-demo -o jsonpath='{.metadata.labels.app}{"\n"}'

# Pod IP and node placement (after scheduling)
kubectl get pod two-containers-demo -o jsonpath='{.status.podIP}{"\t"}{.spec.nodeName}{"\n"}'

# All container images (wildcard)
kubectl get pod two-containers-demo -o jsonpath='{.spec.containers[*].image}{"\n"}'

# First container's image (index)
kubectl get pod two-containers-demo -o jsonpath='{.spec.containers[0].image}{"\n"}'

# Filter: image of the 'web' container only
kubectl get pod two-containers-demo -o jsonpath='{.spec.containers[?(@.name=="web")].image}{"\n"}'

# Filter: port of the 'web' container
kubectl get pod two-containers-demo -o jsonpath='{.spec.containers[?(@.name=="web")].ports[*].containerPort}{"\n"}'
```

### Core JSONPath patterns — list responses

```bash
# All pod names in current namespace
kubectl get pods -o jsonpath='{.items[*].metadata.name}{"\n"}'

# One pod name per line (using range loop)
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'

# Name + status phase per line
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# All container images across all pods (one per line)
kubectl get pods -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}'

# Only Running pods (combine field-selector + jsonpath)
kubectl get pods --field-selector=status.phase=Running \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
```

### Deployments

```bash
# Image used in the pod template
kubectl get deploy my-deploy \
  -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'

# Desired vs ready replicas
kubectl get deploy my-deploy \
  -o jsonpath='{.spec.replicas}{" desired / "}{.status.readyReplicas}{" ready"}{"\n"}'

# Strategy type
kubectl get deploy my-deploy \
  -o jsonpath='{.spec.strategy.type}{"\n"}'
```

### Services & Endpoints

```bash
# ClusterIP and ports
kubectl get svc my-svc \
  -o jsonpath='{.spec.clusterIP}{"\t"}{.spec.ports[*].port}{"\n"}'

# NodePort (if exposed)
kubectl get svc my-svc \
  -o jsonpath='{.spec.ports[*].nodePort}{"\n"}'

# Endpoint IPs backing a Service
kubectl get endpoints my-svc \
  -o jsonpath='{range .subsets[*].addresses[*]}{.ip}{"\n"}{end}'
```

### Nodes

```bash
# All node names
kubectl get nodes -o jsonpath='{.items[*].metadata.name}{"\n"}'

# Node name + internal IP
kubectl get nodes \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.addresses[?(@.type=="InternalIP")].address}{"\n"}{end}'

# Node conditions (full status dump per node)
kubectl get nodes \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .status.conditions[*]}{.type}{"="}{.status}{" "}{end}{"\n"}{end}'

# Kernel version per node
kubectl get nodes \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.kernelVersion}{"\n"}{end}'
```

### ConfigMaps & Secrets

```bash
# ConfigMap value (key with dots must be quoted)
kubectl get cm my-config \
  -o jsonpath='{.data."app.properties"}{"\n"}'

# Secret value (base64 encoded — pipe to decode)
kubectl get secret my-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo

# All keys in a Secret's data field
kubectl get secret my-secret \
  -o jsonpath='{range .data}{"\n"}{@}{end}'
```

### Piping to jq (when JSONPath isn't enough)

```bash
# jsonpath-as-json emits valid JSON — safe to pipe
kubectl get pods -o jsonpath-as-json='{.items[*].metadata.name}' | jq length

# Full jq pipeline for complex logic
kubectl get pods -o json | \
  jq -r '.items[] | select(.status.phase=="Running") | .metadata.name'

# Join pod name with its container images
kubectl get pods -o json | \
  jq -r '.items[] | .metadata.name + " -> " + (.spec.containers[].image)'
```

---

## 5. Production Flow — Real-World Patterns

### Pattern 1: Audit all image versions across a namespace

```bash
#!/bin/bash
# List every pod, container, and image in a namespace — great for CVE audits
kubectl get pods -n production \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.name}{"="}{.image}{" "}{end}{"\n"}{end}'
```

### Pattern 2: Health-check script using JSONPath

```bash
#!/bin/bash
# Exit non-zero if any Deployment has mismatched desired/ready replicas
kubectl get deploy -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.replicas}{" "}{.status.readyReplicas}{"\n"}{end}' \
  | awk '$2 != $3 { print "DEGRADED: " $1; exit 1 }'
```

### Pattern 3: Find all pods NOT on a specific node

```bash
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}' \
  | grep -v "node-01"
```

### Pattern 4: Extract TLS certificate expiry from a Secret

```bash
kubectl get secret my-tls -o jsonpath='{.data.tls\.crt}' \
  | base64 -d \
  | openssl x509 -noout -dates
```

### Pattern 5: Cross-namespace image inventory

```bash
# All images across ALL namespaces
kubectl get pods --all-namespaces \
  -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{range .spec.containers[*]}{.image}{" "}{end}{"\n"}{end}' \
  | sort | column -t
```

### When to abandon JSONPath for jq

| Situation | Use |
|---|---|
| Single field, one object | `jsonpath` |
| Tabular multi-field output | `jsonpath` with `{range}` |
| Filter arrays by field | Either — JSONPath `?()` or `jq select()` |
| Count, sum, complex conditionals | `jq` always |
| Output must be valid JSON | `-o jsonpath-as-json` then `jq` |
| Cross-referencing two resource types | `jq` or shell loops |

---

## 6. Mistakes — What Actually Breaks in Real Systems

### Mistake 1: Forgetting `.items[*]` on list responses

**Symptom:** Empty output or `error: template: output:1:2: unknown field`

**Root cause:** `kubectl get pods` returns a `PodList` object. The pods live in `.items`. You cannot do `.metadata.name` at the top level of a list response.

```bash
# ❌ Wrong — tries to get .metadata.name of the PodList itself
kubectl get pods -o jsonpath='{.metadata.name}'

# ✅ Correct
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
```

### Mistake 2: Shell variable expansion eating your JSONPath

**Symptom:** `bash: .items[*].metadata.name: bad substitution` or garbled output

**Root cause:** Double quotes allow `$` expansion; `[*]` is interpreted as glob

```bash
# ❌ Wrong — double quotes let shell interpret $ and *
kubectl get pods -o jsonpath="{.items[*].metadata.name}"

# ✅ Correct — single quotes protect the entire template
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
```

### Mistake 3: Querying a field that doesn't exist on all objects

**Symptom:** Some lines in range output are blank or compressed together

**Root cause:** Not all pods have `.spec.nodeName` (Pending pods), not all services have `.spec.ports[*].nodePort` (ClusterIP services)

**Fix:** Use `--field-selector` to pre-filter, or handle in `jq` with `// "N/A"` default operator:
```bash
kubectl get pods -o json | \
  jq -r '.items[] | [.metadata.name, (.spec.nodeName // "Pending")] | @tsv'
```

### Mistake 4: Dot-separated keys not quoted

**Symptom:** Empty output for annotation queries

**Root cause:** `kubectl get pod -o jsonpath='{.metadata.annotations.checksum/config}'` — the `/` breaks the path

```bash
# ❌ Wrong
kubectl get pod my-pod -o jsonpath='{.metadata.annotations.checksum/config}'

# ✅ Correct — quote keys with special characters
kubectl get pod my-pod -o jsonpath='{.metadata.annotations."checksum/config"}'
```

### Mistake 5: Expecting jsonpath output to be valid JSON

**Symptom:** `jq: parse error` when piping JSONPath output to jq

**Root cause:** `-o jsonpath` does NOT wrap results in JSON. It just concatenates raw values.

```bash
# ❌ Breaks — 'my-pod my-pod2' is not valid JSON
kubectl get pods -o jsonpath='{.items[*].metadata.name}' | jq '.'

# ✅ Use jsonpath-as-json to get a valid JSON array
kubectl get pods -o jsonpath-as-json='{.items[*].metadata.name}' | jq '.'
```

### Mistake 6: Using JSONPath on a single object but treating it as a list

**Symptom:** `error: .items is null`

**Root cause:** `kubectl get pod <specific-name>` returns a single Pod object, not a PodList. No `.items`.

```bash
# ❌ Wrong — single pod has no .items
kubectl get pod my-pod -o jsonpath='{.items[0].metadata.name}'

# ✅ Correct — access fields directly
kubectl get pod my-pod -o jsonpath='{.metadata.name}'
```

---

## 7. Interview Answers — Verbatim-Ready

### Q: Why does Kubernetes use JSON internally if we write YAML?

"YAML is the authoring format — it's human-readable, supports comments, and is less noisy than JSON for large files. But when kubectl submits a manifest to the API server, it serializes the YAML to JSON because JSON is the universal wire format for HTTP APIs. The API server stores objects in etcd as Protocol Buffers for efficiency, but it exposes and accepts them as JSON over HTTPS. So YAML is the developer convenience; JSON is what actually moves across the network and gets stored."

### Q: What is JSONPath in the context of kubectl?

"JSONPath is a query language for navigating JSON documents. In kubectl, the `-o jsonpath` flag applies a JSONPath expression to the decoded API response object and extracts exactly the fields you specify. It's how you go from a full 200-line JSON object down to just the one IP address or image tag you actually care about. The key operators are dot notation for field access, array wildcards with `[*]`, index access with `[n]`, and filter expressions like `?(@.name=="web")` for matching specific elements. kubectl also adds `{range}{end}` loops for iterating over collections, which isn't in standard JSONPath."

### Q: What's the difference between `-o jsonpath` and `-o jsonpath-as-json`?

"Both apply a JSONPath expression, but `-o jsonpath` concatenates the raw results as a string — so if you extract multiple pod names, you get 'pod1 pod2 pod3' with no surrounding structure. That's not valid JSON, so you can't pipe it to `jq` directly. `-o jsonpath-as-json` wraps the result in proper JSON syntax — an array or object — so the output is parseable by `jq` or any other JSON tool. I use `jsonpath-as-json` whenever I need to chain with other JSON-aware tooling."

### Q: How does kubectl talk to the API server?

"kubectl reads the kubeconfig file to get the cluster endpoint URL, the CA certificate for TLS verification, and credentials — either a client certificate, a bearer token, or both. All traffic goes over HTTPS. For read operations like GET and LIST, kubectl sends a plain HTTPS GET with no request body and the appropriate Accept header. For writes — create, update, patch — kubectl serializes the manifest to JSON and sends it as the request body with `Content-Type: application/json`. The API server then authenticates the request, checks RBAC authorization, runs admission controllers if applicable, and either returns data from its in-memory watch cache or queries etcd."

### Q: What's the `.items` field and why do you need it?

"When you run a list command like `kubectl get pods`, the API server doesn't return a raw JSON array. It returns a Kubernetes `PodList` object, which is itself a JSON object with a `kind` of `PodList`, `metadata`, and an `items` array that contains the actual Pod objects. So in your JSONPath expression, you have to start with `.items[*]` to address the individual pods — you're navigating into the list wrapper first, then into the array. If you forget `.items` and try to access `.metadata.name` at the top level, you get the name of the PodList itself, which is usually empty, not the pod names you wanted."

### Q: When would you use `jq` instead of JSONPath?

"I reach for `jq` when the query gets complex — specifically when I need aggregation like counting or summing, conditional logic that spans multiple fields, cross-referencing data from different objects, or when the output needs to be valid JSON for downstream processing. For simple field extraction and tabular output, JSONPath with `{range}` loops is faster to type and doesn't require piping. My general rule is: one or two fields from one or many objects → JSONPath; anything involving transformation, aggregation, or complex filtering → pipe to `jq`."

---

## 8. Debugging — Fast Diagnosis Paths

### Symptom → Cause → Fix

```
Problem: kubectl -o jsonpath returns nothing (empty output)
│
├─ Is this a list command (get pods, get nodes)?
│   └─ YES → Did you use .items[*]?
│       ├─ NO  → Add .items[*] before the field path
│       └─ YES → Does the field exist on all objects?
│                └─ Use --field-selector to pre-filter
│
├─ Is this a single object command (get pod <name>)?
│   └─ Check: does the field actually exist on this object?
│       kubectl get pod <name> -o json | grep "<fieldname>"
│
└─ Does the key contain dots or slashes?
    └─ Quote it: .metadata.annotations."my.key/with-slash"
```

```
Problem: jsonpath output looks garbled / all on one line
│
├─ Missing {"\n"} after each range iteration
│   └─ Add: {range .items[*]}{.field}{"\n"}{end}
│
└─ Shell ate special characters
    └─ Switch double quotes to single quotes around template
```

```
Problem: jq parse error when piping jsonpath output
│
└─ kubectl -o jsonpath output is NOT valid JSON
    └─ Switch to: kubectl ... -o jsonpath-as-json='{...}' | jq '.'
```

### Diagnostic command toolkit

```bash
# Step 1: Dump the full object to understand the shape
kubectl get pod <name> -o json | less
kubectl get pod <name> -o json | jq 'keys'  # top-level fields
kubectl get pod <name> -o json | jq '.spec | keys'  # spec sub-fields

# Step 2: Validate your path incrementally
kubectl get pod <name> -o jsonpath='{.spec}'  # get spec first
kubectl get pod <name> -o jsonpath='{.spec.containers}'  # then containers
kubectl get pod <name> -o jsonpath='{.spec.containers[0]}'  # then index

# Step 3: Check a field exists before querying it
kubectl get pod <name> -o json | jq '.spec.containers[0].resources'

# Step 4: Verify filter expressions
# Equivalent jq filter to test before putting in jsonpath
kubectl get pod <name> -o json | \
  jq '.spec.containers[] | select(.name=="web") | .image'

# Step 5: Debug range loops — add a separator to see boundaries
kubectl get pods \
  -o jsonpath='{range .items[*]}[{.metadata.name}]{"\n"}{end}'
```

### Quick validation for CKA exam pressure

```bash
# "Does my jsonpath work?"
# Test on one object first, then expand to list
kubectl get pod <any-pod> -o jsonpath='{.metadata.name}{"\n"}'  # works?
kubectl get pods -o jsonpath='{.items[0].metadata.name}{"\n"}'  # list version works?
kubectl get pods -o jsonpath='{.items[*].metadata.name}{"\n"}'  # all of them?
```

---

## 9. Kill Switch — 10-Second Recall

```
$ = root
. = child field
[*] = all array elements
[0] = first element
?(@.name=="x") = filter where name equals x
{range .items[*]}{...}{"\n"}{end} = loop over list

LIST RESPONSE → always need .items[*]
SINGLE OBJECT → no .items

KEY WITH DOTS/SLASHES → quote it: ."my.key"
WANT VALID JSON OUTPUT → use -o jsonpath-as-json
COMPLEX LOGIC → kubectl get ... -o json | jq '...'
```

**The five operators that cover 90% of CKA exam questions:**

| Operator | What it does |
|---|---|
| `.field` | Navigate to a field |
| `[*]` | Select all array elements |
| `[0]` | Select first element |
| `?(@.f=="v")` | Filter array elements |
| `{range}{end}` | Loop (kubectl extension) |

---

## 10. Appendix — Quick Reference Card

### Essential commands

```bash
# Single object, single field
kubectl get pod <name> -o jsonpath='{.metadata.name}'

# List, all names
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# List, one per line (range loop)
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'

# Filter by label first, then extract
kubectl get pods -l app=web -o jsonpath='{.items[*].metadata.name}'

# Filter by field selector first, then extract
kubectl get pods --field-selector=status.phase=Running \
  -o jsonpath='{.items[*].metadata.name}'

# Multi-field tabular output
kubectl get pods \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\t"}{.spec.nodeName}{"\n"}{end}'

# Emit valid JSON (pipe-safe)
kubectl get pods -o jsonpath-as-json='{.items[*].metadata.name}' | jq length

# Secret value decoded
kubectl get secret <name> -o jsonpath='{.data.<key>}' | base64 -d; echo

# Annotation with special key
kubectl get pod <name> -o jsonpath='{.metadata.annotations."kubectl.kubernetes.io/last-applied-configuration"}'
```

### JSONPath grammar

```
$               root
.field          child access
..field         recursive descent (rarely used in kubectl)
[n]             array index (0-based)
[*]             all array elements
[a,b]           multiple indices
[a:b]           slice
?(@.f=="v")     filter: select elements where field equals value
?(@.f>"v")      filter: greater than
@               current element (inside filter)
```

### kubectl JSONPath extensions (not standard JSONPath)

```
{"\n"}          emit newline
{"\t"}          emit tab
{range X}{end}  iterate over X
```

### YAML ↔ JSON mapping cheatsheet

| YAML | JSON equivalent |
|---|---|
| `key: value` | `"key": "value"` |
| Indented block | `{}` nested object |
| `- item` list | `["item"]` array |
| `# comment` | *(no equivalent)* |
| `key: 'string'` | `"key": "string"` |
| `key: 42` | `"key": 42` |
| `key: true` | `"key": true` |

### Common field paths reference

```
# Metadata
.metadata.name
.metadata.namespace
.metadata.labels.<key>
.metadata.annotations."<key>"
.metadata.creationTimestamp

# Pod spec
.spec.nodeName
.spec.containers[*].name
.spec.containers[*].image
.spec.containers[*].ports[*].containerPort
.spec.containers[?(@.name=="<c>")].image
.spec.initContainers[*].image
.spec.serviceAccountName
.spec.volumes[*].name

# Pod status
.status.phase
.status.podIP
.status.hostIP
.status.conditions[*].type
.status.containerStatuses[*].ready
.status.containerStatuses[*].restartCount

# Deployment
.spec.replicas
.spec.strategy.type
.spec.template.spec.containers[*].image
.status.readyReplicas
.status.updatedReplicas

# Service
.spec.type
.spec.clusterIP
.spec.ports[*].port
.spec.ports[*].nodePort
.spec.ports[*].targetPort
.spec.selector

# Node
.status.addresses[?(@.type=="InternalIP")].address
.status.nodeInfo.kernelVersion
.status.nodeInfo.kubeletVersion
.status.conditions[?(@.type=="Ready")].status
.spec.taints[*].key
```

---
