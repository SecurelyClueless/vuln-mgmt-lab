# Troubleshooting Log

Issues encountered while building and operating the lab, how each was diagnosed, and how it was fixed.

---

## 1. Router config had a gateway on every interface

**Symptom**
Spotted during config review, before it caused a failure.

**Diagnosis**
`/etc/network/interfaces` had a `gateway` line on all four interfaces; the three inner interfaces pointed at their own addresses.

**Root cause**
A `gateway` line sets the default route. A router needs exactly one, pointing upstream. Inner interfaces get a connected route from their address alone.

**Fix**
Removed `gateway` from ens37, ens38 and ens39; kept `gateway 192.168.1.2` on ens33 only.

**Lesson**
The `.1` addresses on inner segments are gateways for the hosts in those segments, not for the router.

---

## 2. Router WAN had two IP addresses and two default routes

**Symptom**
`ip a` showed `192.168.1.10` (static) and `192.168.1.139` (dynamic) on ens33; `ip route` showed two default routes.

**Diagnosis**
The extra address was flagged `dynamic`, `proto dhcp`, `noprefixroute`, `metric 101` — the NetworkManager signature. `systemctl is-active NetworkManager` confirmed it.

**Root cause**
NetworkManager was managing ens33 alongside ifupdown and requesting its own DHCP lease.

**Fix**
Disabled and masked NetworkManager, flushed ens33, restarted networking, rebooted to confirm, set DNS in `/etc/resolv.conf`.

**Lesson**
One network manager per machine. Two tools on the same interface create conflicting state.

---

## 3. Router WAN static IP inside the VMware DHCP range

**Symptom**
Spotted during review.

**Root cause**
`192.168.1.139` sat inside VMware NAT's DHCP pool, risking a future IP conflict.

**Fix**
Moved the WAN to `192.168.1.10`, outside the pool.

**Lesson**
Static addresses belong outside DHCP ranges; firewall rules and the admin route depend on the router's address staying fixed.

---

## 4. Router could not ping the Windows admin host

**Symptom**
Router → 192.168.1.1 failed; Windows → 192.168.1.10 worked.

**Root cause**
Windows Firewall blocks inbound ICMP echo on the Public profile, which VMware adapters fall under.

**Fix**
Not required (admin traffic originates from Windows). Optional scoped rule allowing ICMP from 192.168.1.10 only.

**Lesson**
Testing both directions isolates which side is blocking.

---

## 5. Windows admin host could not reach vm-scan

**Symptom**
`ping 10.10.30.10` from Windows failed.

**Diagnosis**
_Fill in._

**Root cause**
_Fill in._

**Fix**
_Fill in._

---

## 6. DMZ host could not reach its gateway (asymmetric path)

**Symptom**
dmz-host configured correctly (10.10.20.10/24, gateway 10.10.20.1) but could not ping the router. dmz-host's ARP table showed the gateway at a MAC belonging to a different router interface.

**Diagnosis**
`tcpdump -eni any arp or icmp` on the router showed echo requests arriving on ens37 (configured as Internal) while the router sent ARP requests for 10.10.20.10 out of ens38 (configured as DMZ) with no reply. Comparing `ip -br link` MACs with VMware adapter MACs showed ens37's adapter was attached to the DMZ network and ens38's to Internal LAN.

**Root cause**
VMware adapter-to-network mapping did not match the Linux interface configuration. Interface names come from the virtual PCI slot; MACs come from VMware settings; the link between them changed while adapters were being edited. Linux answering ARP for any local IP on any interface ("ARP flux") hid the problem at layer 2.

**Fix**
Swapped the VMware networks on the two router adapters by MAC. No config files changed. Current mapping documented in `network/README.md`.

**Lesson**
Troubleshoot bottom-up. Verify layer 2 (which switch, which MAC) before blaming routing or the firewall. Configs show intent; packet captures show reality.

---

## 7. rp_filter not applied

**Symptom**
Spoofed-looking traffic (DMZ source arriving on the Internal interface) was processed rather than dropped.

**Root cause**
`sysctl` is in `/usr/sbin`, outside a normal user's PATH on Debian. Running `sysctl --system` without sudo failed with "command not found", so the settings were never applied.

**Fix**
`sudo sysctl --system`; confirmed `net.ipv4.conf.all.rp_filter = 1`.

**Lesson**
Always confirm a security setting's effective value, not just the config file.

---

## 8. dmz-host received no IP address (cloud-init override)

**Symptom**
Static netplan config written, but ens33 had no address. NetworkManager log showed a DHCP transaction starting on ens33.

**Diagnosis**
`sudo netplan get` showed `dhcp4: true` in the merged config. `ls /etc/netplan` showed `50-cloud-init.yaml` alongside the manual file.

**Root cause**
Netplan merges files alphabetically; the installer-generated `50-cloud-init.yaml` overrode `01-network-manager-all.yaml`. The DMZ has no DHCP server, so the interface never came up.

**Fix**
Moved `50-cloud-init.yaml` to a backup folder; disabled cloud-init network management (`/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg`); set `dhcp4: false`; permissions 600.

**Lesson**
Check the merged, effective configuration (`netplan get`) rather than the file you edited.

---

## 9. NetworkManager not running on dmz-host

**Symptom**
`nmcli` reported "NetworkManager is not running"; netplan profile existed but was not active.

**Root cause**
NetworkManager was disabled on dmz-host, but its netplan file named NetworkManager as the renderer.

**Fix**
`systemctl unmask` and `enable --now NetworkManager`.

**Lesson**
"One manager per machine" means the right manager for that machine: ifupdown on the Debian router, NetworkManager via netplan on the Ubuntu desktop host.

---

## 10. Docker socket permission denied

**Symptom**
`docker compose pull` failed with "permission denied while trying to connect to the Docker daemon socket".

**Root cause**
User added to the `docker` group, but group membership only applies after a new login.

**Fix**
Logged out and back in (or `newgrp docker`).

**Lesson**
Membership of the `docker` group is effectively root access — acceptable in a lab, a finding on a real server.

---

## 11. Containers did not restart after reboot

**Symptom**
Tomcat missing from a scan; container showed "created 17 hours ago, up 25 seconds".

**Root cause**
Compose services had no restart policy, so containers stayed stopped after dmz-host rebooted. The scan silently missed the service.

**Fix**
Added `restart: unless-stopped` to every compose service.

**Lesson**
Confirm every expected service is running before scanning, or results will be incomplete without any error.

---

## 12. Nessus missed Apache httpd 2.4.49 (scanner false negative)

**Symptom**
No CVE findings for Apache 2.4.49; only an Info "Apache HTTP Server Version" result.

**Root cause**
The scan policy's "avoid potential false alarms" setting suppresses banner-only version checks on Linux, where distributions backport fixes without changing version numbers.

**Fix**
Validated manually: an unauthenticated path-traversal request returned the container's `/etc/passwd`, confirming CVE-2021-41773. Tracked as F-003 and remediated.

**Lesson**
Scanner output is evidence, not truth. Review exposed services and validate high-risk items manually.

---

## 13. Nessus did not identify the JDWP debug port

**Symptom**
Port 5005 open on dmz-host but no finding raised.

**Root cause**
Nessus reported the service as "unknown"; no plugin matched.

**Fix**
Identified by comparing open ports (host KB) against expected services; tracked as F-004 and remediated.

**Lesson**
Reviewing open ports against an expected-services baseline catches what plugins miss.

---

## 14. In-container fix failed on bind-mounted Tomcat config

**Symptom**
`rm -rf` of Tomcat's manager app failed with "Device or resource busy"; folders remained.

**Root cause**
The vulhub compose file bind-mounted `context.xml` from the host into the manager app. A mount point cannot be deleted from inside the container.

**Fix**
Removed the `volumes` section from docker-compose and added a start-up `command` that deletes the default apps on every container start (CHG-002).

**Lesson**
Fixes must be durable. Changes made inside a running container are lost when it is recreated; put remediation in the build or compose definition.

---

## 15. Router had a full GNOME desktop installed

**Symptom**
Authenticated scan listed gnome-shell and Apache components on the "headless" router.

**Root cause**
The Debian installer's desktop task (`task-gnome-desktop`) was selected at install time; Apache components were pulled in as desktop dependencies.

**Fix**
Purged the desktop task, gdm3, apache2-bin and libapache2-mod-dnssd; ran `apt autoremove --purge` (CHG-003).

**Lesson**
Unnecessary software is attack surface. Infrastructure devices should be built from a minimal image.

---

## Template

```
## N. Short title

**Symptom**
**Diagnosis**
**Root cause**
**Fix**
**Lesson**
```
