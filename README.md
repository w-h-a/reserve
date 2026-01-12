# reserve

## Problem

Many Agents (or LLM-backed) workloads are **episodic** and **stateful**. They need to run generated code, manipulate large files, and retain variables across a conversation.

* **The Latency Trap:** Initializing a secure sandbox with Python libraries and mounting a user's dataset takes minutes. This destroys the chat experience.
* **The Cost Trap:** Keeping a sandbox running while the user thinks wastes massive amounts of compute.

Standard Kubernetes primitives fail here:
* **Jobs** are too slow (Cold start every time).
* **Deployments** are stateless (Cannot remember variables between requests) and expensive (Pay for idle time).

## Solution

`reserve` creates a **warm pool of stateful sandboxes**.

1.  **Pre-Warm:** It boots the heavy environment (Python, System Libs) before the user arrives.
2.  **Hibernate:** It pauses the environment by deleting the compute (Pod) but locking the storage (PVC) to a `Reserve` identity.
3.  **Resume:** When the agent needs to run code, the `Reserve` wakes up instantly with all files and context intact.

## Usage

### 1. Define a Warm Pool of Sandboxes
Create a `ReserveSet` to maintain a pool of initialized-but-paused environments.

```yaml
apiVersion: pool.w-h-a.io/v1alpha1
kind: ReserveSet
metadata:
  name: python-sandboxes
spec:
  replicas: 10
  template:
    metadata:
      labels:
        role: code-interpreter
    spec:
      containers:
      - name: runtime
        image: python:3.11-heavy-v4
        resources:
          limits:
            nvidia.com/gpu: 1
        volumeMounts:
        - name: workspace
          mountPath: /home/agent/workspace
```

### 2. Claim a Sandbox (The Agent Session)
When a user starts a chat, your control plane claims a specific `Reserve` by unpausing it.

```yaml
apiVersion: pool.w-h-a.io/v1alpha1
kind: Reserve
metadata:
  name: python-sandboxes-x9d2s # One of the reserves from the set
spec:
  paused: false # Wakes up the Pod immediately
```

* **Result:** The Pod launches in < 2 seconds because the image is cached on the node and the volume is already bound.
* **Action:** The Agent writes `data.csv` to `/home/agent/workspace`.
* **Pause:** When the user stops typing, you set `paused: true`. The GPU is released, but `data.csv` remains locked to this `Reserve`.

## Architecture

### The Reserve Controller (The Session Manager)
Acts as a state machine for a single agent identity.
* **State: Running:** Ensures a Pod exists matching the template.
* **State: Paused:** Gracefully terminates the Pod but **preserves** the `Reserve` object and its associated `PersistentVolumeClaims`.
* **State Transition:** When switching from `Paused` -> `Running`, it re-attaches the *same* PVCs to the new Pod, preserving the brain (files/context) of the agent.

### The ReserveSet Controller (The Pool Manager)
Acts as the capacity manager for the platform.
* **Warm Buffer:** Monitors the pool size and creates new `Reserve` objects to maintain the desired `replicas` count.
* **Scale Control:** Uses an **expectation cache** to prevent thundering herd issues during scale-up events without spamming the Kubernetes API.
