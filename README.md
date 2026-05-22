# Kubernetes Cluster Bootstrap Automation with Ansible + kubeadm

## Overview

This project automates the provisioning of a Kubernetes cluster from scratch using:

* Ansible for orchestration and configuration management
* kubeadm for Kubernetes cluster bootstrapping
* containerd as the container runtime
* Calico as the CNI plugin
* Kubernetes Dashboard for cluster visibility and administration

The playbook was originally designed and tested on **2 RHEL 10 servers**:

* 1 Control Plane Node
* 1 Worker Node

However, the automation was intentionally written with modularity and portability in mind, making several parts reusable across other Linux distributions with minimal modifications.

Apart from the kubeadm-bootstrap.yaml, I had an inventory file that contains the host names and group names that this playbook uses to execute certain steps. I have shared the example inventory file in this readme file below, it's not exactly the same as the one I used in my environment. So before you can run this playbook, you need to create your own inventory file.
---

# Motivation Behind This Project

I've been working with multiple kubernetes clusters in my profession, faced multiple challenges and had tremendous learnings too. Especially when I got the responsibility to bootstrap a production level kubernetes cluster on a bare metal enterprise server, not once, not twice, but thrice. Yes 3 times!!

It was during this third time that I decided to write this playbook, so that I can use it setup clusters or join new nodes anytime I want, with ease, with reproducible steps that are already documented before anything is executed. Because if anything goes wrong, you can fix it if you know what caused the event and where things went wrong. Saves you the time for RCA.

When I was setting up the cluster, the docs or internet help didn't work out very good for me because my environment was very different from a generic setup, and I had many challenges and scenarios that weren't already covered by someone. For instance, I was working on an internet restricted baremetal, with existing docker workloads that I couldn't risk getting shut down or have docker's containter runtime interfere with my kubernetes bootstrap.

Most Kubernetes tutorials I came across were on:

* Running Kubernetes locally with Minikube or Kind
* Cloud-managed clusters like EKS, GKE, or AKS
* Quick-start scripts that work only in ideal environments

Very few explained how to:

* Bootstrap Kubernetes from bare Linux servers
* Handle OS-level dependencies correctly
* Configure container runtimes properly
* Solve networking prerequisites
* Make installations reproducible and idempotent
* Automate the process in a maintainable way

The outcome of this project for me was not just to "install Kubernetes", but to develop the thinking of a Systems Administrator or a Platform Engineer.

This project reflects the mindset of building:

* Repeatable infrastructure
* Reliable provisioning workflows
* Operationally safe automation
* Declarative infrastructure setup
* Production-aware Kubernetes bootstrapping

---

# Why I Used Ansible Instead of Manual Setup

A manual kubeadm setup may work once.

Ansible makes it:

* Reproducible
* Scalable
* Version controllable
* Idempotent
* Auditable
* Easier to maintain
* Faster to recover or rebuild

## Benefits of Using Ansible Here

### 1. Infrastructure as Code (IaC)

Every configuration change is defined declaratively.

This means:

* No undocumented manual steps
* No configuration drift
* Easy environment recreation
* Easier onboarding for teams

### 2. Idempotency

The playbook was written to avoid unnecessary changes when re-run.

Examples:

* `creates:` checks prevent re-running kubeadm init
* swap removal persists safely in `/etc/fstab`
* service states are declarative
* configuration files are updated only when needed

This is critical in real infrastructure operations.

### 3. Multi-Node Orchestration

Ansible allows coordinated provisioning across nodes.

Examples from this project:

* Configure all nodes consistently
* Initialize control plane separately
* Dynamically retrieve worker join command
* Join worker nodes automatically

### 4. Separation of Concerns

The playbook separates:

* Common node preparation
* Control plane initialization
* Worker node joining

This improves readability, maintainability, and extensibility.

### 5. Operational Reliability

The automation includes:

* Runtime validation
* Service management
* Retry mechanisms
* Kernel tuning
* Network preparation
* Container runtime configuration

These are the details that usually break Kubernetes installations in real environments.

---

# Architecture

## Cluster Topology

```text
+-----------------------+
| Control Plane Node    |
|-----------------------|
| kube-apiserver        |
| kube-controller       |
| kube-scheduler        |
| etcd                  |
| containerd            |
| Calico                |
| Dashboard             |
+-----------------------+
           |
           |
+-----------------------+
| Worker Node           |
|-----------------------|
| kubelet               |
| kube-proxy            |
| containerd            |
| Calico                |
+-----------------------+
```

---

# Features

## Automated Kubernetes Bootstrap

The playbook automates:

* Kubernetes package installation
* kubeadm initialization
* kubelet setup
* kubectl installation
* worker node joining

---

## Container Runtime Configuration

The playbook configures:

* containerd installation
* systemd cgroup driver
* runtime startup and enablement
* CRI compatibility

This is important because Kubernetes runtime compatibility issues are one of the most common causes of cluster instability.

---

## Docker Compatibility Handling

The playbook includes logic to:

* Detect Docker installation
* Validate cgroup driver alignment
* Configure Docker for Kubernetes compatibility
* Prevent containerd conflicts

This demonstrates awareness of:

* CRI conflicts
* cgroup mismatches
* runtime interoperability problems

These are real operational issues frequently encountered in Kubernetes environments.

---

## Kubernetes Networking Preparation

The automation configures:

* `br_netfilter`
* `overlay`
* sysctl networking parameters
* IP forwarding
* bridge network packet processing

These settings are mandatory for Kubernetes networking and CNI plugins.

---

## Calico CNI Deployment

The cluster uses:

* Calico CNI
* Configurable Pod CIDR
* Automated manifest deployment
* Readiness validation

Networking is often one of the hardest parts of Kubernetes setup.

This automation ensures:

* consistent networking configuration
* deterministic pod networking
* automatic readiness verification

---

## Kubernetes Dashboard Deployment

The playbook automatically:

* Deploys Kubernetes Dashboard
* Creates admin ServiceAccount
* Configures RBAC permissions
* Generates authentication token
* Provides secure access instructions

This improves observability and operational usability immediately after bootstrap.

---

# Operational Scenarios Considered While Designing This Playbook

One major goal of this project was to account for real-world operational edge cases.

## 1. Swap Handling

Kubernetes requires swap to be disabled.

The playbook:

* disables swap at runtime
* removes swap persistence from `/etc/fstab`

This ensures reboot-safe configuration.

---

## 2. SELinux Considerations

RHEL-based systems often encounter SELinux-related issues during Kubernetes bootstrap.

The playbook:

* optionally sets SELinux to permissive mode
* makes the behavior configurable

This reflects awareness of:

* OS security enforcement
* kubeadm compatibility concerns
* operational flexibility

---

## 3. Time Synchronization

Cluster components depend heavily on synchronized clocks.

The playbook ensures:

* chronyd installation
* time synchronization service enablement

This prevents:

* TLS certificate issues
* token expiration problems
* distributed system inconsistencies

---

## 4. Container Runtime Stability

The playbook explicitly configures:

```toml
SystemdCgroup = true
```

This is important because mismatched cgroup drivers between:

* kubelet
* Docker
* containerd

can cause:

* kubelet instability
* pod launch failures
* node registration problems

---

## 5. Runtime Conflict Awareness

The automation considers scenarios where:

* Docker already exists
* Docker-installed `containerd.io` conflicts with Kubernetes runtime expectations

The playbook includes defensive checks and compatibility logic.

This reflects practical operational thinking instead of assuming a clean environment.

---

## 6. Idempotent Re-Runs

Real infrastructure automation must tolerate repeated executions.

The playbook includes safeguards such as:

* `creates:` conditions
* conditional runtime actions
* service state validation
* selective file replacement

This allows safer re-execution after:

* partial failures
* node restarts
* interrupted provisioning

---

## 7. Readiness Verification

The automation waits for:

* Calico pods
* Dashboard pods

before proceeding.

This avoids race conditions and incomplete cluster initialization.

---

## 8. Secure Dashboard Access Guidance

Instead of exposing the dashboard insecurely, the playbook recommends:

* localhost-only kubectl proxy
* SSH tunneling

This demonstrates awareness of Kubernetes security best practices.

---

# Technology Stack

| Component  | Purpose                   |
| ---------- | ------------------------- |
| Ansible    | Infrastructure automation |
| Kubernetes | Container orchestration   |
| kubeadm    | Kubernetes bootstrap      |
| containerd | Container runtime         |
| Calico     | CNI networking            |
| RHEL 10    | Operating system          |
| kubelet    | Node agent                |
| kubectl    | Cluster management CLI    |
| chronyd    | Time synchronization      |

---

# Playbook Structure

## Phase 1 — Common Node Preparation

Executed on:

* Control Plane
* Worker Nodes

Tasks include:

* installing dependencies
* configuring kernel modules
* sysctl tuning
* disabling swap
* configuring SELinux
* installing containerd
* installing Kubernetes packages

---

## Phase 2 — Control Plane Initialization

Executed only on:

* Control Plane Node

Tasks include:

* kubeadm init
* kubeconfig setup
* Calico deployment
* Dashboard deployment
* token generation

---

## Phase 3 — Worker Node Join

Executed only on:

* Worker Nodes

Tasks include:

* retrieving join command
* joining cluster
* enabling kubelet

---

# Why This Project Demonstrates DevOps and Sysadmin Thinking

This project is more than a Kubernetes installation script.

It demonstrates understanding of:

## Linux System Administration

* kernel modules
* system services
* SELinux
* networking
* sysctl tuning
* package management
* runtime configuration

---

## Infrastructure Automation

* declarative provisioning
* idempotency
* orchestration
* environment consistency
* operational reproducibility

---

## Kubernetes Internals

* kubeadm lifecycle
* CRI runtime integration
* cgroup management
* networking requirements
* control plane bootstrap
* worker node orchestration

---

## Reliability Engineering

* readiness checks
* retry handling
* conditional execution
* runtime validation
* operational safety

---

## Security Awareness

* controlled dashboard exposure
* RBAC configuration
* service account handling
* SELinux considerations
* runtime isolation awareness

---

# OS Compatibility Analysis

## Was This Built Specifically for RHEL 10?

Yes.

This playbook was specifically developed and tested on:

* RHEL 10

Several implementation details are RHEL-oriented.

Examples:

* usage of `yum`
* RPM repository configuration
* SELinux handling
* RHEL package naming assumptions
* chronyd service expectations

---

# Is It OS Agnostic?

Not fully.

The playbook is currently:

* Kubernetes-focused
* partially portable
* not fully distro-agnostic yet

The Kubernetes concepts are portable.

However, some OS-specific tasks would require adaptation.

---

# Can It Run on Ubuntu or Debian?

Not directly without modifications.

Ubuntu/Debian systems would require changes such as:

## Required Changes

### Package Manager

Replace:

```yaml
yum:
```

with:

```yaml
apt:
```

---

### Kubernetes Repository Configuration

RHEL uses:

```text
/etc/yum.repos.d/
```

Ubuntu requires:

```text
/etc/apt/sources.list.d/
```

---

### Package Names

Some packages differ slightly between distros.

Examples:

* conntrack-tools vs conntrack
* gnupg2 vs gnupg

---

### SELinux

Ubuntu commonly uses:

* AppArmor

instead of SELinux.

SELinux tasks would need conditional handling.

---

### Service and Runtime Differences

Container runtime packaging may vary.

---

# What Would Make This Truly Cross-Platform?

I believe this playbook will be very helpful for RHEL10 users. Although please be aware to first check all steps properly and make any changes according to your environment and constraints. Especially if you're planning to run it in a production system that is already running workloads you do not want to disturb. I plan to make this production-grade and OS-agnostic so that it can be run on any linux distro, for which, the next steps would be:

### 1. Use Ansible Role Structure

Separate:

* runtime setup
* kubeadm setup
* networking
* dashboard deployment

into reusable roles.

---

### 2. Add OS-Specific Task Files

Example:

```yaml
- include_tasks: "{{ ansible_os_family }}.yml"
```

This allows:

* RHEL support
* Ubuntu support
* Debian support
* Rocky Linux support
* AlmaLinux support

---

### 3. Replace Hardcoded Package Logic

Use variables:

```yaml
container_runtime_package:
```

and distro mappings.

---

### 4. Externalize Configuration

Move:

* Kubernetes version
* dashboard version
* pod CIDR
* runtime config

into:

* `group_vars`
* `host_vars`

---

# Inventory

```ini
[k8s_all]
PROD01 ansible_host=<CONTROL_PLANE_IP>
PROD02 ansible_host=<WORKER_IP>

[control_plane]
PROD01

[workers]
PROD02
```

---

# Execution

```bash
ansible-playbook -i inventory.ini k8s-calico-dashboard.yml
```

# Final Thoughts

This project represents practical infrastructure engineering beyond simple Kubernetes installation.

It demonstrates:

* systems thinking
* automation design
* operational awareness
* reliability considerations
* Linux administration knowledge
* Kubernetes platform engineering fundamentals

The focus was not only on getting Kubernetes running, but on understanding:

* why clusters fail
* how Linux impacts Kubernetes
* how runtime compatibility matters
* how reproducible infrastructure should be designed
* how automation should behave safely in real environments

This project reflects the mindset of treating infrastructure as an engineered system rather than a collection of shell commands.