---
date: '2025-05-12T22:00:00Z'
title: 'KEP: fUser'
thumbnail: '/img/kep/kep.png'
description: "Kubernetes Enhancement Proposal: securityContext fsUser"
tags: ["Lukáš Hejtmánek", "Dušan Úradník","CERIT-SC", "Kubernetes", "KEP", "Security"]
colormode: true
draft: true
---

## Kubernetes `securityContext`

In Kubernetes, the `securityContext` is widely used to enhance the security of containers running in a Pod. It allows administrators—or even end users deploying the Pod—to define security-related settings that control how containers are executed.

Using the `securityContext`, users can:

- Specify a user ID (`runAsUser`) or group ID (`runAsGroup`) under which the container should run.
- Enforce non-root execution by disabling the ability to run as the root user.
- Control privilege escalation to prevent processes from gaining additional privileges.
- Optionally, enable privileged mode, which grants the container full access to the host system (not recommended unless absolutely necessary).

The `securityContext` can be defined at two levels:

- **Pod-level**: Applies to all containers within the Pod.
- **Container-level**: Overrides specific settings from the Pod-level `securityContext`.

Note that some options are only valid at the Pod level, while others can only be applied at the container level.

One notable Pod-level option is `fsGroup`. When set, the specified group ID is added to the **supplementary groups** of all processes within the container. This means the processes can access files or volumes that require group permissions.

Additionally, when a volume—such as a `ConfigMap`, `Secret`, or a CSI volume (e.g., NFS)—is mounted into the container, Kubernetes assigns the `fsGroup` ID to all files within that volume (in case of CSI, it is optional). This ensures that container processes have the necessary read/write access to the volume contents.


### Limitations of the `securityContext`

Unfortunately, there is no setting analogous to `fsGroup`—such as a hypothetical `fsUser`—that allows specifying the **file owner** (user ID) for mounted volumes.

#### Why does this matter?

Some applications, such as `ssh` and `psql` (the PostgreSQL command-line client), enforce strict file ownership and permission rules. For example:

- `ssh` requires the `authorized_keys` file to be owned by the correct user and not readable by group or others.
- `psql` requires `.pgpass` to be owned by the user and similarly restricts group or world-readable permissions.

Files mounted from a `Secret` (or `ConfigMap`, though not recommended for sensitive data) are always owned by `root`. Even when `fsGroup` is used to grant group access, it is often insufficient, as these applications explicitly check for user ownership and strict permission settings.

#### Workarounds

To address this, many users resort to:

- **`initContainers`**: These containers run before the main application and can copy files from the mounted location to another location with the correct ownership and permissions.
- **Custom entrypoint scripts**: Scripts within the container can perform similar copy-and-fix operations during startup.

While these workarounds functionally solve the problem, they introduce complexity and potential security risks.

#### A Better Solution

A feature like `fsUser` would allow mounted files to be assigned the correct ownership automatically, simplifying deployments and improving security by eliminating the need for post-mount fixes.

## Kubernetes Enhancement Proposal (KEP)

A **Kubernetes Enhancement Proposal (KEP)** is the standard mechanism for proposing, communicating, and coordinating new initiatives within the Kubernetes project. Essentially, if a new feature or functionality is needed, it should be introduced through a KEP.

A KEP is a formal proposal that outlines the motivation behind the feature, defines the scope through goals and non-goals (i.e., what the proposal will and will not cover), and provides detailed user stories that explore the problem space. It also includes design details to help stakeholders understand the implementation approach.

A key aspect of introducing new features in Kubernetes is **feature staging**. This process involves gradually rolling out the feature through three main phases:

- **Alpha**: The feature is disabled by default and protected behind a *feature gate*.
- **Beta**: The feature is enabled by default but still behind a feature gate.
- **GA (General Availability)**: The feature is considered stable, and the feature gate is removed.

The KEP process typically begins with a virtual meeting with the relevant **Special Interest Group (SIG)**, such as `sig-node`, `sig-storage`, or `sig-auth`. The idea is presented informally to gather early feedback. If the SIG is supportive, the KEP is formalized and submitted as a GitHub Pull Request (PR) to the [KEP repository](https://github.com/kubernetes/enhancements).

Once submitted, the PR is subject to review by the community. Feedback and suggestions are expected, and the proposal must be revised accordingly before it can be accepted.


