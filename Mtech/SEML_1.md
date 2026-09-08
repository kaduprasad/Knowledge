# Topic 1: Docker and Kubernetes Architecture

## Why This Topic Comes First

The September 2026 regular endsem devoted **12 of 40 marks** to Kubernetes:

- Q2: Kubernetes scale-up sequence and architecture, **6 marks**.
- Q4(a): Multi-container Pod YAML, **3 marks**.
- Q4(b): Kubernetes commands, **3 marks**.

Lecture source: [`SEML_all_paginated_compressed.pdf`](../../materials/lecture/SEML_all_paginated_compressed.pdf), pages 111-118.

![Docker and Kubernetes architecture](../assets/seml_kubernetes_architecture.png)

## Step 1: Understand the Problem Docker Solves

An application may work in development but fail in testing or production because the operating system, runtime, libraries, or configuration differs. Docker packages the application and its dependencies into a portable unit so the same environment can be reproduced.

**Plain-language memory line:** Docker answers, **"How do I package and run this application consistently?"**

For an ML inference service, the container can include:

- Inference code such as a Python script or FastAPI service.
- The trained model artifact.
- Libraries such as TensorFlow, PyTorch, scikit-learn, or NumPy.
- The Python runtime and required OS libraries.
- Configuration, preprocessing, and postprocessing code.

## Step 2: Distinguish Dockerfile, Image, and Container

| Term | Meaning | Easy analogy | State |
|---|---|---|---|
| **Dockerfile** | Text instructions used to build an image | Recipe | Source/build instructions |
| **Docker image** | Read-only package containing code, runtime, libraries, and tools | Prepared template | Stored artifact |
| **Docker container** | A running instance of an image | Dish made from the recipe | Running process |
| **Registry** | Repository used to push, store, and pull images | Image library | Shared storage |

The relationship is:

$$
\boxed{\text{Dockerfile} \xrightarrow{\texttt{docker build}} \text{Image}
\xrightarrow{\texttt{docker run}} \text{Container}}
$$

One image can create multiple containers. Changing a running container does not automatically change the original image.

## Step 3: Distinguish Docker from Kubernetes

Docker and Kubernetes solve different problems.

| Docker | Kubernetes |
|---|---|
| Builds and runs containers | Orchestrates containerized applications across a cluster |
| Focuses on packaging and one container runtime environment | Focuses on desired state, scheduling, scaling, recovery, and networking |
| Uses Dockerfile, image, container, and registry | Uses Deployment, ReplicaSet, Pod, Service, control plane, and worker nodes |
| Answers **how the application is packaged** | Answers **where and how many instances should run** |

**Exam trap:** Do not write that Kubernetes replaces the container image. Kubernetes schedules Pods whose containers are created from images.

## Step 4: Understand the Kubernetes Cluster

A Kubernetes cluster contains a **control plane** and one or more **worker nodes**.

### Control Plane Components

| Component | Exact job | What to write in an exam |
|---|---|---|
| **API server** | Exposes the Kubernetes API and receives commands or manifest changes | Central entry point through which components communicate with the cluster |
| **etcd** | Distributed key-value store holding cluster configuration and state | Source of truth for desired and current cluster state |
| **Controller manager** | Runs reconciliation controllers | Detects a difference between desired and current state and acts to correct it |
| **Scheduler** | Assigns unscheduled Pods to suitable worker nodes | Considers resources, node availability, and policies; it does not start containers itself |

### Worker Node Components

| Component | Exact job | What to write in an exam |
|---|---|---|
| **Kubelet** | Node agent that watches assigned Pod specifications | Ensures the required containers are created, running, and healthy |
| **Container runtime** | Pulls images and starts/stops containers | Performs container execution on the node |
| **Kube-proxy** | Maintains Pod and Service network connectivity | Implements network rules and traffic forwarding on the node |
| **Pod** | Smallest deployable Kubernetes unit | Holds one or more containers that share network and storage resources |

### Client-Side Tool

**kubectl** is the command-line client. It sends requests to the API server to deploy applications, inspect resources, view status, and retrieve logs. It is not a worker-node component.

## Step 5: Learn the Kubernetes Object Hierarchy

$$
\boxed{\text{Deployment} \rightarrow \text{ReplicaSet} \rightarrow \text{Pod}
\rightarrow \text{Container} \rightarrow \text{Application}}
$$

| Object | Responsibility |
|---|---|
| **Deployment** | Declares the application template and desired replica count; manages updates |
| **ReplicaSet** | Ensures that the requested number of matching Pods exists |
| **Pod** | Provides the shared execution boundary for one or more containers |
| **Container** | Runs the packaged application process from an image |
| **Service** | Provides a stable IP/DNS endpoint for a changing group of Pods |

### Example

Suppose a Deployment specifies `replicas: 5` for an ML inference application:

1. The **Deployment** declares that five instances are desired.
2. Its **ReplicaSet** maintains five matching Pods.
3. The **scheduler** selects worker nodes for newly created Pods.
4. Each node's **kubelet** asks the container runtime to pull the image and start the container.
5. A **Service**, when configured, gives clients a stable endpoint and routes traffic to healthy Pods.

The detailed event-by-event reconciliation is Topic 2.

## Step 6: Understand Desired State and Reconciliation

Kubernetes is declarative. The manifest states the result we want, not every command required to reach it.

Let:

- $D$ = desired number of replicas in the Deployment.
- $C$ = current number of running replicas.
- $\Delta$ = number of replicas that must be created or removed.

$$
\Delta = D-C
$$

If $D=5$ and $C=1$:

**Formula**

$$
\Delta = D-C
$$

**Substitute values**

$$
\Delta = 5-1
$$

**Result**

$$
\boxed{\Delta=4\text{ additional Pods}}
$$

The controllers repeatedly compare desired state with current state. This control loop is called **reconciliation**.

## Step 7: Map Components to the September 2026 PYQ

### Real Problem Statement

**SEML EC3 Regular, 5 September 2026, Q2, 6 marks**

The paper gives a Kubernetes Deployment manifest for an ML inference service. Its replica count is changed from 1 to 5 and deployed through a Continuous Deployment tool. The question asks for the detailed internal sequence of events inside the cluster and requires all Kubernetes architecture components.

### Given Data

| Quantity | Value | Source |
|---|---:|---|
| Initial replicas, $C$ | 1 | Manifest shown in Q2 |
| Desired replicas, $D$ | 5 | Change stated in Q2 |
| Additional Pods | $5-1=4$ | Desired state minus current state |
| Deployment name | `ml-inference-deployment` | Manifest shown in Q2 |
| Container name | `ml-inference-container` | Manifest shown in Q2 |
| Image | `ml-inference:latest` | Manifest shown in Q2 |
| Container port | 8000 | Manifest shown in Q2 |

### Architecture Components the Answer Must Name

1. Continuous Deployment tool or `kubectl` client.
2. API server.
3. etcd.
4. Deployment controller in the controller manager.
5. ReplicaSet and ReplicaSet controller.
6. Scheduler.
7. Worker nodes.
8. Kubelet.
9. Container runtime.
10. Kube-proxy/networking.
11. Pods and containers.

Missing the scheduler, kubelet, or controller manager makes the answer incomplete because each controls a different stage.

## Exam-Ready Component Flow

Use this compact chain when first planning the answer:

$$
\boxed{
\text{CD tool}
\rightarrow \text{API server}
\rightarrow \text{etcd}
\rightarrow \text{controllers}
\rightarrow \text{ReplicaSet/Pods}
\rightarrow \text{scheduler}
\rightarrow \text{kubelet/runtime}
\rightarrow \text{running application}
}
$$

This is only the skeleton. Topic 2 will expand it into the complete six-mark sequence, including status updates and convergence.

## Common Exam Traps

1. **Scheduler starts containers:** Incorrect. The scheduler assigns a Pod to a node; the kubelet and container runtime start its containers.
2. **etcd creates Pods:** Incorrect. etcd stores state; controllers act on that state.
3. **Deployment directly runs containers:** Incomplete. A Deployment manages a ReplicaSet, which maintains Pods.
4. **Pod and container are synonyms:** Incorrect. A Pod can contain one or more containers with shared network/storage resources.
5. **kubectl is part of the control plane:** Incorrect. It is a client that communicates with the API server.
6. **Service and Pod are interchangeable:** Incorrect. A Service supplies stable networking for a changing set of Pods.
7. **Five replicas means five extra Pods:** Incorrect here. The system already has one, so it needs four additional Pods.

## Closed-Book Recall

Answer these without looking back:

1. What is the difference between a Dockerfile, image, and container?
2. What problem does Kubernetes solve that Docker alone does not solve?
3. Name the four control-plane components and give one job for each.
4. Name the three main worker-node components and give one job for each.
5. Expand: Deployment -> ReplicaSet -> Pod -> Container.
6. Why does changing replicas from 1 to 5 create four, not five, additional Pods?
7. Which component selects the worker node, and which component actually starts the container?

## One-Minute Summary

- **Docker packages** the ML application and dependencies into an image and runs it as a container.
- **Kubernetes orchestrates** containers across worker nodes.
- The **API server** receives requests; **etcd** stores state; **controllers** reconcile state; the **scheduler** selects nodes.
- On each worker, the **kubelet** enforces the Pod specification, the **container runtime** runs containers, and **kube-proxy** supports networking.
- Remember: **Deployment -> ReplicaSet -> Pod -> Container**.
