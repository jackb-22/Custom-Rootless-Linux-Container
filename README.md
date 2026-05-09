# Zookeeper Architecture Documentation

*Note: The full C source code for this project is kept private in accordance with Columbia University academic integrity policies. This repository provides an architectural overview of the systems design, Linux kernel mechanisms, and security model implemented by the runtime.*

## Overview

Zookeeper is a lightweight, rootless container runtime written from scratch in C. It is designed as an educational systems project that recreates many of the core isolation mechanisms used by modern container engines such as Docker and Podman.

The runtime launches arbitrary programs inside isolated execution environments without requiring a privileged daemon. It relies on Linux namespaces, cgroups v2, user namespaces, seccomp-bpf, Linux capabilities, `pivot_root`, and user-mode networking to build a container-like environment entirely from user space.

At a high level, Zookeeper provides:

- Process isolation through Linux namespaces
- Rootless execution through UID/GID mapping
- Filesystem isolation through bind mounts and `pivot_root`
- Resource limits through cgroups v2
- Syscall filtering through seccomp-bpf
- Privilege reduction through Linux capabilities
- Optional user-mode networking through `slirp4netns`
- Parent/child synchronization during container setup

Zookeeper is not intended to be a production container runtime. Instead, it demonstrates how container primitives work internally by implementing the core mechanics directly against Linux kernel APIs.

---

## Container Startup Lifecycle

Zookeeper follows a carefully ordered startup sequence to ensure the container environment is fully configured before the target process begins execution.

1. Parse command-line options, including:
   - Root filesystem path
   - Networking mode
   - Resource limits
   - Program and arguments to execute

2. Create a synchronization pipe between the parent runtime process and the child container process.

3. Clone the child process with the requested Linux namespaces.

4. Configure the child process's UID/GID mappings using subordinate ID ranges from `/etc/subuid` and `/etc/subgid`.

5. Use `newuidmap` and `newgidmap` to safely map host subordinate IDs into the container's user namespace.

6. Request a delegated cgroup scope from systemd over DBus.

7. Apply cgroups v2 resource limits such as:
   - `cpu.max`
   - `memory.max`
   - `pids.max`

8. If networking is enabled, attach user-mode networking through `slirp4netns`.

9. Signal the child process that host-side setup is complete.

10. In the child process, configure the container filesystem:
    - Make mount propagation private
    - Bind mount the root filesystem
    - Mount a fresh `/proc`
    - Call `pivot_root`
    - Detach the old host root

11. Drop unnecessary Linux capabilities.

12. Install the seccomp-bpf syscall filter.

13. Execute the target payload using `execvp`.

This ordering is important because several setup steps must occur from the parent process before the child can safely enter its final isolated environment.

---

## System Architecture and Core Components

## 1. Process Isolation with Linux Namespaces

The foundation of Zookeeper's isolation model is Linux namespaces. During container creation, Zookeeper uses the `clone()` system call to create a child process inside a separate execution context.

The runtime can isolate the following namespaces:

### User Namespace

The user namespace allows the container process to believe it is running as root while actually mapping to an unprivileged user on the host.

Inside the container, the process may have UID 0. On the host, however, that UID maps to an unprivileged subordinate ID range. This is the key mechanism that allows Zookeeper to operate without a root daemon.

### PID Namespace

The PID namespace gives the container its own process ID space.

The first process inside the container becomes PID 1 from the container's point of view. This prevents processes inside the container from viewing or interacting with unrelated host processes.

### Mount Namespace

The mount namespace gives the container its own filesystem view.

This allows Zookeeper to construct a private root filesystem, mount a new `/proc`, and prevent container mount operations from affecting the host.

### Network Namespace

When networking is isolated, the container receives its own network namespace. This separates the container's network interfaces, routing tables, and network stack from the host.

When networking is enabled, Zookeeper integrates this namespace with `slirp4netns` to provide rootless outbound network access.

### IPC Namespace

The IPC namespace isolates System V IPC and POSIX message queues, preventing the container from sharing IPC resources with the host.

### UTS Namespace

The UTS namespace isolates hostname and domain name information. This allows the container to have its own hostname independent of the host system.

### Cgroup Namespace

The cgroup namespace isolates the process's view of the cgroup hierarchy, helping prevent the container from observing unrelated host cgroup details.

---

## 2. Rootless Execution and ID Mapping

Zookeeper is designed to run without root privileges. To accomplish this, it uses Linux user namespaces combined with subordinate UID and GID mappings.

The runtime parses the host's subordinate ID configuration files:

- `/etc/subuid`
- `/etc/subgid`

These files define ranges of UIDs and GIDs that an unprivileged user is allowed to map into a user namespace.

After creating the child process, Zookeeper configures its ID mappings using:

- `newuidmap`
- `newgidmap`

These setuid helpers safely write the child process's UID and GID maps.

The result is that the process can appear to be root inside the container while still being mapped to an unprivileged identity on the host.

For example:

```text
Container UID 0  ->  Host subordinate UID
Container GID 0  ->  Host subordinate GID
````

This allows privileged-looking operations inside the container while preserving host-level safety boundaries.

---

## 3. Resource Management with Cgroups v2 and Systemd DBus

Zookeeper uses cgroups v2 to limit and account for container resource usage.

On modern Linux systems managed by systemd, unprivileged users usually cannot freely create and delegate cgroups directly. To handle this, Zookeeper communicates with systemd over DBus to request a delegated transient scope.

### Transient Scope Creation

The runtime sends a `StartTransientUnit` request to systemd. This creates a temporary scope unit under the user's systemd hierarchy.

This scope is used as the container's delegated cgroup.

### Cgroup-Enforced Limits

Once systemd creates and delegates the scope, Zookeeper writes directly to the cgroups v2 pseudo-filesystem to enforce limits.

The runtime can configure:

### CPU Limits

CPU usage is controlled through:

```text
cpu.max
```

This limits how much CPU time the container can consume within a given scheduling period.

### Memory Limits

Memory usage is controlled through:

```text
memory.max
```

This places an upper bound on the amount of memory the container can allocate.

### Process Limits

Process creation is controlled through:

```text
pids.max
```

This prevents fork bombs or accidental runaway process creation inside the container.

Together, these controls allow Zookeeper to restrict the container's CPU, memory, and process usage without requiring root privileges.

---

## 4. Filesystem and Rootfs Management

Zookeeper constructs a private filesystem view for the container payload.

The filesystem setup is carefully ordered to prevent host filesystem leakage and to ensure the process sees the user-provided root directory as `/`.

### Private Mount Propagation

Before modifying mounts, Zookeeper marks mount propagation as private.

This prevents mount events inside the container from propagating back to the host.

Conceptually:

```text
Host mounts  X  Container mounts
```

Container mount changes remain local to the container's mount namespace.

### Root Filesystem Bind Mount

The user-provided root directory is bind-mounted into a controlled temporary location.

This prepares the root filesystem so it can become the container's `/`.

### procfs Isolation

Zookeeper mounts a fresh instance of `procfs` inside the container.

The new `/proc` is mounted with restrictive flags such as:

```text
MS_NOSUID
MS_NODEV
MS_NOEXEC
```

This ensures that tools inside the container observe the container's isolated PID namespace rather than the host's process tree.

### pivot_root

After the root filesystem is prepared, Zookeeper uses the `pivot_root` syscall to replace the process's root directory.

The old host root is moved to a temporary location inside the new root and then detached.

This prevents the process from escaping back into the original host filesystem through path traversal or inherited directory references.

The final result is:

```text
Container /  ->  User-provided root filesystem
Old host /   ->  Detached and inaccessible
```

---

## 5. Security and Kernel Sandboxing

Zookeeper applies multiple defense-in-depth mechanisms before executing the target process.

## Linux Capabilities

Linux capabilities split the traditional power of root into smaller privileges.

Even if the container process appears to be root inside its user namespace, Zookeeper drops unnecessary capabilities before running the payload.

This reduces the damage a compromised or misbehaving process can cause.

Examples of capabilities that may be restricted include privileges related to:

* Raw I/O
* Kernel module loading
* System time modification
* Host-level administration
* Dangerous filesystem operations

The goal is to give the container only the privileges it needs and remove everything else.

## Seccomp-bpf

Zookeeper also installs a seccomp-bpf filter before executing the payload.

Seccomp allows the runtime to restrict which syscalls the containerized process may invoke.

Dangerous or unnecessary syscalls can be blocked at the kernel boundary, reducing the available attack surface.

Examples of syscall categories that may be restricted include:

* User namespace creation from inside the container
* Kernel keyring operations
* Certain debugging or tracing operations
* Performance monitoring
* Low-level memory policy manipulation
* Terminal injection operations

This provides a second layer of protection beyond namespaces and capabilities.

---

## 6. User-Mode Networking with slirp4netns

Zookeeper supports optional rootless networking through `slirp4netns`.

Traditional container networking often requires root privileges to create bridges, configure virtual Ethernet devices, or manipulate host network settings. Zookeeper avoids this by using user-mode networking.

When networking is enabled:

1. The container is created with its own network namespace.
2. Zookeeper starts a helper process using `slirp4netns`.
3. `slirp4netns` attaches to the container's network namespace.
4. The container receives outbound network connectivity through a virtual network interface.

This allows the container to access external networks without requiring privileged host networking configuration.

The networking implementation is designed for rootless execution. IPv6 support may be experimental depending on the host and `slirp4netns` configuration.

---

## 7. Parent/Child Synchronization

Container setup requires coordination between the parent runtime process and the child container process.

Some setup steps must happen from the parent after the child exists but before the child executes the final payload.

Examples include:

* Writing UID/GID mappings
* Creating or joining the delegated cgroup
* Configuring resource limits
* Attaching user-mode networking
* Completing namespace setup

To coordinate this, Zookeeper uses an IPC pipe.

The child process blocks early in initialization and waits for the parent to finish host-side setup. Once the parent completes the required configuration, it writes to the pipe and releases the child.

This prevents the payload from executing before the container environment is fully isolated and configured.

The synchronization flow is:

```text
Parent process                    Child process
--------------                    -------------
create pipe
clone child  ------------------>  start in namespaces
configure UID/GID maps            wait on pipe
create cgroup scope
apply cgroup limits
start networking
signal child  ----------------->  continue setup
                                  mount filesystem
                                  drop capabilities
                                  install seccomp
                                  exec payload
```

---

## 8. Command-Line Configuration

Zookeeper exposes runtime behavior through command-line flags.

The runtime supports configuration for:

* Root filesystem path
* Networking mode
* CPU limits
* Memory limits
* Process limits
* Program to execute inside the container

A typical invocation resembles:

```bash
./zookeeper -P -n -r ~/zoo-fs /bin/bash
```

This launches `/bin/bash` inside the provided root filesystem with the selected isolation and runtime options.

The exact available flags depend on the build and assignment configuration.

---

## 9. Design Goals

Zookeeper was designed around the following goals:

### Rootless Operation

The runtime should not require a privileged daemon or direct root execution.

User namespaces, subordinate ID mappings, and systemd cgroup delegation allow most container setup to happen safely from an unprivileged user account.

### Minimal Dependencies

Most functionality is implemented directly in C using Linux system calls and kernel interfaces.

External helpers are used only where they are standard parts of rootless container setup, such as:

* `newuidmap`
* `newgidmap`
* `slirp4netns`
* systemd DBus

### Explicit Isolation

The runtime makes isolation boundaries explicit instead of hiding them behind a high-level container API.

Each major container primitive is configured directly:

* Namespace creation
* UID/GID mapping
* Mount setup
* Cgroup configuration
* Capability dropping
* Seccomp filtering

### Educational Transparency

The project is intended to show how containers actually work under the hood.

Rather than implementing image pulling, OCI compatibility, or layered filesystems, Zookeeper focuses on the core Linux primitives that make containers possible.

---

## 10. Limitations

Zookeeper is an educational container runtime and does not attempt to fully replace Docker, Podman, runc, or other production-grade runtimes.

Current limitations include:

* No OCI image support
* No image pulling or registry integration
* No layered filesystem support
* No overlayfs image management
* No container checkpointing or restore
* No advanced network policy system
* No production-grade seccomp profile management
* No full init system inside the container
* Limited lifecycle management after process launch
* Experimental networking behavior depending on host configuration

These limitations are intentional. The project focuses on implementing and understanding the core kernel mechanisms behind containers rather than building a complete container platform.

---

## 11. Key Linux Concepts Demonstrated

Zookeeper demonstrates practical use of several Linux systems programming concepts:

* `clone()`
* Linux namespaces
* User namespaces
* PID namespaces
* Mount namespaces
* Network namespaces
* IPC namespaces
* UTS namespaces
* Cgroup namespaces
* UID/GID mapping
* `/etc/subuid`
* `/etc/subgid`
* `newuidmap`
* `newgidmap`
* cgroups v2
* systemd DBus APIs
* bind mounts
* mount propagation
* `procfs`
* `pivot_root`
* Linux capabilities
* seccomp-bpf
* `execvp`
* IPC pipes
* rootless networking
* `slirp4netns`

---

## Summary

Zookeeper is a rootless container runtime implemented from scratch in C. It builds isolated process environments using the same Linux primitives that power modern container engines, including namespaces, cgroups, capabilities, seccomp, and root filesystem pivoting.

The project emphasizes low-level systems programming and container internals. It demonstrates how a runtime can create a secure, resource-limited, and isolated execution environment without relying on a privileged daemon or high-level container framework.
