# reserve

## Problem

Heavy agential workloads have a "Cold Start Dilemma":

1.  **They are heavy:** Bootstrapping a container (pulling 5GB images, loading models into RAM, mounting volumes) takes minutes.
2.  **They are bursty:** Users need them instantly, but only use them for short sessions.

You are stuck with two bad options:
* **Kubernetes Job:** Save money, but the user waits 3 minutes for the container to boot. 
* **Deployment:** Instant access, but you pay for idle GPUs 24/7.

## Solution

`reserve` introduces a new primitive that sits between "Running" and "Deleted."

It allows you to **pre-warm** a workload (perform the heavy boot process) and then **pause** it (delete the compute/Pod but preserve the PVC and Identity).

* **Reserve (CRD):** It holds the PVCs and the PodTemplate. It can toggle `spec.paused` to drop the Pod without losing the data.
* **ReserveSet (CRD):** A pool manager (like a ReplicaSet) that maintains a buffer of Warm environments ready for instant claiming.

## Usage

### 1. Define a Warm Pool
Create a `ReserveSet` to maintain a pool of initialized-but-paused environments.

```yaml
apiVersion: pool.w-h-a.io/v1alpha1
kind: ReserveSet
metadata:
  name: pytorch-agents
spec:
  replicas: 10
  template:
    metadata:
      labels:
        app: agent
    spec:
      containers:
      - name: main
        image: my-heavy-ai-image:v1
        volumeMounts:
        - name: model-cache
          mountPath: /data
```

### 2. Claim a Reserve
When a user arrives, your control plane (or a separate binding controller) claims a specific Reserve by unpausing it.

```yaml
apiVersion: pool.w-h-a.io/v1alpha1
kind: Reserve
metadata:
  name: pytorch-agents-x9d2s # One of the reserves from the set
spec:
  paused: false # Wakes up the Pod immediately
```

Because the PVCs are already bound and the image is cached on the node from the pre-warm phase, the pod starts in seconds instead of minutes.

## Architecture

### The Reserve Controller
Acts as a state machine for a single identity.
* **State: Running:** Ensures a Pod exists matching the template.
* **State: Paused:** Gracefully terminates the Pod but **preserves** the `Reserve` object and its associated `PersistentVolumeClaims`.
* **State Transition:** When switching from `Paused` -> `Running`, it re-attaches the *same* PVCs to the new Pod, preserving the "Brain" of the agent.

### The ReserveSet Controller
Acts as the capacity manager.
* Uses an **Expectation Cache** (Token Bucket) to manage bursty creations without spamming the Kubernetes API.
* Monitors the pool size and creates new `Reserve` objects to maintain the desired `replicas` count.
