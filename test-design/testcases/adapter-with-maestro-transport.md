# Feature: Adapter Framework - Maestro Transportation Layer

## Table of Contents

1. [Adapter can create ManifestWork and report status via Maestro transport](#test-title-adapter-can-create-manifestwork-and-report-status-via-maestro-transport)
2. [Adapter can skip ManifestWork operation when generation is unchanged](#test-title-adapter-can-skip-manifestwork-operation-when-generation-is-unchanged)
3. [Adapter can route ManifestWork to correct consumer based on targetCluster](#test-title-adapter-can-route-manifestwork-to-correct-consumer-based-on-targetcluster)
4. [Adapter can handle Maestro server unavailability gracefully](#test-title-adapter-can-handle-maestro-server-unavailability-gracefully)
5. [ManifestWork apply fails when targeting unregistered consumer](#test-title-manifestwork-apply-fails-when-targeting-unregistered-consumer)
6. [Main discovery fails when ManifestWork name is wrong](#test-title-main-discovery-fails-when-manifestwork-name-is-wrong)
7. [Nested discovery returns empty when criteria match nothing in manifests](#test-title-nested-discovery-returns-empty-when-criteria-match-nothing-in-manifests)
8. [Post-action fails when status API is unreachable or returns error](#test-title-post-action-fails-when-status-api-is-unreachable-or-returns-error)

---

## Environment Setup

Before running these tests, deploy the full HyperFleet stack on a dedicated GKE cluster. The following Make targets from `hyperfleet-infra` are used:

```bash
# 1. Create GKE cluster
make install-terraform TF_ENV=dev-{name}

# 2. Get kubectl credentials
gcloud container clusters get-credentials hyperfleet-dev-{name} \
  --zone us-central1-a --project hcm-hyperfleet

# 3. Generate Helm values from Terraform outputs
make tf-helm-values TF_ENV=dev-{name}

# 4. Deploy Maestro stack
make install-maestro
# Note: You may need to manually install OCM CRDs if the Helm chart CRD installation fails:
#   kubectl apply -f https://raw.githubusercontent.com/open-cluster-management-io/api/main/work/v1/0000_00_work.open-cluster-management.io_manifestworks.crd.yaml
#   kubectl apply -f https://raw.githubusercontent.com/open-cluster-management-io/api/main/work/v1/0000_01_work.open-cluster-management.io_appliedmanifestworks.crd.yaml
#   kubectl rollout restart deployment/maestro-agent -n maestro

# 5. Create Maestro consumer (represents a target cluster)
make create-maestro-consumer MAESTRO_CONSUMER=cluster1

# 6. Deploy HyperFleet API
make install-api

# 7. Deploy Sentinels
make install-sentinels

# 8. Deploy Maestro transport adapter
# The adapter name here must match ADAPTER_NAME below.
# If using a different adapter (e.g., cl-maestro), update both accordingly.
make install-adapter2

# 9. Set test variables
export ADAPTER_NAME='adapter2'
export MAESTRO_CONSUMER='cluster1'
export API_URL='http://localhost:8000'

# 10. Port-forward HyperFleet API for local access
kubectl port-forward -n hyperfleet svc/hyperfleet-api 8000:8000 &
```

---

## Test Title: Adapter can create ManifestWork and report status via Maestro transport

### Description

This test validates the complete Maestro transport happy path: creating a cluster via the HyperFleet API triggers the adapter to create a ManifestWork (resource bundle) on the Maestro server, the Maestro agent applies the ManifestWork content to the target cluster (verified via kubectl), the adapter discovers the ManifestWork and its nested sub-resources via statusFeedback, evaluates post-processing CEL expressions, and reports the final status back to the HyperFleet API.

---

| **Field** | **Value** |
|-----------|-----------|
| **Pos/Neg** | Positive |
| **Priority** | Tier0 |
| **Status** | Draft |
| **Automation** | Automated |
| **Version** | MVP |
| **Created** | 2026-02-12 |
| **Updated** | 2026-03-02 |

---

### Preconditions

1. HyperFleet API and Sentinel services are deployed and running successfully
2. Maestro is deployed and running successfully with an active agent
3. At least one Maestro consumer is registered (e.g., `${MAESTRO_CONSUMER}`)
4. Adapter is deployed in Maestro transport mode (`transport.client: "maestro"`)
5. Adapter task config defines nestedDiscoveries (`namespace0`, `configmap0`) and post-processing CEL expressions

---

### Test Steps

#### Step 1: Create a cluster via HyperFleet API
**Action:**
```bash
CLUSTER_ID=$(curl -s -X POST ${API_URL}/api/hyperfleet/v1/clusters \
  -H "Content-Type: application/json" \
  -d '{
    "kind": "Cluster",
    "name": "maestro-happy-path-'$(date +%Y%m%d-%H%M%S)'",
    "spec": {
      "platform": {
        "type": "gcp",
        "gcp": {"projectID": "test-project", "region": "us-central1"}
      },
      "release": {"version": "4.14.0"}
    }
  }' | jq -r '.id')
echo "CLUSTER_ID=${CLUSTER_ID}"
```

**Expected Result:**
- API returns HTTP 201 with a valid cluster ID and `generation: 1`

#### Step 2: Verify ManifestWork was created on Maestro
**Action:**
- Query the Maestro resource-bundles API from inside the maestro pod:
```bash
# Capture resource bundle ID for subsequent steps
RESOURCE_BUNDLE_ID=$(kubectl exec -n maestro deployment/maestro -- \
  curl -s http://localhost:8000/api/maestro/v1/resource-bundles \
  | jq -r --arg cid "${CLUSTER_ID}" \
    '.items[] | select(.metadata.labels["hyperfleet.io/cluster-id"] == $cid) | .id')
echo "RESOURCE_BUNDLE_ID=${RESOURCE_BUNDLE_ID}"

# Display resource bundle details
kubectl exec -n maestro deployment/maestro -- \
  curl -s http://localhost:8000/api/maestro/v1/resource-bundles/${RESOURCE_BUNDLE_ID} \
  | jq '{id: .id, consumer_name: .consumer_name, version: .version,
       manifest_names: [.manifests[].metadata.name]}'
```

**Expected Result:**
- A resource bundle (ManifestWork) is created on Maestro targeting `${MAESTRO_CONSUMER}`
- The resource bundle contains all expected inline manifests as resources
- `manifest_names` follows the naming pattern `${CLUSTER_ID}-${ADAPTER_NAME}-<resource_type>`:
  - `${CLUSTER_ID}-${ADAPTER_NAME}-namespace` (Namespace)
  - `${CLUSTER_ID}-${ADAPTER_NAME}-configmap` (ConfigMap)

Example output:
```json
{
  "id": "auto-generated unique ID by Maestro",
  "consumer_name": "${MAESTRO_CONSUMER}, the target consumer this ManifestWork is routed to",
  "version": 1,
  "manifest_names": [
    "${CLUSTER_ID}-${ADAPTER_NAME}-namespace",
    "${CLUSTER_ID}-${ADAPTER_NAME}-configmap"
  ]
}
```

#### Step 3: Verify ManifestWork metadata (labels and annotations)
**Action:**
```bash
kubectl exec -n maestro deployment/maestro -- \
  curl -s http://localhost:8000/api/maestro/v1/resource-bundles/${RESOURCE_BUNDLE_ID} \
  | jq '.metadata | {labels, annotations}'
```

**Expected Result:**

1. **Code logic additions** (dynamically set by adapter code):
   - `consumer_name`: set to the resolved `targetCluster` value (e.g., `${MAESTRO_CONSUMER}`)
   - `hyperfleet.io/generation` (label + annotation): set from the cluster's current generation value

2. **Manifest template configuration** (from adapter task config template):
   - Labels: `hyperfleet.io/cluster-id`, `hyperfleet.io/adapter`
   - Annotations: `hyperfleet.io/managed-by`

Example output:
```json
{
  "labels": {
    "hyperfleet.io/cluster-id": "${CLUSTER_ID}",
    "hyperfleet.io/generation": "1, code logic: set from cluster generation",
    "hyperfleet.io/adapter": "${ADAPTER_NAME}, template config: identifies the adapter"
  },
  "annotations": {
    "hyperfleet.io/generation": "1, code logic: used for idempotency check",
    "hyperfleet.io/managed-by": "${ADAPTER_NAME}, template config: indicates managing adapter"
  }
}
```

#### Step 4: Verify feedbackRules configuration in Maestro resource bundle
**Action:**
```bash
# Query feedbackRules
kubectl exec -n maestro deployment/maestro -- \
  curl -s http://localhost:8000/api/maestro/v1/resource-bundles/${RESOURCE_BUNDLE_ID} \
  | jq '.manifest_configs'
```

**Expected Result:**
- `manifestConfigs` contains feedbackRules with JSONPaths for status collection:
  - Namespace: `.status.phase`
  - ConfigMap: `.data`, `.metadata.resourceVersion`

Example output:
```json
[
  {
    "resourceIdentifier": {
      "name": "${CLUSTER_ID}-${ADAPTER_NAME}-namespace",
      "group": "",
      "resource": "namespaces"
    },
    "feedbackRules": [
      {"type": "JSONPaths", "jsonPaths": [{"name": "phase", "path": ".status.phase"}]}
    ]
  },
  {
    "resourceIdentifier": {
      "name": "${CLUSTER_ID}-${ADAPTER_NAME}-configmap",
      "group": "",
      "resource": "configmaps",
      "namespace": "${CLUSTER_ID}-${ADAPTER_NAME}-namespace"
    },
    "feedbackRules": [
      {"type": "JSONPaths", "jsonPaths": [
        {"name": "data", "path": ".data"},
        {"name": "resourceVersion", "path": ".metadata.resourceVersion"}
      ]}
    ]
  }
]
```

#### Step 5: Verify K8s resources created by Maestro agent on target cluster

Wait ~15 seconds for the Maestro agent to apply the ManifestWork content to the target cluster.

**Action:**
```bash
# Verify Namespace
kubectl get ns ${CLUSTER_ID}-${ADAPTER_NAME}-namespace

# Verify ConfigMap
kubectl get configmap ${CLUSTER_ID}-${ADAPTER_NAME}-configmap \
  -n ${CLUSTER_ID}-${ADAPTER_NAME}-namespace
```

**Expected Result:**
- Namespace `${CLUSTER_ID}-${ADAPTER_NAME}-namespace` exists and is `Active`
- ConfigMap `${CLUSTER_ID}-${ADAPTER_NAME}-configmap` exists in the namespace

#### Step 6: Verify adapter status report to HyperFleet API
**Action:**
```bash
curl -s ${API_URL}/api/hyperfleet/v1/clusters/${CLUSTER_ID}/statuses \
  | jq '.items[] | select(.adapter == "'"${ADAPTER_NAME}"'")'
```

**Expected Result:**
- Status entry with `adapter: "${ADAPTER_NAME}"`
- `observed_generation: 1`
- `observed_time` is present and is a valid timestamp
- Three conditions with expected values:
  - Applied = True, reason = `AppliedManifestWorkComplete`
  - Available = True, reason = `AllResourcesAvailable`
  - Health = True, reason = `Healthy`
- `data.manifestwork.name` = `"${CLUSTER_ID}-${ADAPTER_NAME}"`
- `data.namespace.phase` = `"Active"`
- `data.namespace.name` = `"${CLUSTER_ID}-${ADAPTER_NAME}-namespace"`
- `data.configmap.clusterId` = `"${CLUSTER_ID}"`
- `data.configmap.name` = `"${CLUSTER_ID}-${ADAPTER_NAME}-configmap"`

Example output:
```json
{
  "adapter": "${ADAPTER_NAME}",
  "observed_generation": 1,
  "observed_time": "2026-01-01T00:00:00Z",
  "conditions": [
    {
      "type": "Applied",
      "status": "True",
      "reason": "AppliedManifestWorkComplete"
    },
    {
      "type": "Available",
      "status": "True",
      "reason": "AllResourcesAvailable"
    },
    {
      "type": "Health",
      "status": "True",
      "reason": "Healthy"
    }
  ],
  "data": {
    "manifestwork": {
      "name": "${CLUSTER_ID}-${ADAPTER_NAME}"
    },
    "namespace": {
      "phase": "Active",
      "name": "${CLUSTER_ID}-${ADAPTER_NAME}-namespace"
    },
    "configmap": {
      "clusterId": "${CLUSTER_ID}",
      "name": "${CLUSTER_ID}-${ADAPTER_NAME}-configmap"
    }
  }
}
```

#### Step 7: Cleanup
**Action:**
```bash
# Delete the namespace created by Maestro agent
kubectl delete ns ${CLUSTER_ID}-${ADAPTER_NAME}-namespace --ignore-not-found

# Delete the resource bundle on Maestro (via Maestro API)
kubectl exec -n maestro deployment/maestro -- \
  curl -s -X DELETE http://localhost:8000/api/maestro/v1/resource-bundles/${RESOURCE_BUNDLE_ID}
```

> **Note:** This is a workaround cleanup method. Once the HyperFleet API supports DELETE operations for clusters, this step should be replaced with:
> ```bash
> curl -X DELETE ${API_URL}/api/hyperfleet/v1/clusters/${CLUSTER_ID}
> ```

---

## Test Title: Adapter can skip ManifestWork operation when generation is unchanged

### Description

This test validates the generation-based idempotency mechanism for ManifestWork operations via Maestro transport. When a ManifestWork does not exist, it should be created. When the same event is reprocessed with the same generation, the operation should be skipped.

---

| **Field** | **Value** |
|-----------|-----------|
| **Pos/Neg** | Positive |
| **Priority** | Tier0 |
| **Status** | Draft |
| **Automation** | Not Automated |
| **Version** | MVP |
| **Created** | 2026-02-12 |
| **Updated** | 2026-02-26 |

---

### Preconditions

1. HyperFleet API, Sentinel, and Adapter (Maestro mode) are deployed and running
2. Maestro server is accessible with at least one registered consumer

---

### Test Steps

#### Step 1: Create a cluster (triggers initial ManifestWork creation)
**Action:**
```bash
CLUSTER_ID=$(curl -s -X POST ${API_URL}/api/hyperfleet/v1/clusters \
  -H "Content-Type: application/json" \
  -d '{
    "kind": "Cluster",
    "name": "gen-test-'$(date +%Y%m%d-%H%M%S)'",
    "spec": {
      "platform": {"type": "gcp", "gcp": {"projectID": "test", "region": "us-central1"}},
      "release": {"version": "4.14.0"}
    }
  }' | jq -r '.id')
echo "CLUSTER_ID=${CLUSTER_ID}"
```

**Expected Result:**
- Cluster created with `generation: 1`

#### Step 2: Verify "Skip" operation on subsequent processing (same generation)
**Action:**
- The Sentinel continuously polls and re-publishes events every ~5 seconds. Wait for the next event processing cycle and check logs:
```bash
# Wait for a few more cycles
sleep 15
kubectl logs -n hyperfleet -l app.kubernetes.io/instance=hyperfleet-${ADAPTER_NAME} --tail=20 \
  | grep "Resource\[resource0\]"
```

**Expected Result:**
- Subsequent processing shows: `Resource[resource0] processed: operation=skip reason=generation 1 unchanged`

#### Step 3: Verify Maestro resource version does not change on Skip
**Action:**
```bash
# Capture resource bundle ID
RESOURCE_BUNDLE_ID=$(kubectl exec -n maestro deployment/maestro -- \
  curl -s http://localhost:8000/api/maestro/v1/resource-bundles \
  | jq -r --arg cid "${CLUSTER_ID}" \
    '.items[] | select(.metadata.labels["hyperfleet.io/cluster-id"] == $cid) | .id')
echo "RESOURCE_BUNDLE_ID=${RESOURCE_BUNDLE_ID}"

# Query the resource bundle version from Maestro - should stay at version 1
kubectl exec -n maestro deployment/maestro -- \
  curl -s http://localhost:8000/api/maestro/v1/resource-bundles/${RESOURCE_BUNDLE_ID} \
  | jq '{id: .id, version: .version}'
```

**Expected Result:**
- `version: 1` remains unchanged across multiple Skip operations

#### Step 4: Cleanup
**Action:**
```bash
kubectl delete ns ${CLUSTER_ID}-${ADAPTER_NAME}-namespace --ignore-not-found
kubectl exec -n maestro deployment/maestro -- \
  curl -s -X DELETE http://localhost:8000/api/maestro/v1/resource-bundles/${RESOURCE_BUNDLE_ID}
```

> **Note:** This is a workaround cleanup method. Once the HyperFleet API supports DELETE operations for clusters, this step should be replaced with:
> ```bash
> curl -X DELETE ${API_URL}/api/hyperfleet/v1/clusters/${CLUSTER_ID}
> ```

---

## Test Title: Adapter can route ManifestWork to correct consumer based on targetCluster

### Description

This test validates that the adapter can route ManifestWorks to different Maestro consumers based on the `targetCluster` template value. The adapter task config uses `targetCluster: "{{ .placementClusterName }}"` where `placementClusterName` is captured from a precondition expression. By changing this expression to point to a different consumer, ManifestWorks are routed to the new consumer.

---

| **Field** | **Value** |
|-----------|-----------|
| **Pos/Neg** | Positive |
| **Priority** | Tier1 |
| **Status** | Draft |
| **Automation** | Automated |
| **Version** | MVP |
| **Created** | 2026-02-12 |
| **Updated** | 2026-02-26 |

---

### Preconditions

1. HyperFleet environment deployed with Maestro transport adapter
2. Initial consumer `${MAESTRO_CONSUMER}` already registered
3. Adapter task config uses `targetCluster: "{{ .placementClusterName }}"` where `placementClusterName` is set via precondition capture expression

---

### Test Steps

#### Step 1: Register a second Maestro consumer
**Action:**
```bash
make create-maestro-consumer MAESTRO_CONSUMER=cluster2
```

**Expected Result:**
- Consumer `cluster2` created successfully

#### Step 2: Update adapter task config to use the new consumer
**Action:**
- Extract, modify, and re-apply the adapter task config:
```bash
# Extract current task config
kubectl get configmap hyperfleet-${ADAPTER_NAME}-task -n hyperfleet \
  -o jsonpath='{.data.task-config\.yaml}' > /tmp/adapter2-task-original.yaml

# Modify placementClusterName from "${MAESTRO_CONSUMER}" to "cluster2"
# In the task config, change:
#   expression: "\"${MAESTRO_CONSUMER}\""
# To:
#   expression: "\"cluster2\""

# Apply the modified config
kubectl create configmap hyperfleet-${ADAPTER_NAME}-task -n hyperfleet \
  --from-file=task-config.yaml=/tmp/adapter2-task-cluster2.yaml \
  --dry-run=client -o yaml | kubectl apply -f -

# Restart adapter to pick up new config
kubectl rollout restart deployment/hyperfleet-${ADAPTER_NAME} -n hyperfleet
kubectl rollout status deployment/hyperfleet-${ADAPTER_NAME} -n hyperfleet --timeout=60s
```

**Expected Result:**
- Adapter restarts with `placementClusterName` = `"cluster2"`

#### Step 3: Create a cluster and verify routing to cluster2
**Action:**
```bash
CLUSTER_ID=$(curl -s -X POST ${API_URL}/api/hyperfleet/v1/clusters \
  -H "Content-Type: application/json" \
  -d '{
    "kind": "Cluster",
    "name": "multi-consumer-test-'$(date +%Y%m%d-%H%M%S)'",
    "spec": {
      "platform": {"type": "gcp", "gcp": {"projectID": "test", "region": "us-central1"}},
      "release": {"version": "4.14.0"}
    }
  }' | jq -r '.id')
echo "CLUSTER_ID=${CLUSTER_ID}"
```

Wait ~15 seconds for the adapter to process.

**Expected Result:**
- API returns HTTP 201 with a valid cluster ID

#### Step 4: Verify ManifestWork is on the correct consumer via Maestro API
**Action:**
```bash
kubectl exec -n maestro deployment/maestro -- \
  curl -s http://localhost:8000/api/maestro/v1/resource-bundles \
  | jq '.items[] | {consumer_name: .consumer_name,
       cluster_id: .metadata.labels["hyperfleet.io/cluster-id"]}'
```

**Expected Result:**
- New cluster's resource bundle has `consumer_name: "cluster2"`
- Previously created clusters (before config change) remain on `consumer_name: "${MAESTRO_CONSUMER}"`

#### Step 5: Restore adapter config and cleanup
**Action:**
```bash
# Restore original config
kubectl create configmap hyperfleet-${ADAPTER_NAME}-task -n hyperfleet \
  --from-file=task-config.yaml=/tmp/adapter2-task-original.yaml \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl rollout restart deployment/hyperfleet-${ADAPTER_NAME} -n hyperfleet
kubectl rollout status deployment/hyperfleet-${ADAPTER_NAME} -n hyperfleet --timeout=60s
```

---

## Test Title: Adapter can handle Maestro server unavailability gracefully

### Description

This test validates the adapter's behavior when the Maestro server is unreachable. The adapter should handle connection failures gracefully, report appropriate error status back to the HyperFleet API, and not crash. When Maestro recovers, the adapter should automatically retry and succeed on subsequent events.

---

| **Field** | **Value** |
|-----------|-----------|
| **Pos/Neg** | Negative |
| **Priority** | Tier2 |
| **Status** | Draft |
| **Automation** | Not Automated |
| **Version** | MVP |
| **Created** | 2026-02-12 |
| **Updated** | 2026-03-27 |

---

### Preconditions

1. Environment is prepared using [hyperfleet-infra](https://github.com/openshift-hyperfleet/hyperfleet-infra) with all required platform resources
2. HyperFleet API and HyperFleet Sentinel services are deployed and running successfully
3. Adapter is deployed in Maestro transport mode and initially connected to Maestro
4. Ability to scale down the Maestro deployment

---

### Test Steps

#### Step 1: Scale down Maestro to simulate unavailability
**Action:**
- Scale down the Maestro deployment to make the Maestro server unreachable:
```bash
kubectl scale deployment maestro -n maestro --replicas=0
```

**Expected Result:**
- Maestro gRPC and HTTP endpoints become unreachable

#### Step 2: Submit an API request to create a Cluster resource
**Action:**
- Submit a POST request to create a Cluster resource:
```bash
curl -X POST ${API_URL}/api/hyperfleet/v1/clusters \
  -H "Content-Type: application/json" \
  -d @testdata/payloads/clusters/cluster-request.json
```

**Expected Result:**
- API returns successful response with cluster ID (API is independent of Maestro)

#### Step 3: Verify adapter failure is reported via status API

**Action:**
- Poll adapter statuses until the Maestro adapter reports its status:
```bash
curl -X GET ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}/statuses
```

**Expected Result:**
- The Maestro adapter is present in the statuses response (adapter handled the error gracefully without crashing)
- The adapter reports `Applied` condition with `status: "False"`
- The adapter reports `Available` condition with `status: "False"`
- The adapter reports `Health` condition with `status: "False"`, with reason and message indicating Maestro connection failure

#### Step 4: Verify cluster top-level status reflects adapter failure

**Action:**
- Retrieve cluster status:
```bash
curl -X GET ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}
```

**Expected Result:**
- Cluster `Ready` condition remains `status: "False"`
- Cluster `Available` condition remains `status: "False"`
- Cluster does not transition to Ready state while the Maestro adapter reports failure

#### Step 5: Restore Maestro and verify recovery
**Action:**
- Scale up the Maestro deployment:
```bash
kubectl scale deployment maestro -n maestro --replicas=1
```
- Poll adapter statuses until the Maestro adapter recovers:
```bash
curl -X GET ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}/statuses
```
- Retrieve cluster status:
```bash
curl -X GET ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}
```

**Expected Result:**
- Maestro adapter conditions transition to: `Applied=True`, `Available=True`, `Health=True`
- Cluster `Ready` condition transitions to `status: "True"`
- Cluster `Available` condition transitions to `status: "True"`

> **Note:** After Maestro restores, the adapter's CloudEvents client (MQTT-based) takes 60-90 seconds to re-establish the connection. During this window, events may fail with "the cloudevents client is not ready". The adapter automatically recovers once the connection is restored.

#### Step 6: Cleanup Resources (AfterEach)

**Action:**
- Delete the namespace created for this cluster:
```bash
kubectl delete namespace {cluster_id}
```
- Restore Maestro to normal state (if not already restored)

**Expected Result:**
- Namespace and all associated resources are deleted successfully
- Maestro is running normally

**Note:** This is a workaround cleanup method. Once the HyperFleet API supports DELETE operations for clusters, this step should be replaced with:
```bash
curl -X DELETE ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}
```

---

## Test Title: ManifestWork apply fails when targeting unregistered consumer

### Description

This test validates the adapter's behavior when a cluster event targets a Maestro consumer that is not registered. The ManifestWork apply operation should fail with "not registered in Maestro" error, and the adapter should report appropriate failure status via the Health condition without crashing.

---

| **Field** | **Value** |
|-----------|-----------|
| **Pos/Neg** | Negative |
| **Priority** | Tier1 |
| **Status** | Draft |
| **Automation** | Automated |
| **Version** | MVP |
| **Created** | 2026-02-12 |
| **Updated** | 2026-02-26 |

---

### Preconditions

1. HyperFleet API, Sentinel, and Maestro are deployed and running successfully
2. Adapter is deployed in Maestro transport mode (`transport.client: "maestro"`)
3. Adapter task config is configured to target a consumer named "unregistered-consumer" which does NOT exist in Maestro
4. At least one valid Maestro consumer exists for comparison (e.g., `cluster1`)
5. **Option 1**: Use the pre-configured adapter config: `testdata/adapter-configs/cl-m-unreg-consumer/`
6. **Option 2**: Temporarily modify an existing adapter's task config to point to "unregistered-consumer"

---

### Test Steps

#### Step 1: Verify Maestro is healthy and "unregistered-consumer" does not exist
**Action:**
```bash
# Verify Maestro is running
kubectl get pods -n maestro -l app=maestro --no-headers

# List all registered consumers to confirm "unregistered-consumer" is NOT present
kubectl exec -n maestro deployment/maestro -- \
  curl -s http://localhost:8000/api/maestro/v1/consumers \
  | jq '.items[].name'
```

**Expected Result:**
- Maestro pod is `Running`
- "unregistered-consumer" does NOT appear in the consumer list
- Other consumers (e.g., "cluster1") exist for comparison

#### Step 2: Deploy or verify adapter with unregistered consumer configuration
**Action:**

**Option A: Using pre-configured adapter (recommended)**
```bash
export ADAPTER_NAME='cl-m-unreg-consumer'

# Deploy the adapter using the pre-configured adapter config
      - name: "placementClusterName"
        expression: "\"unregistered-consumer\""  # Points to non-existent consumer to test apply failure
# Use helm install cmd to deploy
 helm install {release_name} {adapter_charts_folder} --namespace {namespace_name} --create-namespace  -f testdata/adapter-configs/cl-m-unreg-consumer/values.yaml
```

**Option B: Modify existing adapter config**
```bash
export ADAPTER_NAME='test-adapter'  # or your existing adapter name

# Backup original config
kubectl get configmap hyperfleet-${ADAPTER_NAME}-task -n hyperfleet \
  -o jsonpath='{.data.task-config\.yaml}' > /tmp/adapter-task-backup.yaml

# Modify task config: change placementClusterName to "unregistered-consumer"
# Edit the file to change:
#   expression: "\"cluster1\""
# To:
#   expression: "\"unregistered-consumer\""

# Apply modified config
kubectl create configmap hyperfleet-${ADAPTER_NAME}-task -n hyperfleet \
  --from-file=task-config.yaml=/tmp/adapter-task-modified.yaml \
  --dry-run=client -o yaml | kubectl apply -f -

# Restart adapter to pick up new config
kubectl rollout restart deployment/hyperfleet-${ADAPTER_NAME} -n hyperfleet
kubectl rollout status deployment/hyperfleet-${ADAPTER_NAME} -n hyperfleet --timeout=60s
```

**Expected Result:**
- Adapter pod restarts successfully
- Adapter task config now targets "unregistered-consumer"

#### Step 3: Verify adapter is running and ready
**Action:**
```bash
kubectl get pods -n hyperfleet -l app.kubernetes.io/instance=hyperfleet-${ADAPTER_NAME} --no-headers
```

**Expected Result:**
- Adapter pod is `Running` with `1/1 Ready`

#### Step 4: Create a cluster to trigger adapter processing
**Action:**
```bash
CLUSTER_ID=$(curl -s -X POST ${API_URL}/api/hyperfleet/v1/clusters \
  -H "Content-Type: application/json" \
  -d '{
    "kind": "Cluster",
    "name": "invalid-consumer-test-'$(date +%Y%m%d-%H%M%S)'",
    "spec": {
      "platform": {"type": "gcp", "gcp": {"projectID": "test", "region": "us-central1"}},
      "release": {"version": "4.14.0"}
    }
  }' | jq -r '.id')
echo "CLUSTER_ID=${CLUSTER_ID}"
```

**Expected Result:**
- API returns HTTP 201 with a valid cluster ID
- Cluster has `generation: 1`

#### Step 5: Verify error status reported to HyperFleet API
**Action:**
```bash
curl -s ${API_URL}/api/hyperfleet/v1/clusters/${CLUSTER_ID}/statuses \
  | jq '.items[] | select(.adapter == "'"${ADAPTER_NAME}"'")'
```

**Expected Result:**
- Adapter status entry exists with `adapter: "${ADAPTER_NAME}"`
- `observed_generation: 1` (adapter processed the event)
- `last_report_time` is present and recent
- **Condition validation**:
  - `Applied: False` - ManifestWork was not created (consumer not registered)
  - `Available: False` - Resources not available (ManifestWork not applied)
  - `Health: False` - Adapter execution failed at ResourceFailed phase
    - Health reason: `ExecutionFailed:ResourceFailed`
    - Health message contains: "consumer \"xxxxxx\" is not registered in Maestro"


#### Step 6: Verify no ManifestWork was created on Maestro
**Action:**
```bash
kubectl exec -n maestro deployment/maestro -- \
  curl -s http://localhost:8000/api/maestro/v1/resource-bundles \
  | jq '.items[] | select(.metadata.labels["hyperfleet.io/cluster-id"] == "'"${CLUSTER_ID}"'")'
```

**Expected Result:**
- No resource bundle (ManifestWork) exists for the cluster ID
- Query returns empty result or null
- This confirms the apply operation failed before creating the ManifestWork

#### Step 7: Verify no Kubernetes resources were created
**Action:**
```bash
# Attempt to find namespace that would have been created
kubectl get ns | grep ${CLUSTER_ID}
```

**Expected Result:**
- No namespace exists with the cluster ID
- This confirms that Maestro agent did not apply any resources (because ManifestWork was never created)

#### Step 8: Cleanup
**Action:**

**If using Option A (pre-configured adapter):**
```bash
# Delete the test adapter deployment
helm uninstall {release_name} -n {namespace}

# Note: Cluster will remain in API until DELETE endpoint is available
```

**If using Option B (modified existing adapter):**
```bash
# Restore original adapter config
kubectl create configmap hyperfleet-${ADAPTER_NAME}-task -n hyperfleet \
  --from-file=task-config.yaml=/tmp/adapter-task-backup.yaml \
  --dry-run=client -o yaml | kubectl apply -f -

# Restart adapter with restored config
kubectl rollout restart deployment/hyperfleet-${ADAPTER_NAME} -n hyperfleet
kubectl rollout status deployment/hyperfleet-${ADAPTER_NAME} -n hyperfleet --timeout=60s

echo "Adapter config restored successfully"
```

> **Note:** Once the HyperFleet API supports DELETE operations for clusters, this step should be added with:
> ```bash
> curl -X DELETE ${API_URL}/api/hyperfleet/v1/clusters/${CLUSTER_ID}
> ```

---

## Test Title: Main discovery fails when ManifestWork name is wrong

### Description

This test validates the adapter's behavior when the main discovery configuration uses the wrong ManifestWork name. The adapter creates a ManifestWork on Maestro with the correct name, but then tries to discover it using a wrong name (with `-wrong` suffix). This simulates a misconfiguration where the discovery name doesn't match the created resource name. The adapter should fail at the discovery phase and report the error appropriately.

---

| **Field** | **Value** |
|-----------|-----------|
| **Pos/Neg** | Negative |
| **Priority** | Tier1 |
| **Status** | Draft |
| **Automation** | Automated |
| **Version** | MVP |
| **Created** | 2026-03-20 |
| **Updated** | 2026-03-20 |

---

### Preconditions

1. HyperFleet API, Sentinel, and Maestro are deployed and running successfully
2. At least one Maestro consumer is registered (e.g., `cluster1`)
3. Adapter is deployed in Maestro transport mode
4. Adapter task config has discovery names that DO NOT match the actual resource names created
5. **Option 1**: Use the pre-configured adapter config: `testdata/adapter-configs/cl-m-wrong-ds/`
6. **Option 2**: Temporarily modify an existing adapter's task config discovery names to be incorrect

---

### Test Steps

#### Step 1: Verify Maestro is healthy and consumer is registered
**Action:**
```bash
# Verify Maestro is running
kubectl get pods -n maestro -l app=maestro --no-headers

# Verify consumer exists
export MAESTRO_CONSUMER='cluster1'  # or your registered consumer
kubectl exec -n maestro deployment/maestro -- \
  curl -s http://localhost:8000/api/maestro/v1/consumers \
  | jq -r '.items[] | select(.name == "'"${MAESTRO_CONSUMER}"'") | .name'
```

**Expected Result:**
- Maestro pod is `Running`
- Consumer `${MAESTRO_CONSUMER}` exists

#### Step 2: Deploy or verify adapter with wrong discovery configuration
**Action:**

**Option A: Using pre-configured adapter (recommended)**
```bash
export ADAPTER_NAME='cl-m-wrong-ds'

# Deploy the test adapter deployment
 helm install {release_name} {adapter_charts_folder} --namespace {namespace_name} --create-namespace  -f testdata/adapter-configs/cl-m-wrong-ds/values.yaml

OR

# Deploy the adapter using the pre-configured adapter config supported in hyperfleet-infra
# The config has discovery names with "-wrong" suffix that don't match actual resources
make install-adapter-custom ADAPTER_CONFIG_PATH=testdata/adapter-configs/cl-m-wrong-ds
```

**Option B: Modify existing adapter config**
```bash
export ADAPTER_NAME='cl-maestro'  # or your existing adapter name

# Backup original config
kubectl get configmap hyperfleet-${ADAPTER_NAME}-task -n hyperfleet \
  -o jsonpath='{.data.task-config\.yaml}' > /tmp/adapter-task-backup.yaml

# Modify task config nested_discoveries section:
# Change:
#   by_name: "{{ .clusterId | lower }}-{{ .adapter.name }}-namespace"
# To:
#   by_name: "{{ .clusterId | lower }}-{{ .adapter.name }}-namespace-wrong"
# And:
#   by_name: "{{ .clusterId | lower }}-{{ .adapter.name }}-configmap"
# To:
#   by_name: "{{ .clusterId | lower }}-{{ .adapter.name }}-configmap-wrong"

# Apply modified config
kubectl create configmap hyperfleet-${ADAPTER_NAME}-task -n hyperfleet \
  --from-file=task-config.yaml=/tmp/adapter-task-modified.yaml \
  --dry-run=client -o yaml | kubectl apply -f -

# Restart adapter to pick up new config
kubectl rollout restart deployment/hyperfleet-${ADAPTER_NAME} -n hyperfleet
kubectl rollout status deployment/hyperfleet-${ADAPTER_NAME} -n hyperfleet --timeout=60s
```

**Expected Result:**
- Adapter pod restarts successfully
- Adapter task config now has wrong discovery names (with "-wrong" suffix)

#### Step 3: Verify adapter is running
**Action:**
```bash
kubectl get pods -n hyperfleet -l app.kubernetes.io/instance=hyperfleet-${ADAPTER_NAME} --no-headers
```

**Expected Result:**
- Adapter pod is `Running` with `1/1 Ready`

#### Step 4: Create a cluster to trigger adapter processing
**Action:**
```bash
export API_URL='http://localhost:8000'  # Adjust if different

CLUSTER_ID=$(curl -s -X POST ${API_URL}/api/hyperfleet/v1/clusters \
  -H "Content-Type: application/json" \
  -d '{
    "kind": "Cluster",
    "name": "maestro-discovery-fail-'$(date +%Y%m%d-%H%M%S)'",
    "spec": {
      "platform": {
        "type": "gcp",
        "gcp": {"projectID": "test-project", "region": "us-central1"}
      },
      "release": {"version": "4.14.0"}
    }
  }' | jq -r '.id')
echo "CLUSTER_ID=${CLUSTER_ID}"
```

**Expected Result:**
- API returns HTTP 201 with a valid cluster ID
- Cluster has `generation: 1`

#### Step 5: Verify ManifestWork was created successfully on Maestro
**Action:**
```bash
# Wait for ManifestWork creation
sleep 10

# Capture resource bundle ID
RESOURCE_BUNDLE_ID=$(kubectl exec -n maestro deployment/maestro -- \
  curl -s http://localhost:8000/api/maestro/v1/resource-bundles \
  | jq -r --arg cid "${CLUSTER_ID}" \
    '.items[] | select(.metadata.labels["hyperfleet.io/cluster-id"] == $cid) | .id')
echo "RESOURCE_BUNDLE_ID=${RESOURCE_BUNDLE_ID}"

# Display resource bundle details
kubectl exec -n maestro deployment/maestro -- \
  curl -s http://localhost:8000/api/maestro/v1/resource-bundles/${RESOURCE_BUNDLE_ID} \
  | jq '{id: .id, consumer_name: .consumer_name, version: .version,
       manifest_names: [.manifests[].metadata.name]}'
```

**Expected Result:**
- ManifestWork (resource bundle) was created successfully
- Resource bundle has correct consumer name (e.g., `cluster1`)
- Manifests include namespace and configmap with correct actual names:
  - `${CLUSTER_ID}-${ADAPTER_NAME}-namespace`
  - `${CLUSTER_ID}-${ADAPTER_NAME}-configmap`

#### Step 6: Verify Kubernetes resources were created by Maestro agent
**Action:**
```bash
# Wait for Maestro agent to apply resources
sleep 15

# Verify namespace exists
kubectl get ns | grep ${CLUSTER_ID}-${ADAPTER_NAME}

# Verify configmap exists
kubectl get configmap -n ${CLUSTER_ID}-${ADAPTER_NAME}-namespace | grep ${CLUSTER_ID}-${ADAPTER_NAME}
```

**Expected Result:**
- Namespace `${CLUSTER_ID}-${ADAPTER_NAME}-namespace` exists and is `Active`
- ConfigMap `${CLUSTER_ID}-${ADAPTER_NAME}-configmap` exists in the namespace
- Resources were successfully applied by Maestro agent

#### Step 7: Verify error status reported to HyperFleet API
**Action:**
```bash
curl -s ${API_URL}/api/hyperfleet/v1/clusters/${CLUSTER_ID}/statuses \
  | jq '.items[] | select(.adapter == "'"${ADAPTER_NAME}"'")'
```

**Expected Result:**
- Adapter status entry exists with `adapter: "${ADAPTER_NAME}"`
- `observed_generation: 1` (adapter processed the event)
- `last_report_time` is present and recent
- **Condition validation**:
  - `Applied: False` - ManifestWork not discovered (main discovery failed)
    - Reason: `ManifestWorkNotDiscovered`
  - `Available: False` - Resources not available (ManifestWork not found)
    - Reason: `NamespaceNotDiscovered`
  - `Health: False` - Adapter execution failed
    - Reason: `ExecutionFailed:ResourceFailed`
    - Message contains: "failed to discover resource after apply: manifestworks...not found"
- **Data validation**:
  - `data.manifestwork.name` is empty (main discovery failed)
  - `data.namespace.name` is empty (cannot discover nested resources)
  - `data.configmap.name` is empty (cannot discover nested resources)

#### Step 8: Verify ManifestWork was created but cannot be discovered
**Action:**
```bash
# Search for ManifestWork with correct name (without -wrong suffix)
kubectl exec -n maestro deployment/maestro -- \
  curl -s http://localhost:8000/api/maestro/v1/resource-bundles \
  | jq '.items[] | select(.metadata.labels["hyperfleet.io/cluster-id"] == "'"${CLUSTER_ID}"'")'

# Try to find ManifestWork with wrong name (what adapter is looking for)
kubectl exec -n maestro deployment/maestro -- \
  curl -s "http://localhost:8000/api/maestro/v1/resource-bundles/${CLUSTER_ID}-${ADAPTER_NAME}-wrong"
```

**Expected Result:**
- ManifestWork with correct name `${CLUSTER_ID}-${ADAPTER_NAME}` exists on Maestro
- ManifestWork with wrong name `${CLUSTER_ID}-${ADAPTER_NAME}-wrong` does NOT exist (404)
- Adapter created the ManifestWork correctly but cannot discover it due to wrong discovery name
- K8s resources (namespace, configmap) were created by Maestro agent

#### Step 9: Cleanup
**Action:**

**Common cleanup steps:**
```bash
# Delete the resource bundle on Maestro (triggers agent to clean up K8s resources)
kubectl exec -n maestro deployment/maestro -- \
  curl -s -X DELETE http://localhost:8000/api/maestro/v1/resource-bundles/${RESOURCE_BUNDLE_ID}

# Delete namespace as safety cleanup
kubectl delete ns ${CLUSTER_ID}-${ADAPTER_NAME}-namespace --ignore-not-found

# Wait for cleanup
sleep 5
```
> **Note:** Once the HyperFleet API supports DELETE operations for clusters, it can be replaced via this cleanup step:
> ```bash
> curl -X DELETE ${API_URL}/api/hyperfleet/v1/clusters/${CLUSTER_ID}
> ```

**If using Option A (pre-configured adapter):**
```bash
# Delete the test adapter deployment
helm uninstall hyperfleet-${ADAPTER_NAME} -n hyperfleet

# Or using make target supported in hyperfleet-infra
make uninstall-adapter ADAPTER_NAME=cl-maestro-wrong-discovery 
```

**If using Option B (modified existing adapter):**
```bash
# Restore original adapter config
kubectl create configmap hyperfleet-${ADAPTER_NAME}-task -n hyperfleet \
  --from-file=task-config.yaml=/tmp/adapter-task-backup.yaml \
  --dry-run=client -o yaml | kubectl apply -f -

# Restart adapter with restored config
kubectl rollout restart deployment/hyperfleet-${ADAPTER_NAME} -n hyperfleet
kubectl rollout status deployment/hyperfleet-${ADAPTER_NAME} -n hyperfleet --timeout=60s

echo "Adapter config restored successfully"
```
---

## Test Title: Nested discovery returns empty when criteria match nothing in manifests

### Description

This test validates the adapter's behavior when a ManifestWork is successfully created and discovered, but the nested discovery criteria match nothing in the `spec.workload.manifests` array. The ManifestWork apply and primary discovery succeed, but nested discovery returns empty results. This is not a hard failure - it's logged as debug information, and CEL expressions using `orValue("")` fallbacks handle the missing data gracefully. The adapter reports status with conditions showing pending/unknown state due to unavailable nested resource data.

---

| **Field** | **Value** |
|-----------|-----------|
| **Pos/Neg** | Negative |
| **Priority** | Tier1 |
| **Status** | Draft |
| **Automation** | Automated |
| **Version** | MVP |
| **Created** | 2026-03-20 |
| **Updated** | 2026-03-20 |

---

### Preconditions

1. HyperFleet API, Sentinel, and Maestro are deployed and running successfully
2. At least one Maestro consumer is registered (e.g., `cluster1`)
3. Adapter is deployed in Maestro transport mode
4. Adapter task config has nested discovery criteria that look for resources NOT present in the ManifestWork manifests
5. **Option 1**: Use the pre-configured adapter config: `testdata/adapter-configs/cl-m-wrong-nest/`
6. **Option 2**: Temporarily modify an existing adapter's task config to have mismatched nested discovery criteria

---

### Test Steps

#### Step 1: Verify Maestro is healthy and consumer is registered
**Action:**
```bash
# Verify Maestro is running
kubectl get pods -n maestro -l app=maestro --no-headers

# Verify consumer exists
export MAESTRO_CONSUMER='cluster1'  # or your registered consumer
kubectl exec -n maestro deployment/maestro -- \
  curl -s http://localhost:8000/api/maestro/v1/consumers \
  | jq -r '.items[] | select(.name == "'"${MAESTRO_CONSUMER}"'") | .name'
```

**Expected Result:**
- Maestro pod is `Running`
- Consumer `${MAESTRO_CONSUMER}` exists

#### Step 2: Deploy or verify adapter with empty nested discovery configuration
**Action:**

**Option A: Using pre-configured adapter (recommended)**
```bash
export ADAPTER_NAME='cl-m-wrong-nest'

# Deploy the test adapter deployment
helm install {release_name} {adapter_charts_folder} --namespace {namespace_name} --create-namespace -f testdata/adapter-configs/cl-m-wrong-nest/values.yaml

# OR using make target supported in hyperfleet-infra
make install-adapter-custom ADAPTER_CONFIG_PATH=testdata/adapter-configs/cl-m-wrong-nest
```

**Option B: Modify existing adapter config**
```bash
export ADAPTER_NAME='cl-maestro'  # or your existing adapter name

# Backup original config
kubectl get configmap hyperfleet-${ADAPTER_NAME}-task -n hyperfleet \
  -o jsonpath='{.data.task-config\.yaml}' > /tmp/adapter-task-backup.yaml

# Modify task config nested_discoveries section to look for non-existent resources:
# Change:
#   by_name: "{{ .clusterId | lower }}-{{ .adapter.name }}-namespace"
# To:
#   by_name: "{{ .clusterId | lower }}-{{ .adapter.name }}-deployment"
# And:
#   by_name: "{{ .clusterId | lower }}-{{ .adapter.name }}-configmap"
# To:
#   by_name: "{{ .clusterId | lower }}-{{ .adapter.name }}-service"

# Apply modified config
kubectl create configmap hyperfleet-${ADAPTER_NAME}-task -n hyperfleet \
  --from-file=task-config.yaml=/tmp/adapter-task-modified.yaml \
  --dry-run=client -o yaml | kubectl apply -f -

# Restart adapter to pick up new config
kubectl rollout restart deployment/hyperfleet-${ADAPTER_NAME} -n hyperfleet
kubectl rollout status deployment/hyperfleet-${ADAPTER_NAME} -n hyperfleet --timeout=60s
```

**Expected Result:**
- Adapter pod restarts successfully
- Adapter task config now has nested discovery criteria that won't match any manifests

#### Step 3: Verify adapter is running
**Action:**
```bash
kubectl get pods -n hyperfleet -l app.kubernetes.io/instance=hyperfleet-${ADAPTER_NAME} --no-headers
```

**Expected Result:**
- Adapter pod is `Running` with `1/1 Ready`

#### Step 4: Create a cluster to trigger adapter processing
**Action:**
```bash
export API_URL='http://localhost:8000'  # Adjust if different

CLUSTER_ID=$(curl -s -X POST ${API_URL}/api/hyperfleet/v1/clusters \
  -H "Content-Type: application/json" \
  -d '{
    "kind": "Cluster",
    "name": "maestro-empty-discovery-'$(date +%Y%m%d-%H%M%S)'",
    "spec": {
      "platform": {
        "type": "gcp",
        "gcp": {"projectID": "test-project", "region": "us-central1"}
      },
      "release": {"version": "4.14.0"}
    }
  }' | jq -r '.id')
echo "CLUSTER_ID=${CLUSTER_ID}"
```

**Expected Result:**
- API returns HTTP 201 with a valid cluster ID
- Cluster has `generation: 1`

#### Step 5: Verify ManifestWork was created successfully on Maestro
**Action:**
```bash
# Wait for ManifestWork creation
sleep 10

# Capture resource bundle ID
RESOURCE_BUNDLE_ID=$(kubectl exec -n maestro deployment/maestro -- \
  curl -s http://localhost:8000/api/maestro/v1/resource-bundles \
  | jq -r --arg cid "${CLUSTER_ID}" \
    '.items[] | select(.metadata.labels["hyperfleet.io/cluster-id"] == $cid) | .id')
echo "RESOURCE_BUNDLE_ID=${RESOURCE_BUNDLE_ID}"

# Display resource bundle details
kubectl exec -n maestro deployment/maestro -- \
  curl -s http://localhost:8000/api/maestro/v1/resource-bundles/${RESOURCE_BUNDLE_ID} \
  | jq '{id: .id, consumer_name: .consumer_name, version: .version,
       manifest_names: [.manifests[].metadata.name]}'
```

**Expected Result:**
- ManifestWork (resource bundle) was created successfully
- Resource bundle has correct consumer name (e.g., `cluster1`)
- Manifests include the actual resources (namespace and configmap):
  - `${CLUSTER_ID}-${ADAPTER_NAME}-namespace`
  - `${CLUSTER_ID}-${ADAPTER_NAME}-configmap`
- Note: Nested discovery is looking for deployment and service which don't exist

#### Step 6: Verify Kubernetes resources were created by Maestro agent
**Action:**
```bash
# Wait for Maestro agent to apply resources
sleep 15

# Verify namespace exists
kubectl get ns | grep ${CLUSTER_ID}-${ADAPTER_NAME}

# Verify configmap exists
kubectl get configmap -n ${CLUSTER_ID}-${ADAPTER_NAME}-namespace | grep ${CLUSTER_ID}-${ADAPTER_NAME}
```

**Expected Result:**
- Namespace `${CLUSTER_ID}-${ADAPTER_NAME}-namespace` exists and is `Active`
- ConfigMap `${CLUSTER_ID}-${ADAPTER_NAME}-configmap` exists in the namespace
- Resources were successfully applied by Maestro agent

#### Step 7: Verify status reported with pending/unknown conditions
**Action:**
```bash
curl -s ${API_URL}/api/hyperfleet/v1/clusters/${CLUSTER_ID}/statuses \
  | jq '.items[] | select(.adapter == "'"${ADAPTER_NAME}"'")'
```

**Expected Result:**
- Adapter status entry exists with `adapter: "${ADAPTER_NAME}"`
- `observed_generation: 1` (adapter processed the event)
- `last_report_time` is present and recent
- **Condition validation**:
  - `Applied: True` with `reason: "AppliedManifestWorkComplete"` - ManifestWork was created successfully
  - `Available: False` with `reason: "NamespaceNotDiscovered"` - Nested resources not discovered
  - `Health: True` with `reason: "Healthy"` - Adapter executed successfully (nested discovery failure doesn't affect health)
- **Data field validation**:
  - `manifestwork.name`: `"${CLUSTER_ID}-${ADAPTER_NAME}"` (main discovery succeeded)
  - `namespace.name`: `""` (empty - nested discovery failed)
  - `namespace.phase`: `"Unknown"` (nested discovery failed)
  - `configmap.name`: `""` (empty - nested discovery failed)


#### Step 8: Verify ManifestWork and actual resources exist
**Action:**
```bash
# Verify ManifestWork exists on Maestro
kubectl exec -n maestro deployment/maestro -- \
  curl -s http://localhost:8000/api/maestro/v1/resource-bundles/${RESOURCE_BUNDLE_ID} \
  | jq '{id: .id, version: .version, manifests: [.manifests[].metadata.name]}'

# Verify actual K8s resources exist (namespace and configmap)
kubectl get ns ${CLUSTER_ID}-${ADAPTER_NAME}-namespace
kubectl get configmap ${CLUSTER_ID}-${ADAPTER_NAME}-configmap \
  -n ${CLUSTER_ID}-${ADAPTER_NAME}-namespace
```

**Expected Result:**
- ManifestWork exists with correct manifests (namespace and configmap)
- Kubernetes namespace and configmap exist and are functional
- Nested discovery failure doesn't affect the actual resources

#### Step 9: Cleanup
**Action:**

**Common cleanup steps:**
```bash
# Delete the resource bundle on Maestro (triggers agent to clean up K8s resources)
kubectl exec -n maestro deployment/maestro -- \
  curl -s -X DELETE http://localhost:8000/api/maestro/v1/resource-bundles/${RESOURCE_BUNDLE_ID}

# Delete namespace as safety cleanup
kubectl delete ns ${CLUSTER_ID}-${ADAPTER_NAME}-namespace --ignore-not-found

# Wait for cleanup
sleep 5
```

> **Note:** Once the HyperFleet API supports DELETE operations for clusters, it can be replaced via this cleanup step:
> ```bash
> curl -X DELETE ${API_URL}/api/hyperfleet/v1/clusters/${CLUSTER_ID}
> ```

**If using Option A (pre-configured adapter):**
```bash
# Delete the test adapter deployment
helm uninstall hyperfleet-${ADAPTER_NAME} -n hyperfleet

# Or using make target supported in hyperfleet-infra
make uninstall-adapter ADAPTER_NAME=cl-m-wrong-nest
```

**If using Option B (modified existing adapter):**
```bash
# Restore original adapter config
kubectl create configmap hyperfleet-${ADAPTER_NAME}-task -n hyperfleet \
  --from-file=task-config.yaml=/tmp/adapter-task-backup.yaml \
  --dry-run=client -o yaml | kubectl apply -f -

# Restart adapter with restored config
kubectl rollout restart deployment/hyperfleet-${ADAPTER_NAME} -n hyperfleet
kubectl rollout status deployment/hyperfleet-${ADAPTER_NAME} -n hyperfleet --timeout=60s

echo "Adapter config restored successfully"
```

---

## Test Title: Post-action fails when status API is unreachable or returns error

### Description

This test validates the adapter's behavior when ManifestWork creation and discovery succeed, but the POST to `/clusters/{clusterId}/statuses` endpoint fails (returns 500 error or is unreachable). The adapter should handle the API failure gracefully, record the post-action failure in execution metadata, log the error appropriately, and continue running without crashing.

---

| **Field** | **Value** |
|-----------|-----------|
| **Pos/Neg** | Negative |
| **Priority** | Tier1 |
| **Status** | Draft |
| **Automation** | Automated |
| **Version** | MVP |
| **Created** | 2026-03-20 |
| **Updated** | 2026-03-20 |

---

### Preconditions

1. HyperFleet API, Sentinel, and Maestro are deployed and running successfully
2. At least one Maestro consumer is registered (e.g., `cluster1`)
3. Pre-configured test adapter available: `testdata/adapter-configs/cl-m-bad-api/`
   - This adapter has an invalid API URL configured to simulate unreachable API
   - Clean approach that doesn't affect test environment or existing adapters

---

### Test Steps

#### Step 1: Deploy test adapter with invalid API URL
**Action:**
```bash
export ADAPTER_NAME='cl-m-bad-api'

# Deploy the test adapter with pre-configured invalid API URL
# This adapter will successfully connect to Maestro but fail when POSTing to status API
helm install hyperfleet-${ADAPTER_NAME} {adapter_charts_folder} \
  --namespace hyperfleet \
  --create-namespace \
  -f testdata/adapter-configs/cl-m-bad-api/values.yaml

# OR using make target supported in hyperfleet-infra
make install-adapter-custom ADAPTER_CONFIG_PATH=testdata/adapter-configs/cl-m-bad-api

# Wait for adapter to be ready
kubectl rollout status deployment/hyperfleet-${ADAPTER_NAME} -n hyperfleet --timeout=60s

# Verify adapter is running
kubectl get pods -n hyperfleet -l app.kubernetes.io/instance=hyperfleet-${ADAPTER_NAME} --no-headers
```

**Expected Result:**
- Test adapter pod is `Running` with `1/1 Ready`
- Adapter is configured with invalid API URL: `http://invalid-hyperfleet-api-endpoint.local:9999`

#### Step 2: Create a cluster to trigger adapter processing
**Action:**
```bash
export API_URL='http://localhost:8000'  # Adjust if different

CLUSTER_ID=$(curl -s -X POST ${API_URL}/api/hyperfleet/v1/clusters \
  -H "Content-Type: application/json" \
  -d '{
    "kind": "Cluster",
    "name": "maestro-api-fail-test-'$(date +%Y%m%d-%H%M%S)'",
    "spec": {
      "platform": {
        "type": "gcp",
        "gcp": {"projectID": "test-project", "region": "us-central1"}
      },
      "release": {"version": "4.14.0"}
    }
  }' | jq -r '.id')
echo "CLUSTER_ID=${CLUSTER_ID}"
```

**Expected Result:**
- API returns HTTP 201 with a valid cluster ID
- Cluster has `generation: 1`

#### Step 3: Verify ManifestWork was created successfully despite API failure
**Action:**
```bash
# Capture resource bundle ID
RESOURCE_BUNDLE_ID=$(kubectl exec -n maestro deployment/maestro -- \
  curl -s http://localhost:8000/api/maestro/v1/resource-bundles \
  | jq -r --arg cid "${CLUSTER_ID}" \
    '.items[] | select(.metadata.labels["hyperfleet.io/cluster-id"] == $cid) | .id')
echo "RESOURCE_BUNDLE_ID=${RESOURCE_BUNDLE_ID}"

# Display resource bundle details
kubectl exec -n maestro deployment/maestro -- \
  curl -s http://localhost:8000/api/maestro/v1/resource-bundles/${RESOURCE_BUNDLE_ID} \
  | jq '{id: .id, consumer_name: .consumer_name, version: .version,
       manifest_names: [.manifests[].metadata.name]}'
```

**Expected Result:**
- ManifestWork (resource bundle) was created successfully on Maestro

#### Step 4: Verify Kubernetes resources were created by Maestro agent
**Action:**
```bash
# Wait for Maestro agent to apply resources
sleep 15

# Verify namespace exists
kubectl get ns | grep ${CLUSTER_ID}-${ADAPTER_NAME}

# Verify configmap exists
kubectl get configmap -n ${CLUSTER_ID}-${ADAPTER_NAME}-namespace | grep ${CLUSTER_ID}-${ADAPTER_NAME}
```

**Expected Result:**
- Namespace `${CLUSTER_ID}-${ADAPTER_NAME}-namespace` exists and is `Active`
- ConfigMap `${CLUSTER_ID}-${ADAPTER_NAME}-configmap` exists
- Resources were successfully applied despite post-action failure

#### Step 5: Verify post-action failure via indirect evidence (beyond logs)
**Action:**
```bash
# Method: Check ManifestWork status in Maestro (should be healthy)
kubectl exec -n maestro deployment/maestro -- \
  curl -s http://localhost:8000/api/maestro/v1/resource-bundles/${RESOURCE_BUNDLE_ID} \
  | jq '{
      id: .id,
      status: .status,
      conditions: [.status.conditions[] | {type: .type, status: .status}]
    }'
```
**Expected Result:**
- **ManifestWork in Maestro:**
  - Status shows `Applied` and `Available` conditions are `True` (from Maestro agent's perspective)
  - This confirms apply and discovery phases succeeded

#### Step 6: Verify no status was reported to API (expected behavior)
**Action:**
```bash
# Since the adapter has invalid API URL, status should NOT be in the API
curl -s ${API_URL}/api/hyperfleet/v1/clusters/${CLUSTER_ID}/statuses \
  | jq '.items[] | select(.adapter == "'"${ADAPTER_NAME}"'")'
```

**Expected Result:**
- No status entry for this adapter (empty result)
- This confirms that POST to /statuses failed as expected

#### Step 9: Cleanup
**Action:**
```bash
# Delete the resource bundle on Maestro
kubectl exec -n maestro deployment/maestro -- \
  curl -s -X DELETE http://localhost:8000/api/maestro/v1/resource-bundles/${RESOURCE_BUNDLE_ID}

# Delete namespace
kubectl delete ns ${CLUSTER_ID}-${ADAPTER_NAME}-namespace --ignore-not-found

> **Note:** Once the HyperFleet API supports DELETE operations for clusters, add this cleanup step:
> ```bash
> curl -X DELETE ${API_URL}/api/hyperfleet/v1/clusters/${CLUSTER_ID}
> ```

# Delete the test adapter deployment
helm uninstall hyperfleet-${ADAPTER_NAME} -n hyperfleet

# OR using make target supported in hyperfleet-infra
make uninstall-adapter ADAPTER_NAME=cl-m-bad-api

# Verify adapter is deleted
kubectl get pods -n hyperfleet -l app.kubernetes.io/instance=hyperfleet-${ADAPTER_NAME} --no-headers
```

---

