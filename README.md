# Zookeeper Architecture Documentation

## Overview

Zookeeper is a lightweight, rootless container runtime engineered from scratch in C. Designed to emulate the core functionality of modern container engines like Docker or Podman, Zookeeper provides secure, isolated execution environments for arbitrary processes. It achieves this by heavily leveraging low-level Linux kernel features, including Namespaces, Control Groups (cgroups v2), Seccomp, and Capabilities, all while running entirely in user-space without requiring root daemon privileges.

## System Architecture & Core Components

### 1. Process Isolation (Linux Namespaces)

At the heart of Zookeeper’s isolation mechanism is the orchestration of Linux namespaces. When launching a containerized process, the runtime creates an entirely new execution context separated from the host system. This is achieved by utilizing the clone system call to unshare the following namespaces:

* **User Namespace:** Maps the container's root user to an unprivileged user on the host.
* **PID Namespace:** Isolates the process ID number space, ensuring the containerized process runs as PID 1 and cannot view host processes.
* **Mount Namespace:** Provides an isolated filesystem view, allowing the container to have its own root directory (`/`).
* **Network, IPC, UTS, and Cgroup Namespaces:** Isolates network interfaces, inter-process communication, hostnames, and cgroup hierarchies respectively.

### 2. Rootless Execution & ID Mapping

To operate securely without root privileges, Zookeeper implements complex User ID (UID) and Group ID (GID) mapping. It parses the host's `/etc/subuid` and `/etc/subgid` configurations to allocate a block of subordinate IDs. Using external setuid helpers (`newuidmap` and `newgidmap`), it maps these subordinate IDs to the container’s isolated user namespace. This allows the process to act as "root" inside the container while remaining safely unprivileged on the host OS.

### 3. Resource Management (Cgroups v2 & Systemd DBus)

Because standard unprivileged users cannot arbitrarily create cgroups on modern systemd-managed Linux distributions, Zookeeper dynamically interfaces with the host's Systemd process via the **DBus API**.

* **Transient Scopes:** The runtime sends a `StartTransientUnit` request to Systemd to carve out a delegated scope unit (`user.slice`) specifically for the container.
* **Hardware Limitation:** Once the delegated cgroup is provisioned, Zookeeper directly manipulates the `cgroups v2` pseudo-filesystem to enforce strict, user-defined limits on CPU utilization (`cpu.max`), memory footprint (`memory.max`), and process creation limits (`pids.max`).

### 4. Filesystem & Rootfs Management

Zookeeper constructs a dedicated, private filesystem for the payload. The architecture follows a strict sequence to ensure no host filesystem leakage:

* **Private Mounts:** The mount propagation is set to private, preventing container mounts from reflecting back to the host.
* **Bind Mounting:** The user-provided root directory is bind-mounted into a secure temporary location.
* **procfs Isolation:** A fresh instance of `/proc` is mounted with strict flags (`MS_NOSUID`, `MS_NODEV`, `MS_NOEXEC`) so the container tools accurately reflect the isolated PID namespace.
* **Pivot Root:** Finally, the runtime uses the `pivot_root` syscall to securely swap the container's root filesystem, unmounting and detaching the old host root to prevent directory traversal escapes.

### 5. Security & Kernel Sandboxing

To minimize the attack surface of the containerized process, Zookeeper implements dual-layered kernel sandboxing prior to process execution:

* **Linux Capabilities:** Unnecessary kernel privileges are systematically dropped, preventing the container from performing privileged operations even if it acts as root within its namespace.
* **Seccomp-bpf:** A strict Secure Computing (seccomp) filter is applied to the process, intercepting and blocking dangerous or unauthorized system calls at the kernel level.

### 6. User-Mode Networking

Zookeeper supports isolated networking by integrating with `slirp4netns`. The runtime forks a secondary networking process that binds to the container's network namespace, providing outbound network access (via a virtual `eth0` interface) and IPv6 support without requiring host-level bridge configurations or root network privileges.

### 7. Process Synchronization

Because container initialization involves multiple asynchronous steps (cgroup allocation, ID mapping, network attachment), Zookeeper employs an Inter-Process Communication (IPC) pipe to synchronize the host (parent) and the container (child). The child process deliberately blocks until the parent completes the DBus DBus cgroup delegation, ID mapping, and `slirp4netns` setup, ensuring the environment is fully secure and configured before the final `execvp` payload execution.

