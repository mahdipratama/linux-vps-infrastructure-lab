# Linux VPS Infrastructure Lab

## Initial Server Assessment

**Environment:** Awan Data VPS  
**Operating system:** Ubuntu 24.04 LTS (Noble Numbat)

## 1. Purpose and scope

This document records the initial, read-only assessment of the VPS before making configuration changes. The goal is to establish a baseline for subsequent Linux administration, networking, security, containerization, and application-deployment work.

No packages were installed or removed and no firewall, network, or service configuration was changed during this assessment.

## 2. Executive summary

| Area                     | Finding                                                                             |
| ------------------------ | ----------------------------------------------------------------------------------- |
| Operating system         | Ubuntu 24.04 LTS, codename `noble`                                                  |
| Kernel                   | `6.8.0-31-generic`                                                                  |
| Architecture             | `x86_64`                                                                            |
| Virtual CPU              | 4 logical CPUs visible to the guest OS                                              |
| Virtualization           | KVM; full virtualization reported                                                   |
| NUMA                     | 1 NUMA node visible to the guest OS                                                 |
| Memory                   | 7.8 GiB total; 7.2 GiB available at inspection                                      |
| Swap                     | No active swap configured/reported                                                  |
| Disk                     | 40 GB virtual disk; root filesystem is ext4                                         |
| Root filesystem usage    | 2.4 GB used; 35 GB available; 7% used                                               |
| Docker/container runtime | Docker CLI and standard Docker/containerd/Podman packages or processes not detected |
| Listening sockets        | One TCP listening socket detected: port 22 (`sshd`)                                 |
| Running services         | 19 services reported running by systemd                                             |
| Host firewall            | No filtering rules observed in the inspected nftables/iptables rulesets             |
| Provider firewall        | Not verified; no firewall-management menu was found in the customer panel           |

## 3. Operating system and kernel

Command used:

```bash
cat /etc/os-release
uname -r
uname -m
```

Observed results:

- Distribution: Ubuntu
- Release: 24.04 LTS
- Codename: Noble Numbat (`noble`)
- Kernel: `6.8.0-31-generic`
- Machine architecture: `x86_64`

Ubuntu belongs to the Debian family. The environment therefore uses the Debian package ecosystem, including `apt` and `.deb` packages.

## 4. CPU and virtualization

Command used:

```bash
lscpu
```

Observed results:

| Property                      | Observed value     |
| ----------------------------- | ------------------ |
| Architecture                  | `x86_64`           |
| Logical CPUs visible to guest | 4 (`0–3`)          |
| Model name reported           | AMD EPYC Processor |
| Vendor ID reported            | `GenuineIntel`     |
| Hypervisor vendor             | KVM                |
| Virtualization type           | Full               |
| Threads per core              | 1                  |
| Cores per socket              | 1                  |
| Sockets presented to guest    | 4                  |
| NUMA nodes                    | 1                  |

**Assessment note:** The CPU Vendor ID and model-name fields appear inconsistent. In a virtual machine, CPU identity and topology are presented by the hypervisor and do not necessarily identify the physical host CPU. This observation is recorded without inferring the provider's physical hardware configuration.

The guest OS sees four logical CPUs in one NUMA node. No NUMA tuning is indicated for this introductory lab.

## 5. Memory

Command used:

```bash
free -h
```

Observed results:

| Metric       |   Value |
| ------------ | ------: |
| Total memory | 7.8 GiB |
| Used         | 585 MiB |
| Free         | 7.0 GiB |
| Shared       | 2.5 MiB |
| Buff/cache   | 431 MiB |
| Available    | 7.2 GiB |
| Swap total   |     0 B |

`available` is the more useful quick indicator of memory that can be made available to applications than `free` alone, because Linux uses otherwise-unused memory for caches.

No swap space was active or reported during the inspection. Whether to configure swap will be considered later based on workload and operational requirements; no change was made during assessment.

## 6. Storage and filesystems

Commands used:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
df -hT
```

Observed block-device layout:

```text
NAME   SIZE TYPE FSTYPE MOUNTPOINTS
vda     40G disk
└─vda1  40G part ext4   /
```

Root filesystem usage:

| Filesystem  | Type | Size | Used | Available | Use | Mount point |
| ----------- | ---- | ---: | ---: | --------: | --: | ----------- |
| `/dev/vda1` | ext4 |  40G | 2.4G |       35G |  7% | `/`         |

The guest OS sees one 40 GB virtual disk with one ext4 partition mounted as the root filesystem. The `df` output also showed temporary `tmpfs` mounts at `/run`, `/dev/shm`, `/run/lock`, and `/run/user/0`. These are memory-backed temporary filesystems, not additional persistent virtual disks.

The small difference between filesystem size, used space, and available space can reflect filesystem metadata, reserved blocks, and display rounding. No filesystem changes were made.

## 7. Docker and container-runtime assessment

Checks performed:

```bash
command -v docker
docker -v
dpkg -l | grep -Ei 'docker|containerd|podman'
systemctl list-unit-files --type=service | grep -Ei 'docker|containerd|podman'
ps -eo pid,comm,args | grep -E '[d]ockerd|[c]ontainerd|[p]odman'
```

Findings:

- Docker executable was not found in `PATH`.
- `docker -v` reported that the command was not found.
- No matching Docker, containerd, or Podman packages were listed by the package query.
- No matching systemd service unit files were found by the query.
- No matching `dockerd`, `containerd`, or `podman` processes were found by the process query.

**Assessment:** No standard Docker or container-runtime installation was detected by these checks. This is not a guarantee that no custom binary or non-standard installation exists in an uninspected location. Docker will be installed in the dedicated containerization phase, after its concepts and installation approach are covered.

## 8. Listening sockets and running services

Commands used:

```bash
ss -tulpn
systemctl list-units --type=service --state=running
```

The listening-socket check reported one entry:

| Protocol | Local address | Port | Process |
| -------- | ------------- | ---: | ------- |
| TCP      | `*`           |   22 | `sshd`  |

The wildcard local address indicates that the socket is not limited to the loopback address. It does **not**, by itself, prove that the port is reachable from the public internet; network/provider filtering may also affect reachability.

Systemd reported 19 running services:

- `cron.service`
- `dbus.service`
- `fwupd.service`
- `getty@tty1.service`
- `ModemManager.service`
- `multipathd.service`
- `polkit.service`
- `qemu-guest-agent.service`
- `rsyslog.service`
- `ssh.service`
- `systemd-journald.service`
- `systemd-logind.service`
- `systemd-networkd.service`
- `systemd-timesyncd.service`
- `systemd-udevd.service`
- `udisks2.service`
- `unattended-upgrades.service`
- `upower.service`
- `user@0.service`

A running service does not necessarily open a network listening port. The service list and listening-socket list answer different questions.

## 9. Host firewall assessment

Checks performed:

```bash
ufw status verbose
dpkg -l | grep -Ei 'ufw|nftables|iptables'
nft list ruleset
iptables -V
iptables -S
ip6tables -S
```

Findings:

- The `ufw` command was not found.
- Package status showed `ufw` as `rc` (removed, with residual configuration files recorded by `dpkg`).
- `iptables` was installed; version output was `iptables v1.8.10 (nf_tables)`.
- `nftables` and its related userspace libraries were installed.
- `nft list ruleset` returned no ruleset output.
- IPv4 `iptables -S` showed only these default policies:

  ```text
  -P INPUT ACCEPT
  -P FORWARD ACCEPT
  -P OUTPUT ACCEPT
  ```

- IPv6 `ip6tables -S` showed the same default policies.
- No additional `-A` filtering rules were shown by the IPv4 or IPv6 queries.

**Assessment:** No host-level packet-filtering rules were observed in the nftables and iptables checks performed. The default policies shown are `ACCEPT`.
This should be treated as a security finding: do not expose additional application or database services until an intentional firewall policy has been designed and validated.

UFW was not installed during assessment. It will be considered in the dedicated networking and security phase, after confirming SSH access and a recovery path through the provider console.

## 10. Provider-level firewall

The customer panel was inspected, but no menu for Firewall, Network Security, Security Groups, or equivalent network-rule management was found. Awan Data support subsequently confirmed in its reply that a Provider-Level Firewall or Network Firewall that customers can manage directly through the control panel is currently not available for this VPS service. Support stated that firewall configuration is generally performed inside the VPS itself, according to the customer's needs.

Support also said that the request for more detailed information about firewall configuration and additional provider security services would be forwarded to its technical team.

Status: Customer-managed provider firewall through the panel is not available, according to the support reply. This does not establish that Awan Data has no network-level filtering or protective controls anywhere in its underlying infrastructure. Whether provider-side filtering exists, is enabled by default, or is offered as an additional service remains unverified pending the technical team's response.

## 11. Risks, constraints, and follow-up actions

| Finding                                               | Implication                                                                                                                           | Follow-up                                                                                                        |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| No host filtering rules observed                      | Inbound traffic may not be restricted at the guest OS firewall layer                                                                  | Design and validate a host firewall policy in the security phase                                                 |
| No customer-manageable provider firewall in the panel | Customers must configure the guest OS firewall for their own VPS access policy; underlying provider-side filtering remains unverified | Await the provider technical team's clarification about network-level filtering and additional security services |
| SSH TCP/22 is listening                               | Remote administration depends on SSH access                                                                                           | Review SSH configuration and preserve console recovery before firewall changes                                   |
| No active swap reported                               | Memory-pressure behavior has no configured swap space to fall back on                                                                 | Revisit based on workload and resource monitoring                                                                |
| 40 GB disk                                            | Capacity is finite and shared by OS, images, containers, databases, and logs                                                          | Monitor disk consumption during deployments                                                                      |
| CPU identity fields inconsistent                      | Guest-reported CPU identity may not describe physical host hardware                                                                   | Treat guest-visible vCPU allocation as the reliable assessment input                                             |

## 12. Baseline conclusion

The VPS currently presents Ubuntu 24.04 LTS on a KVM virtual machine with four logical CPUs, 7.8 GiB RAM, and a 40 GB ext4 root filesystem. No standard Docker/container-runtime installation was detected. The only listening socket observed was SSH on TCP port 22, while systemd reported 19 running services.

The most important item to address before public application deployment is network security: no host-level filtering rules were observed. Awan Data support confirmed that customers currently cannot manage a Provider-Level/Network Firewall through the control panel; the existence and scope of any underlying provider-side filtering remain unverified pending technical-team clarification. No changes were made during this initial assessment.

## 13. Next learning phase

**Phase 2 — Linux Server Administration**

The next phase will focus on Linux users and groups, filesystem navigation, permissions, package management, processes, systemd, logs, and basic Bash administration. Security-sensitive changes will be postponed until their impact and recovery process are understood.

---

## Appendix: Commands used

```bash
# OS and kernel
cat /etc/os-release
uname -r
uname -m

# CPU, memory, storage
lscpu
free -h
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
df -hT

# Docker/container runtime
command -v docker
docker -v
dpkg -l | grep -Ei 'docker|containerd|podman'
systemctl list-unit-files --type=service | grep -Ei 'docker|containerd|podman'
ps -eo pid,comm,args | grep -E '[d]ockerd|[c]ontainerd|[p]odman'

# Networking and services
ss -tulpn
systemctl list-units --type=service --state=running

# Firewall
ufw status verbose
dpkg -l | grep -Ei 'ufw|nftables|iptables'
nft list ruleset
iptables -V
iptables -S
ip6tables -S
```
