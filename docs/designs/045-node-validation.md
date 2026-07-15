# ADR-045: Node Validation

## Context

If a cluster has deployed GPU Operator and NVSentinel, there is no mechanism to ensure that new GPU nodes or existing nodes which underwent remediation are validated prior to being marked as schedulable. New nodes are able to accept GPU workloads after the nvidia-device-plugin advertises GPU capacity on a given node. Additionally, existing nodes which were remediated by NVSentinel are able to accept GPU workloads after the fault-quarantine module unquarantines them. In either case, there is no ability for an operator to declare that a set of GPU performance tests must succeed prior to a node being returned to service.

Note that NVSentinel remediation actions do not include any verification that the remediation action performed successfully resolved the underlying GPU fault (outside of the same fault re-occurring and being re-emitted by the corresponding health monitor). Similarly, new nodes joining a cluster could represent either new hardware being deployed or existing hardware being re-provisioned. In either case, this hardware should be validated, especially if the underlying hardware previously belonged to a node that was terminated and returned for repair to the given provider.

A verification test in both contexts ensures that a given node meets its minimum performance requirements and reduces the probability that a customer workload will encounter a fatal error. A scheduling gate on completing a verification test gives an opportunity for NVSentinel to increase the scope of its health checking by performing:

* **GPU performance tests:** such as NCCL or Nemotron tests, which require exclusive GPU access
* **Disruptive health checks:** such as DCGM diagnostics
* **Inducing faults:** a performance or active health check may induce a fault that would otherwise only surface when a real customer workload runs

## Decision

We will introduce a validation-controller whose purpose is to reconcile validation requests for nodes. Clients can interact with a validation API and request that a set of nodes and corresponding GPUs be validated by the validation-controller. The 2 use-cases we are targeting include:

1. Validating new nodes
2. Validating existing nodes post-remediation

The validation-controller will orchestrate running validation across different clients, nodes, GPUs, and tests through a set of supported test providers. Test providers will create a Kubernetes resource that must be reconciled by a separate controller to execute the test. For example, we will start with supporting a K8s job provider which can validate a subset or all GPUs on a single node along with a NCCL provider which can validate GPUs across multiple nodes. As a result, it will be the responsibility of the validation-controller to map a set of validation requests across different nodes, GPUs, and tests to a set of provider requests.

**Why do we need a validation-controller?**

An alternative solution that avoids the need for a validation-controller would be for all clients requesting validation to directly interface with the supported test providers. This approach is not desirable because the validation-controller comes with the following benefits:

* **Support for validation sessions**: a quarantine session in NVSentinel is made up of a group of faults all of which must be remediated prior to the node being unquarantined. A similar capability is provided with the validation-controller where all validation requests against a given node must be satisfied prior to the node being uncordoned.
    * For example, if a new node requires validation and experiences a fault that results in NVSentinel also requesting validation, both sets of validation tests must succeed prior to the node being uncordoned.
* **Common validation interface:** a shared interface for requesting validation prevents all clients from needing to maintain logic to work with the different test providers, including our supported K8s job and NCCL test providers. A client who wants to validate a group of nodes or GPUs only needs to know the test name and we will maintain a common entrypoint in the validation-controller.
* **Batching multiple validation requests:** if a single client independently creates validation requests across nodes or multiple clients request validation, a dedicated controller provides the opportunity to batch multiple validation requests together. This allows for de-duplicating validation requests across the same node for the same test and also batching nodes together in multi-node tests. This prevents clients from needing to handle batching on their side prior to issuing validation requests.
    * For example, if NVSentinel remediates 2 nodes and requests that both undergo a multi-node NCCL test, we have the opportunity to include those 2 nodes together in a single test from our NCCL provider.
* **Batching multiple nodes into a single validation request:** a validation request will allow targeting a single node or a group of nodes (along with a single GPU or all GPUs on a single node). If a test provider only supports targeting a single node, such as the K8s job provider, the validation-controller will handle creating a validation job per node without requiring the requesting client to create and track the execution of each job. Clients will only need to monitor the overall status for their validation request.

## Validation API and Configuration

In order to run validation tests against GPU nodes, a cluster operator will need the ability to define a set of supported validation tests. Additionally, a client requesting that a GPU node undergoes validation will need to provide the following inputs:

```
nodes:
  - name: gpu-node-01
    gpusImpacted:
      - GPU-abc123
    exclusiveAccess: true
  - name: gpu-node-02
    gpusImpacted:
      - GPU-def456
    exclusiveAccess: false
  - name: gpu-node-03
    gpusImpacted: []
    exclusiveAccess: true
tests:
  - dcgm-level4
  - nccl-loopback
```

**Validation API overview:**

* **nodes / gpusImpacted:** the client will need to provide the impacted nodes along with the list of impacted GPUs per node as API input. If no GPUs are provided, we will assume that all GPUs on the given node need to be validated.
* **tests:** the client will need to pass the set of tests which should be executed against the given entities. While the ValidationConfiguration will support a set of default tests, clients will need the ability to specify which tests should be run depending on if a new node is being validated or if a specific fault was encountered.
    * All entities provided in the validation request will be targeted by the same set of tests. If a client would like to request a different set of tests depending on the impacted entities, they will need to create multiple validation requests.
* **exclusiveAccess:** a boolean which indicates whether the node is fully drained. If true, the node is idle and any GPU may be validated. If false, workloads may still be running on GPUs not included in the list of impacted entities, so we require at least 1 GPU to be provided in the list of impacted entities.
    * For example, if NVSentinel encountered an XID which required a GPU reset, only the impacted GPU will be eligible for validation due to the partial node drain executed before the reset. Conversely, if the XID required a node reboot, the node would be eligible for validation against all its GPUs due to the full node drain before the reboot.
    * This highlights that node draining is not the responsibility of the validation-controller and validation clients must execute node drains prior to creating requests and notifying the controller if the node is partially or fully drained.

The section below on "Triggering Validation" covers how clients will interact with this API and whether this input is passed as a CRD, an NVSentinel HealthEvent, or as part of the node object.

The provided list of tests in the validation request must map to a supported test in the ValidationConfiguration. Upon receiving a validation request, the validation-controller will reference the active ValidationConfiguration:

```
apiVersion: nvsentinel.nvidia.com/v1alpha1
kind: ValidationConfiguration
metadata:
  name: default
spec:
  autoCordon: false
  removeCordon: true
  validationReadinessCriteria:
  - type: AllocatableResource
    resource: nvidia.com/gpu
    operator: Gt
    value: "0"
  defaultTests: 
  - dcgm-level4
  - nccl-loopback
  testProviderConfig:
  - name: nccl-provider
    supportsTestBatching: true
    retries: 2
  - name: k8s-job-provider
    supportsTestBatching: false
    retries: 5
  batchPeriod: 5m
  tests:
    dcgm-level4:
      provider: nccl-provider
      exclusiveNodeAccess: false        
      supportsBatchingGPUsPerNode: true
      minimumGPUsPerNodePerBatch: 1 
      supportsBatchingNodes: true
      minimumNodesPerBatch: 1       
      batchFailurePolicy: fail
    nccl-loopback:
      provider: nccl-provider
      exclusiveNodeAccess: false        
      supportsBatchingGPUsPerNode: true
      minimumGPUsPerNodePerBatch: 2 
      supportsBatchingNodes: true
      minimumNodesPerBatch: 1       
      batchFailurePolicy: fail
    nccl-all-reduce:
      provider: nccl-provider
      exclusiveNodeAccess: false        
      supportsBatchingGPUsPerNode: true
      minimumGPUsPerNodePerBatch: 1 
      supportsBatchingNodes: true
      minimumNodesPerBatch: 2       
      batchFailurePolicy: fail
    nemotron4-15b:
      provider: nccl-provider
      exclusiveNodeAccess: true        
      supportsBatchingGPUsPerNode: true
      minimumGPUsPerNodePerBatch: 1 
      supportsBatchingNodes: true
      minimumNodesPerBatch: 18       
      batchFailurePolicy: open
    cuda-smoke-test:
      provider: k8s-job-provider
      exclusiveNodeAccess: false        
      supportsBatchingGPUsPerNode: true
      minimumGPUsPerNodePerBatch: 1 
      supportsBatchingNodes: false
      minimumNodesPerBatch: 1       
      batchFailurePolicy: fail
```

**Validation configuration overview:**

* **autoCordon**: indicates whether nodes needing validation should be cordoned by the validation-controller if not already cordoned. The cordon should only be applied after the validationReadinessCriteria is satisfied.
    * NVSentinel will leave nodes cordoned for post-remediation validation so this option would only be needed for cordoning new nodes eligible for validation.
* **removeCordon:** indicates whether nodes should be uncordoned after completing validation. A node will only be uncordoned once there are no pending validation requests (a node may be targeted by multiple validation requests).
    * This behavior is similar to the fault-quarantine module where a given quarantine session requires all unhealthy events to recover prior to removing the cordon for a node.
* **validationReadinessCriteria:** a set of criteria which must be satisfied before a validation test can be started on a given node. This is primarily needed for new nodes undergoing validation to ensure that all GPU Operator and NVSentinel operands are ready on the given node.
    * In the provided example, we require the targeted node to report allocatable GPUs prior to running the requested test. We can extend this to supporting node status conditions or a given pod on the node being in ready status.
* **defaultTests:** the default set of tests which will be run against validation requests which do not include any tests. This option will be useful if the same set of tests will be run for validating new or existing nodes and if the operator does not want to make validation clients aware of test names.
* **testProviderConfig:** includes test provider settings that apply to all tests using this provider.
    * **supportsTestBatching**: indicates which test providers support batching their tests into a single test provider request. In the example above, the nccl-provider supports batching which allows all tests using that provider (dcgm-level4, nccl-loopback, nccl-all-reduce, and nemotron4-15b) to be included in a single CRD. Conversely, the k8s-job-provider does not support batching so the cuda-smoke-test will not be batched with any other tests using that provider.
    * **retries:** the number of retries we will allow for all tests referencing this test provider. Note that this setting is at the test provider level and not individual test level because if a test provider supports batching, all tests may need to be retried together.
* **batchPeriod:** the period that allows for batching of tests, nodes, and GPUs across validation requests. This is a global setting that covers all test providers.
* **tests:** the set of supported tests that can be requested by clients in validation requests. The test name is the only piece of the ValidationConfiguration that clients must be aware of when interacting with the validation API.

**Test configuration:**

* **provider:** indicates which test provider uses the given test. In our example we support a nccl-provider which is used for the dcgm-level4,  nccl-loopback,  nccl-all-reduce, and nemotron4-15b tests and a k8s-job-provider which is used for the cuda-smoke-test test.
* **exclusiveNodeAccess:** indicates whether this test requires exclusive access to the node.
    * If false, the client must pass at least 1 GPU as part of the validation request which will be used as test input. Even if the validation request sets exclusiveAccess to true, it will still be required to include explicit GPUs in its request.
    * If true, the test will require access to all GPUs, meaning it does not require specific GPUs to be included in the validation input. If this setting is true, the validation request must also set exclusiveAccess to true (meaning the node is fully drained).
* **supportsBatchingGPUsPerNode:** indicates whether this test supports testing multiple GPUs on a given node in a single test. For example, the k8s-job-provider could choose to either test GPU1 and GPU2 as part of the same underlying job or choose to create 1 job per GPU.
    * Note that if exclusiveNodeAccess is true, this also has to be true since we allow validation requests to pass an empty list of GPUs indicating all GPUs will be tested implicitly. However, if exclusiveNodeAccess is false, this setting could be true or false.
* **minimumGPUsPerNodePerBatch:** if supportsBatchingGPUsPerNode is true, this setting indicates the minimum number of GPUs that must be provided to the test. Note that the GPUs could be provided from the same validation request or from across multiple validation requests.
    * As an example, dcgm-level4 supports 1 GPU per batch whereas nccl-loopback requires 2 GPUs per batch.
* **supportsBatchingNodes:** indicates whether this test supports testing multiple nodes in a single test. For example, the k8s-job-provider will only support a single node per test so supportsBatchingNodes will be false. However, the nccl-provider has to support node batching to support multi-node NCCL tests so supportsBatchingNodes will be true.
* **minimumNodesPerBatch:** if supportsBatchingNodes is true, this setting indicates the minimum number of nodes that must be provided to the test. Note that the nodes could be provided from the same validation request or from across multiple validation requests.
    * As an example, nccl-loopback supports 1 node per batch whereas  nccl-all-reduce requires 2 nodes per batch.
* **batchFailurePolicy:** this setting defines what to do when a test cannot start due to it not meeting its configured minimumGPUsPerNodePerBatch or minimumNodesPerBatch. Possible values include:
    * **fail:** mark the overall validation request as failed if this individual test cannot be run.
    * **open:** do not run this individual test if the batch sizes are not satisfied and implicitly consider the test as successful.
    * **wait:** wait for the next batch window to see if other validation requests can help it meet its minimum requirements. This policy will result in other tests in the same validation request being blocked.

## Triggering Validation

**Option 1: Node object as the validation trigger (not recommended)**

A missing or false node status condition could be used to signal that a new or existing node needs to be validated. This approach would be similar to how the upstream node-readiness-controller detects whether a given node is ready to accept workloads.

We could define an arbitrary number of node status conditions which map to a given set of tests. However, unless we always validate all GPUs, clients will need the ability to pass the impacted GPUs and whether the node has been fully drained to the API. As a result, to support per-validation configuration rather than using the global defaults, we can leverage a node annotation similar to the structured quarantineHealthEvent annotation in the fault-quarantine module.

```
apiVersion: v1
kind: Node
metadata:
  name: gpu-node-01
  annotations:
    nvsentinel.nvidia.com/validation-request: |
      [
        {
          "nodeName": "gpu-node-01",
          "tests": ["dcgm-level4", "nccl-loopback"],
          "gpusImpacted": ["GPU-abc123", "GPU-def456"],
          "exclusiveAccess": false
        },
        {
          "nodeName": "gpu-node-02",
          "tests": ["nccl-all-reduce", "nemotron4-15b"],
          "exclusiveAccess": true
        }
      ]
...
status:
  conditions:
    - type: NewNodeValidation
      status: "True"
    - type: ExistingNodeValidation
      status: "False"
```

* **Validating new nodes:** new nodes will be validated if the NewNodeValidation condition is missing. Since the nvsentinel.nvidia.com/validation-request annotation is only relevant to existing nodes, we will run a default set of tests from the ValidationConfiguration and assume that exclusiveAccess is true.
* **Validating existing nodes:** existing nodes will be validated post remediation if the ExistingNodeValidation condition is false. A client can request validation-specific configuration through the nvsentinel.nvidia.com/validation-request annotation and the controller will fall back to defaults defined in the ValidationConfiguration if not provided.
* **Supporting multiple clients:** any client requesting node validation would need to be aware of both the node status condition names and the name and structure of the validation-request annotation. While the node status conditions could be derived from the ValidationConfiguration, a client outside of NVSentinel would need to maintain this structure independently.
* **Overlapping validations:** it’s possible for either multiple clients to request validation or for a new node validation to be superseded by a post-remediation validation. We would need to align all clients to either append their validation request to the existing annotation or allow clients to overwrite previous validations which may not have had a successful remediation.
* **Validating multiple nodes:** this approach does not allow a client to specify that a group of nodes should be validated and the client would have to write to each node object independently.
    * Note that it is still possible for multiple nodes to be validated together even if they all independently must declare validation is needed depending on the batching behavior for the required tests.

**Conclusion:** this approach would be recommended if all validation clients are in NVSentinel and if we only need to support declaring individual nodes as needing validation.

**Option 2: HealthEvent as the validation trigger (not recommended)**

Rather than require clients to update node object status conditions and annotations, we could update the HealthEvent API to support post-remediation validation. With a new HealthEvent status field, we could add the validation-controller as a new component in the breakfix workflow which is triggered after a node is unquarantined by fault-quarantine if the given fault requires post-remediation validation.

The section below on "Validation Clients" will cover how validation tests will be derived from all faults remediated during the current quarantine session. For now, we will assume that fault-quarantine already knows the set of validation tests required and is ready to invoke the validation API. After fault-quarantine receives the last healthy event which results in an unquarantine event for the given node, it could pass all required validation tests to the validation-controller via the HealthEvent API:

```
{
  "_id": "ObjectId('6906a646fa62bde28eafe3cf')",
  "createdAt": "2025-11-02T00:31:02.943Z",
  "healthevent": {
...
  },
  "healtheventstatus": {
    "nodequarantined": "UnQuarantined",
    "userpodsevictionstatus": {
      "status": "InProgress"
    },
    "faultremediated": null,
    "validationrequest": [
      {
        "nodeName": "10.0.8.174",
        "tests": ["dcgm-level4", "nccl-loopback"],
        "gpusImpacted": ["GPU-abc123"],
        "exclusiveAccess": false
      },
        {
          "nodeName": "gpu-node-02",
          "tests": ["nccl-all-reduce", "nemotron4-15b"],
          "exclusiveAccess": true
        }
    ]
  }
}
```

* **Validating new nodes:** we would need a dedicated health-monitor to monitor for new nodes and emit an unhealthy event with the required tests prior to emitting a healthy event to trigger the unquarantine and handoff to the validation-controller from fault-quarantine. An alternative approach would be to support triggering tests with a single event that includes the validation request and is emitted by a health-monitor directly (allowing a bypass of fault-quarantine).
* **Validating existing nodes:** as discussed above, existing nodes will be validated post remediation by having the fault-quarantine module request that an existing node be validated.
* **Supporting multiple clients:** we would be able to support multiple clients requesting validation by having each client interact with the HealthEvent API.
* **Overlapping validations:** we will allow multiple clients to write events requesting validation for the same node. The validation-controller will be responsible for de-duplicating or combining tests from multiple events targeting the same node. For example, if validation was requested both by fault-quarantine and for a new node, the controller would be responsible for handling the duplicated requests.
* **Validating multiple nodes:** this approach does not allow a client to specify that a group of nodes should be validated and will require 1 HealthEvent per node per GPU needing validation.

**Conclusion:** this approach would be recommended if all validation clients are in NVSentinel, we accept a HealthEvent API and MongoDB dependency on validation, and if we only need to support declaring individual nodes as needing validation.

**Option 3: CRD as the validation trigger (recommended)**

Rather than require clients to update node object status conditions and annotations or rely on the HealthEvent API, we could provide a ValidationRequest CRD to allow a client to declare that a particular node or a group of nodes should be validated.

```
apiVersion: nvsentinel.nvidia.com/v1alpha1
kind: ValidationRequest
metadata:
  name: gpu-node-validation
spec:
  nodes:
    - name: gpu-node-01
      gpusImpacted:
        - GPU-abc123
      exclusiveAccess: true
    - name: gpu-node-02
      gpusImpacted:
        - GPU-def456
      exclusiveAccess: false
    - name: gpu-node-03
      gpusImpacted: []
      exclusiveAccess: true
  tests:
    - dcgm-level4
    - nccl-loopback
```

* **Validating new nodes:** we will need the validation-controller to create ValidationRequests for new nodes. Creation of the CRD can be signaled by the absence of a node status condition called NewNodeValidation, which the controller then sets to true after validation completes.
* **Validating existing nodes:** existing nodes will be validated post remediation by having fault-quarantine create a ValidationRequest CRD directly.
* **Supporting multiple clients:** any client requesting node validation only needs to create a ValidationRequest CRD without needing to modify the node object or interact with the HealthEvent API.
* **Overlapping validations:** it’s possible for either multiple clients to request validation or for a new node validation to be superseded by a post-remediation validation. We allow any client internal or external to NVSentinel to create ValidationRequests.
    * Note that the validation-controller will handle de-duplicating or combining tests from multiple ValidationRequests targeting the same node (we will not allow ValidationRequests to be superseded by the latest CRD and rather require all requests are processed).
* **Validating multiple nodes:** while we currently only need to target a single node for either new nodes or existing nodes undergoing post-remediation, this approach does allow a client to specify that a group of nodes should be validated, which would support a client that needs to implement its own batching behavior.

## Validation Clients

Assuming that we are proceeding with option 3 above using a CRD as the validation trigger, we will need to ensure that a ValidationRequest CRD is created for nodes in the following contexts.

**New node validation**

The validation-controller can watch for new nodes missing a given node status condition to signal that it should create a ValidationRequest for new nodes. After a ValidationRequest is created, the controller will set the given status condition to true to ensure it does not create duplicated ValidationRequests for new nodes. The name of this status condition will be added to the ValidationConfiguration. Additionally, we can add a list of tests to the ValidationConfiguration that should be executed against new nodes to prevent each ValidationRequest from needing to populate the list of tests. This allows the configuration to specify both a default group of tests and a group of tests for new nodes. In this example, any ValidationRequest for a new node which does not provide an explicit list of tests will run nccl-all-reduce:

```
apiVersion: nvsentinel.nvidia.com/v1alpha1
kind: ValidationConfiguration
metadata:
  name: default
spec:
...
  newNodeValidation:
    type: NewNodeValidated
    status: "True"
  defaultTests: 
  - dcgm-level4
  - nccl-loopback
  newNodeTests:
  - nccl-all-reduce
...
```

Additionally, after we detect a node missing the NewNodeValidated condition, we will create a ValidationRequest with a new node property to indicate that the tests under newNodeTests should be executed against the node:

```
apiVersion: nvsentinel.nvidia.com/v1alpha1
kind: ValidationRequest
metadata:
  name: gpu-node-validation
spec:
  nodes:
    - name: gpu-node-01
      gpusImpacted: []
      exclusiveAccess: true
  tests: []
  newNode: true
```

**Post-remediation validation**

The fault-quarantine module will need the ability to create a ValidationRequest after an unquarantine event if any of the unhealthy events during that quarantine session requested validation. If validation is required, fault-quarantine will create a ValidationRequest, release ownership of the node, but retain the node cordon. The validation-controller will be responsible for removing the cordon after all requested validations complete against the node.

As a result, fault-quarantine will need an ability to derive validation tests from unhealthy events. Possible options for mapping unhealthy events to validation tests include:

* Add a rule-set evaluation to the fault-quarantine module to derive validation tests from unhealthy events.
    * This option mirrors the existing fault-quarantine rule-set evaluation to determine whether a given unhealthy event should be remediated as part of the current quarantine session.
* Populate the list of validation tests directly in HealthEvents from each health-monitor.

To prevent needing to modify each health-monitor, we will go with option 1 and maintain a rule-set which will be evaluated against each unhealthy event which contributes to the current quarantine session. As a result, if a given HealthEvent would result in a validation test but doesn’t contribute to the current quarantine session, we will not require that fault to be validated.

It’s important to note that the validation rule-set must be valid according to the ValidationConfiguration. Specifically, any fault which requires a test that needs exclusive node access should result in full drains. Otherwise, these will be rejected by the validation-controller. This can be optionally enforced within the rule-set itself by checking if the recommendedAction is COMPONENT\_RESET or not prior to recommending a given test.

The rule-set evaluation may result in zero or multiple tests being requested by validation. Here is an example validation rule-set that determines which tests to run per unhealthy event:

```
ruleSets:
  - name: syslog-xid-119-component-reset
    match:
      all:
        - kind: HealthEvent
          expression: >
            event.agent == 'syslog-health-monitor' &&
            event.recommendedAction == 'COMPONENT_RESET' &&
            event.errorCode.exists(e, e == '119')
    tests:
      - dcgm-level4

  - name: syslog-other
    match:
      all:
        - kind: HealthEvent
          expression: >
            event.agent == 'syslog-health-monitor' &&
            event.recommendedAction != 'COMPONENT_RESET'
    tests:
      - nccl-loopback

  - name: gpu-health-monitor-component-reset
    match:
      all:
        - kind: HealthEvent
          expression: >
            event.agent == 'gpu-health-monitor' &&
            event.recommendedAction == 'COMPONENT_RESET'
    tests:
      - dcgm-level4

  - name: gpu-health-monitor-other
    match:
      all:
        - kind: HealthEvent
          expression: >
            event.agent == 'gpu-health-monitor' &&
            event.recommendedAction != 'COMPONENT_RESET'
    tests:
      - nccl-loopback
```

After the tests have been determined, the other properties needed for the ValidationRequest are the node name, the GPUs impacted, and whether the node was fully or partially drained. The first 2 properties can be derived from the event directly. However, we will need to determine the value for exclusiveAccess based on the result of the node drain.

In general, we can set exclusiveAccess to false if the given unhealthy event had a COMPONENT\_RESET recommended action and set it to true on any other recommended action. However, it is possible that either full or partial drains are cancelled. As a result, we need to look up the drain status for all events needing validation (and were ever part of the quarantineHealthEvent annotation during the session). In other words, it is not safe to proceed with a ValidationRequest unless the event which required the given tests had a successful drain.

To determine the drain state of the node and which ValidationRequests are permitted, we can follow this procedure:

1. Describe each HealthEvent that is part of the current quarantine session to determine its drain status.
    1. Since HealthEvents can be resolved independently during a quarantine session, we will need to either evaluate HealthEvents when they are removed from the quarantineHealthEvent or track all HealthEvents and evaluate all of them when the unquarantine event occurs.
2. If the drain for the given HealthEvent is completed, accept the ValidationRequest from that fault.
3. For events with completed drains, set exclusiveAccess to false for COMPONENT\_RESET actions and set it to true for all other recommended actions.
4. Optionally, for events with cancelled drains, check if there’s overlap for the impacted entities of any completed drain. For example, if there was a completed drain against the same GPU or if any event had a full drain, this would permit the ValidationRequest from cancelled events.

In summary, evaluation of a HealthEvent requires both the validation rule-set execution to determine its corresponding tests and checking the drain status for the event. These 2 results can be used to construct a ValidationRequest.

Suppose that the quarantine session for a node had 3 unhealthy events which resulted in the following requests:

```
      [
        {
          "tests": ["dcgm-level4", "nccl-loopback"],
          "gpusImpacted": ["GPU-abc123"],
          "exclusiveAccess": false
        },
        {
          "tests": ["dcgm-level4", "nccl-loopback"],
          "gpusImpacted": ["GPU-abc123"],
          "exclusiveAccess": false
        },
        {
          "tests": ["nccl-all-reduce", "nemotron4-15b"],
          "gpusImpacted": ["GPU-abc456"],
          "exclusiveAccess": true
        }
      ]
```

After evaluating each event, the last step for fault-quarantine is to convert this state into ValidationRequest CRDs. Since the validation-controller will handle both batching and de-duplicating, we can choose to issue 1 ValidationRequest per HealthEvent or batch multiple HealthEvents into a single ValidationRequest. We have the following options:

1. **Batch all events into a single ValidationRequest:** since a ValidationRequest includes a list of tests which targets all impacted entities, batching all tests into a single request could result in us running a specific test on a GPU when it is not required.
    1. An alternative approach would be to allow batching of events into the same ValidationRequest if the entities across events require the same tests. This would minimize the number of ValidationRequests and prevent running tests against GPUs which weren’t requested. However, it would require re-implementing the batching behavior which is already available in the validation-controller.
2. **Create 1 ValidationRequest per event:** the simplest option would be to create 1 ValidationRequest per HealthEvent and rely on the validation-controller to de-duplicate any tests targeting the same node and GPUs.
