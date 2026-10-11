# Troubleshooting Log

Issues encountered while building the lab, how each was diagnosed, and how it was fixed.

---

## 1. Router had multiple default gateways

**Symptom**
Interfaces came up with errors, and routing behaviour was unreliable.

**Diagnosis**
`/etc/network/interfaces` had a `gateway` line on every interface. The three inner interfaces each pointed to their own address (e.g. `gateway 10.10.10.1` on the interface that owns 10.10.10.1).

**Root cause**
A `gateway` line sets the default route. A router should have exactly one default route, pointing upstream. The inner interfaces don't need one: an address on a /24 already gives the router a connected route to that subnet.

**Fix**
Removed the `gateway` lines from ens37, ens38 and ens39. Kept only `gateway 192.168.1.2` on the WAN interface (ens33).

**Lesson**
The `.1` addresses on the inner segments are the gateways _for hosts in those segments_, not for the router itself.

---

## 2. WAN interface had two IP addresses and two default routes

**Symptom**
`ip a` showed both `192.168.1.10` (static) and `192.168.1.139` (dynamic) on ens33. `ip route` showed two default routes.

**Diagnosis**
The `.139` address was flagged `dynamic` with `proto dhcp`, `noprefixroute` and `metric 101`. That pattern points to NetworkManager. `systemctl is-active NetworkManager` confirmed it was running.

**Root cause**
NetworkManager was managing ens33 alongside ifupdown and requesting its own DHCP lease, independently of the static config in `/etc/network/interfaces`.

**Fix**

```
sudo systemctl disable --now NetworkManager
sudo systemctl mask NetworkManager
sudo ip addr flush dev ens33
sudo systemctl restart networking
```

Rebooted to confirm the fix persisted, then set DNS manually in `/etc/resolv.conf`, since NetworkManager had been managing it.

**Lesson**
A device should have one network configuration manager. Two tools managing the same interface produce conflicting state that's hard to spot without checking route metrics and address flags.

---

## 3. WAN static IP was inside the VMware DHCP range

**Symptom**
No visible failure yet. Spotted during review.

**Diagnosis**
The WAN was statically set to `192.168.1.139`, which falls inside VMware NAT's DHCP pool (typically .128–.254).

**Root cause**
VMware's DHCP server doesn't know about static assignments, so it could hand the same address to another VM and cause an IP conflict.

**Fix**
Moved the WAN to `192.168.1.10`, outside the DHCP range.

**Lesson**
Static addresses belong outside the DHCP pool. The admin route and firewall rules also depend on the router's address staying fixed.

---

## 4. Router could not ping the Windows admin host

**Symptom**
`ping 192.168.1.1` from the router failed, but `ping 192.168.1.10` from Windows succeeded.

**Diagnosis**
Since traffic worked in one direction, the network path was fine and the block was on the Windows side.

**Root cause**
Windows classifies the VMware adapter as an "Unidentified network" under the Public firewall profile, which blocks inbound ICMP echo by default.

**Fix**
Not required for the lab: admin traffic is initiated from Windows. Where needed for troubleshooting, a scoped rule allows ping from the router only:

```
New-NetFirewallRule -DisplayName "Lab router ICMP" -Direction Inbound -Protocol ICMPv4 -IcmpType 8 -RemoteAddress 192.168.1.10 -Action Allow
```

**Lesson**
Testing in both directions isolates which side is blocking. The fix was scoped to one source address rather than opening ICMP for the whole profile.

---

## 5. Windows admin host could not reach vm-scan (10.10.30.10)

**Symptom**
`ping 10.10.30.10` from Windows failed.

**Diagnosis**
print ROUTE on windows to check if it has a route the specified destination

**Root cause**
No root to 10.10.30.0/24 from windows as there is no routing protocol installed.

**Fix**
Add route 10.10.30.0/24 via 192.168.1.1 on windows machine

**Lesson**
Always check path for the destination

---
