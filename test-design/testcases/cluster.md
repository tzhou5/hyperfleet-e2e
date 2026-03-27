# Feature: Clusters Resource Type Lifecycle Management

## Table of Contents

1. [Clusters Resource Type - Workflow Validation](#test-title-clusters-resource-type---workflow-validation)
2. [Clusters Resource Type - K8s Resources Check Aligned with Preinstalled Clusters Related Adapters Specified](#test-title-clusters-resource-type---k8s-resources-check-aligned-with-preinstalled-clusters-related-adapters-specified)
3. [Clusters Resource Type - Adapter Dependency Relationships Workflow Validation for Preinstalled Clusters Related Dependent Adapters](#test-title-clusters-resource-type---adapter-dependency-relationships-workflow-validation-for-preinstalled-clusters-related-dependent-adapters)
4. [Cluster can reflect adapter failure in top-level status](#test-title-cluster-can-reflect-adapter-failure-in-top-level-status)
5. [Cluster can reach correct status after adapter crash and recovery](#test-title-cluster-can-reach-correct-status-after-adapter-crash-and-recovery)

---

## Test Title: Clusters Resource Type - Basic Workflow Validation

### Description

This test validates that the workflow can work correctly for clusters resource type. It verifies that when a cluster resource is created via the HyperFleet API, the system correctly processes the resource through its lifecycle, required adapters (configured in the test config) execute successfully, and accurately reports status transitions back to the API. The test validates required adapters first to identify specific failures, then confirms the cluster reaches the final Ready and Available state. This approach ensures the complete workflow of CLM can successfully handle clusters resource type requests end-to-end.

---

| **Field** | **Value**     |
|-----------|---------------|
| **Pos/Neg** | Positive      |
| **Priority** | Tier0         |
| **Status** | Automated     |
| **Automation** | Automated     |
| **Version** | MVP           |
| **Created** | 2026-01-29    |
| **Updated** | 2026-02-09    |


---

### Preconditions

1. Environment is prepared using [hyperfleet-infra](https://github.com/openshift-hyperfleet/hyperfleet-infra) with all required platform resources
2. HyperFleet API and HyperFleet Sentinel services are deployed and running successfully
3. The adapters defined in testdata/adapter-configs are all deployed successfully 

---

### Test Steps

#### Step 1: Submit an API request to create a Cluster resource

**Action:**
- Submit a POST request to create a Cluster resource:
```bash
curl -X POST ${API_URL}/api/hyperfleet/v1/clusters \
  -H "Content-Type: application/json" \
  -d @testdata/payloads/clusters/cluster-request.json
```

**Expected Result:**
- Response includes the created cluster ID and initial metadata
- Initial cluster conditions have `status: False` for both condition `{"type": "Ready"}` and `{"type": "Available"}`

#### Step 2: Verify initial status of cluster
**Action:**
- Poll cluster status for initial response
```bash
curl -X GET ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}
```

**Expected Result:**
- Cluster `Ready` condition `status: False`
- Cluster `Available` condition `status: False`

#### Step 3: Verify required adapter execution results

**Action:**
- Retrieve adapter statuses information:
```bash
curl -X GET ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}/statuses
```

**Expected Result:**
- Response returns HTTP 200 (OK) status code
- All required adapters from config are present in the response:
  - `clusters-namespace`
  - `clusters-job`
  - `clusters-deployment`
- Each required adapter has all required condition types: `Applied`, `Available`, `Health`
- Each condition has `status: "True"` indicating successful execution
- **Adapter condition metadata validation** (for each condition in adapter.conditions):
  - `reason`: Non-empty string providing human-readable summary of the condition state
  - `message`: Non-empty string with detailed human-readable description
  - `last_transition_time`: Valid RFC3339 timestamp of the last status change
- **Adapter status metadata validation** (for each required adapter):
  - `created_time`: Valid RFC3339 timestamp when the adapter status was first created
  - `last_report_time`: Valid RFC3339 timestamp when the adapter last reported its status
  - `observed_generation`: Non-nil integer value equal to 1 for new creation requests

**Note:** Required adapters are configurable via:
- Config file: `configs/config.yaml` under `adapters.cluster`
- Environment variable: `HYPERFLEET_ADAPTERS_CLUSTER` (comma-separated list)

#### Step 4: Verify final cluster state

**Action:**
- Wait for cluster Ready condition to transition to True
- Retrieve final cluster status information:
```bash
curl -X GET ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}
```

**Expected Result:**
- Cluster `Ready` condition transitions from `status: False` to `status: True`
- Final cluster conditions have `status: True` for both condition `{"type": "Ready"}` and `{"type": "Available"}`
- Validate that the observedGeneration for the Ready and Available conditions is 1 for a new creation request
- This confirms the cluster has reached the desired end state

#### Step 5: Cleanup resources

**Action:**
- Delete the namespace created for this cluster:
```bash
kubectl delete namespace {cluster_id}
```

**Expected Result:**
- Namespace and all associated resources are deleted successfully

**Note:** This is a workaround cleanup method. Once CLM supports DELETE operations for "clusters" resource type, this step should be replaced with:
```bash
curl -X DELETE ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}
```

---

## Test Title: Clusters Resource Type - K8s Resources Check Aligned with Preinstalled Clusters Related Adapters Specified

### Description

This test verifies that Kubernetes resources are successfully created with correct templated values for all required cluster adapters. The test dynamically reads the list of required adapters from config, waits for each adapter to complete execution, then validates that corresponding Kubernetes resources (Namespace, Job, Deployment) exist with properly rendered metadata (labels, annotations) matching the cluster request payload. This ensures adapter Kubernetes resource management and templating work correctly across all configured adapters.

---

| **Field** | **Value**     |
|-----------|---------------|
| **Pos/Neg** | Positive      |
| **Priority** | Tier0         |
| **Status** | Automated     |
| **Automation** | Automated     |
| **Version** | MVP           |
| **Created** | 2026-01-29    |
| **Updated** | 2026-02-11    |


---

### Preconditions

1. Environment is prepared using [hyperfleet-infra](https://github.com/openshift-hyperfleet/hyperfleet-infra) with all required platform resources
2. HyperFleet API and HyperFleet Sentinel services are deployed and running successfully 
3. The adapters defined in testdata/adapter-configs are all deployed successfully

---

### Test Steps

#### Step 1: Submit an API request to create a Cluster resource

**Action:**
- Submit a POST request to create a Cluster resource:
```bash
curl -X POST ${API_URL}/api/hyperfleet/v1/clusters \
  -H "Content-Type: application/json" \
  -d @testdata/payloads/clusters/cluster-request.json
```

**Expected Result:**
- Response includes the created cluster ID and initial metadata
- Initial cluster conditions have `status: False` for both condition `{"type": "Ready"}` and `{"type": "Available"}`

#### Step 2: Wait for all required adapters to complete

**Action:**
- Poll adapter statuses until all required adapters complete execution:
```bash
curl -X GET ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}/statuses
```

**Expected Result:**
- All required adapters from config (cl-namespace, cl-job, cl-deployment) are present
- Each adapter has all three conditions (`Applied`, `Available`, `Health`) with `status: True`

**Note:** Required adapters are configurable via `configs/config.yaml` under `adapters.cluster`

#### Step 3: Verify Kubernetes resources for each adapter with correct metadata

**Action:**
- For each required adapter, retrieve and validate corresponding Kubernetes resources:

**For cl-namespace adapter:**
```bash
kubectl get namespace {cluster_id} -o yaml
```

**Expected Result:**
- Namespace exists with name matching the cluster ID
- Namespace status phase is `Active`
- Required annotations:
  - `hyperfleet.io/generation`: Equals "1" for new creation request

**For cl-job adapter:**
```bash
kubectl get job -n {cluster_id} -l hyperfleet.io/cluster-id={cluster_id},hyperfleet.io/resource-type=job -o yaml
```

**Expected Result:**
- Job exists in the cluster namespace, identified by the label selector
- Job has completed successfully (status.succeeded > 0 or status.conditions contains type=Complete with status=True)
- Required annotations:
  - `hyperfleet.io/generation`: Equals "1" for new creation request

**For cl-deployment adapter:**
```bash
kubectl get deployment -n {cluster_id} -l hyperfleet.io/cluster-id={cluster_id},hyperfleet.io/resource-type=deployment -o yaml
```

**Expected Result:**
- Deployment exists in the cluster namespace, identified by the label selector
- Deployment is available (status.availableReplicas > 0 and status.conditions contains type=Available with status=True)
- Required annotations:
  - `hyperfleet.io/generation`: Equals "1" for new creation request

#### Step 4: Cleanup resources

**Action:**
- Delete the namespace created for this cluster:
```bash
kubectl delete namespace {cluster_id}
```

**Expected Result:**
- Namespace and all associated resources are deleted successfully

**Note:** This is a workaround cleanup method. Once CLM supports DELETE operations for "clusters" resource type, this step should be replaced with:
```bash
curl -X DELETE ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}
```

---

## Test Title: Clusters Resource Type - Adapter Dependency Relationships Workflow Validation

### Description

This test validates that CLM correctly handles adapter dependency relationships when processing a clusters resource request. Specifically, it verifies the dependency relationship where the cl-deployment adapter depends on the cl-job adapter completion. The test continuously polls and validates throughout the workflow period to ensure: (1) cl-deployment's Applied condition remains False until cl-job's Available condition reaches True, enforcing the dependency precondition; (2) during cl-job execution, cl-deployment's Available condition stays Unknown (never False), confirming the adapter waits correctly without attempting execution; (3) successful completion with cl-deployment's Available eventually transitioning to True. This validation demonstrates that the workflow engine properly enforces adapter dependencies and ensures dependent adapters wait for prerequisites before executing.

---

| **Field** | **Value**     |
|-----------|---------------|
| **Pos/Neg** | Positive      |
| **Priority** | Tier0         |
| **Status** | Automated     |
| **Automation** | Automated     |
| **Version** | MVP           |
| **Created** | 2026-01-29    |
| **Updated** | 2026-02-11    |


---

### Preconditions

1. Environment is prepared using [hyperfleet-infra](https://github.com/openshift-hyperfleet/hyperfleet-infra) with all required platform resources
2. HyperFleet API and HyperFleet Sentinel services are deployed and running successfully 
3. The adapters defined in testdata/adapter-configs are all deployed successfully

---

### Test Steps

#### Step 1: Submit an API request to create a Cluster resource
**Action:**
- Submit a POST request to create a Cluster resource:
```bash
curl -X POST ${API_URL}/api/hyperfleet/v1/clusters \
  -H "Content-Type: application/json" \
  -d @testdata/payloads/clusters/cluster-request.json
```

**Expected Result:**
- API returns successful response

#### Step 2: Verify cl-deployment initial state and dependency waiting behavior

**Action:**
- Poll adapter statuses to capture cl-deployment's initial waiting state:
```bash
curl -X GET ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}/statuses
```

**Expected Result:**
At the initial state (when cl-deployment first appears in statuses):
- Response returns HTTP 200 (OK) status code
- The `cl-deployment` adapter is present with initial waiting state:
  - `Applied` condition has `status: "False"` (deployment hasn't been applied yet, waiting for cl-job dependency)
  - `Available` condition has `status: "Unknown"` (deployment hasn't been applied yet)
  - `Health` condition has `status: "True"` (adapter itself is healthy, just waiting)

#### Step 3: Verify dependency relationship and condition transitions throughout entire workflow

**Action:**
- Continuously poll adapter statuses from the initial state until cl-deployment completes:
```bash
curl -X GET ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}/statuses
```

**Expected Result:**
Throughout the entire period (from initial state until cl-deployment completes), validate the following on each poll:

**Validation 1 - Dependency enforcement (during cl-job execution):**
- While `cl-job` adapter's `Available` condition has NOT reached `status: "True"`:
  - The `cl-deployment` adapter's `Applied` condition must remain `status: "False"`
  - The `cl-deployment` adapter's `Available` condition must remain `status: "Unknown"` (never `status: "False"`)
  - This validates that cl-deployment waits for cl-job to complete without attempting to apply resources

**Validation 2 - Success condition:**
- Once `cl-job` adapter's `Available` reaches `status: "True"`, cl-deployment can proceed with execution
- Once `cl-deployment` completes execution, its `Available` condition eventually becomes `status: "True"`
- This confirms the complete dependency workflow succeeded

**Note:** After cl-job completes, cl-deployment's `Available` condition may temporarily be `False` (e.g., `MinimumReplicasUnavailable` during deployment startup) before becoming `True`, which is expected behavior and not validated.

#### Step 4: Cleanup resources

**Action:**
- Delete the namespace created for this cluster:
```bash
kubectl delete namespace {cluster_id}
```

**Expected Result:**
- Namespace and all associated resources are deleted successfully

**Note:** This is a workaround cleanup method. Once CLM supports DELETE operations for "clusters" resource type, this step should be replaced with:
```bash
curl -X DELETE ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}
```

---

## Test Title: Cluster can reflect adapter failure in top-level status

### Description

This test validates that the end-to-end workflow correctly handles adapter failure scenarios. When an adapter's precondition configuration contains an invalid API endpoint URL, the adapter framework should detect the failure and report error status. The cluster's top-level conditions (`Ready`, `Available`) should remain `False`, accurately reflecting that the cluster has not reached a healthy state. This is a common configuration error scenario when external teams implement their own adapters.

---

| **Field** | **Value** |
|-----------|-----------|
| **Pos/Neg** | Negative |
| **Priority** | Tier1 |
| **Status** | Automated |
| **Automation** | Automated |
| **Version** | MVP |
| **Created** | 2026-02-11 |
| **Updated** | 2026-03-19 |


---

### Preconditions

1. Environment is prepared using [hyperfleet-infra](https://github.com/openshift-hyperfleet/hyperfleet-infra) with all required platform resources
2. HyperFleet API and HyperFleet Sentinel services are deployed and running successfully

---

### Test Steps

#### Step 1: Deploy dedicated precondition-error-adapter with invalid precondition URL
**Action:**
- Deploy a precondition-error-adapter via Helm with AdapterConfig containing a precondition that references an invalid API endpoint URL, separate from the normal adapters used in other tests. For example:
```yaml
preconditions:
  - name: "clusterStatus"
    apiCall:
      method: "GET"
      url: "http://invalid-service:8080/api/nonexistent"
    capture:
      - name: "clusterName"
        field: "name"
```

**Expected Result:**
- precondition-error-adapter is deployed and running successfully

#### Step 2: Submit an API request to create a Cluster resource

**Action:**
- Submit a POST request to create a Cluster resource:
```bash
curl -X POST ${API_URL}/api/hyperfleet/v1/clusters \
  -H "Content-Type: application/json" \
  -d @testdata/payloads/clusters/cluster-request.json
```

**Expected Result:**
- API returns successful response with cluster ID

#### Step 3: Verify adapter failure is reported via status API

**Action:**
- Poll adapter statuses until the precondition-error-adapter reports its status:
```bash
curl -X GET ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}/statuses
```

**Expected Result:**
- The precondition-error-adapter is present in the statuses response
- The adapter reports `Applied` condition with `status: "False"`
- The adapter reports `Available` condition with `status: "False"`
- The adapter reports `Health` condition with `status: "False"`, with reason and message indicating precondition failure details

#### Step 4: Verify cluster top-level status reflects adapter failure

**Action:**
- Retrieve cluster status:
```bash
curl -X GET ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}
```

**Expected Result:**
- Cluster `Ready` condition remains `status: "False"`
- Cluster `Available` condition remains `status: "False"`
- Cluster does not transition to Ready state while any adapter reports failure

#### Step 5: Cleanup Resources (AfterEach)

**Action:**
- Delete the namespace created for this cluster:
```bash
kubectl delete namespace {cluster_id}
```
- Uninstall the precondition-error-adapter Helm release
- Clean up the Pub/Sub subscription created by the adapter (if using Google Pub/Sub broker):
```bash
gcloud pubsub subscriptions delete {subscription_id} --project={project_id}
```

**Expected Result:**
- Namespace and all associated resources are deleted successfully
- precondition-error-adapter deployment is removed
- Pub/Sub subscription is deleted (if applicable)

**Note:** This is a workaround cleanup method. Once CLM supports DELETE operations for "clusters" resource type, the namespace deletion should be replaced with:
```bash
curl -X DELETE ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}
```

---

## Test Title: Cluster can reach correct status after adapter crash and recovery

### Description

This test validates the system's self-healing capability. When an adapter crashes during cluster processing, the system should ensure that the cluster's status is eventually reported correctly after the adapter recovers. This confirms that no cluster is left in an inconsistent state due to adapter failures.

---

| **Field** | **Value** |
|-----------|-----------|
| **Pos/Neg** | Negative |
| **Priority** | Tier2 |
| **Status** | Draft |
| **Automation** | Not Automated |
| **Version** | MVP |
| **Created** | 2026-02-11 |
| **Updated** | 2026-03-27 |


---

### Preconditions

1. Environment is prepared using [hyperfleet-infra](https://github.com/openshift-hyperfleet/hyperfleet-infra) with all required platform resources
2. HyperFleet API and HyperFleet Sentinel services are deployed and running successfully
3. The adapters defined in testdata/adapter-configs are all deployed successfully

---

### Test Steps

#### Step 1: Simulate adapter crash by scaling down one required adapter

**Action:**
- Scale down one of the required adapters (e.g., `cl-namespace`) to simulate a crash:
```bash
kubectl scale deployment ${ADAPTER_DEPLOYMENT_NAME} --replicas=0
```

**Expected Result:**
- Adapter becomes unavailable

> **Note:** The original test design proposed using `SIMULATE_RESULT=crash` to make the adapter pod crash. However, `SIMULATE_RESULT` only affects the Job container within the adapter's task execution, not the adapter pod itself. Scaling down the deployment is used as a practical alternative to simulate adapter unavailability.

#### Step 2: Submit an API request to create a Cluster resource

**Action:**
- Submit a POST request to create a Cluster resource:
```bash
curl -X POST ${API_URL}/api/hyperfleet/v1/clusters \
  -H "Content-Type: application/json" \
  -d @testdata/payloads/clusters/cluster-request.json
```

**Expected Result:**
- API returns successful response with cluster ID

#### Step 3: Verify the crashed adapter has not reported status

**Action:**
- Poll adapter statuses:
```bash
curl -X GET ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}/statuses
```
- Retrieve cluster status:
```bash
curl -X GET ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}
```

**Expected Result:**
- Statuses response does not contain an entry for the crashed adapter (e.g., `cl-namespace` is absent)
- Other required adapters have reported their statuses
- Cluster `Ready` condition remains `status: "False"`

#### Step 4: Restore adapter and verify cluster reaches correct status

**Action:**
- Scale up the adapter deployment back to 1 replica:
```bash
kubectl scale deployment ${ADAPTER_DEPLOYMENT_NAME} --replicas=1
```
- Poll adapter statuses until the restored adapter reports:
```bash
curl -X GET ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}/statuses
```
- Retrieve cluster status:
```bash
curl -X GET ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}
```

**Expected Result:**
- Restored adapter status entry is now present in the statuses response
- Restored adapter reports all three condition types with `status: "True"`: `Applied`, `Available`, `Health`
- `observed_generation` is set to `1`
- Cluster `Ready` condition transitions to `status: "True"`
- Cluster `Available` condition transitions to `status: "True"`
- This confirms no cluster is left in an inconsistent state due to adapter failures

#### Step 5: Cleanup Resources (AfterEach)

**Action:**
- Delete the namespace created for this cluster:
```bash
kubectl delete namespace {cluster_id}
```

**Expected Result:**
- Namespace and all associated resources are deleted successfully

**Note:** This is a workaround cleanup method. Once CLM supports DELETE operations for "clusters" resource type, the namespace deletion should be replaced with:
```bash
curl -X DELETE ${API_URL}/api/hyperfleet/v1/clusters/{cluster_id}
```

---
