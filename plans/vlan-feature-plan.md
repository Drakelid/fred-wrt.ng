# VLAN Feature Implementation Plan — ZenWiFi XT8 (Asuswrt-Merlin, gnuton fork)

**Target:** ASUS ZenWiFi XT8 (internal model `RT-AX95Q`, BCM6755 / HND 5.02 `src-rt-5.02axhnd.675x`)
**Base:** `gnuton/asuswrt-merlin.ng`, tag `3004.388.11_1-gnuton1`
**Status:** Draft v1 — items marked **[VERIFY]** must be confirmed against the checked-out source tree before implementation starts.

---

## 1. Goals and scope

### In scope

1. User-definable VLANs (802.1Q) with per-VLAN:
   - IPv4 subnet, DHCP scope, DNS behaviour, and optional IPv6 sub-prefix
   - Wired port membership (untagged/access, or tagged/trunk on a chosen port)
   - Wireless SSID membership (one or more VAPs bound to the VLAN bridge)
   - Firewall policy: internet access on/off, inter-VLAN policy, router management access
2. Secure-by-default posture: a new VLAN is isolated from every other VLAN and from the router's management plane unless a rule explicitly allows it.
3. Persistence across reboot, service restarts, and firmware upgrade (config export/import).
4. Deterministic, atomic firewall application — a partial failure must never leave the device more permissive than the previous state.
5. Web UI to create, edit, and delete VLANs (see the companion UI plan; the VLAN screens are the first target of the new design system).

### Out of scope (phase 1)

- Dynamic VLAN assignment via 802.1X / RADIUS (dot1x VLAN attributes) — phase 3 candidate.
- Propagating VLANs across AiMesh nodes (see §3.4 — a hard platform limitation on the 3004.388 branch).
- Multi-WAN per VLAN — phase 2 candidate. (Per-VLAN tunnel routing is covered in §12.6.)

### Adjacent features

Seven features build directly on this foundation and share its config schema, applier, and firewall generator — commit-confirmed apply, device onboarding with automatic VLAN assignment, per-VLAN DNS policy, metrics and alerting, internet quality monitoring, WireGuard policy routing, and time-limited guest credentials. They are specified in §12. One of them, commit-confirmed apply with auto-rollback (§12.1), is a **prerequisite** for the core work: it is the safety net for the moment a VLAN or firewall change locks the operator out of the management plane.

### Success criteria

- The isolation matrix in §5 passes in full, verified by the adversarial tests in §9.3.
- No measurable loss of routed WAN throughput versus stock firmware once hardware acceleration behaviour is settled (§4.6).
- Reboot, `service restart_firewall`, `service restart_net`, WAN flap, and wireless restart all leave the rule set intact (§9.4).

---

## 2. Platform constraints

| Constraint | Implication |
|---|---|
| Broadcom HND 5.02 platform, `bcm675x` | The switch and VLAN tagging are managed by Broadcom tooling (`ethswctl`, `vlanctl`), not the older `robocfg` used on ARM/AC models. **[VERIFY]** which of `vlanctl`, `ip link ... type vlan`, and `ethswctl` are present and functional in this build. |
| Flow acceleration (Runner / Archer / `fcctl`) | Hardware-accelerated flows can bypass `iptables` once a flow is learned. Any isolation rule that relies solely on netfilter must be validated with acceleration enabled, not just with `fc disable`. |
| Wireless VAPs | Guest/extra SSIDs are virtual interfaces (`wl0.1`, `wl0.2`, `wl1.1`, …). Each radio has a limited VAP count. **[VERIFY]** the per-radio VAP limit and which indices ASUS already reserves for guest networks and AiMesh backhaul. |
| AiMesh backhaul | AiMesh nodes exchange traffic on a dedicated backhaul; ASUS only propagates its own guest network constructs. Custom VLANs terminate at the router. |
| NVRAM size | NVRAM is finite (tens of KB). A per-VLAN key explosion is a real risk; use a compact packed schema (§6). |
| JFFS | Available for scripts and generated config, but not guaranteed mounted early in boot. Scripts must fail safe if JFFS is absent. |
| Custom `httpd` | The web server is ASUS's own. New API surface should reuse `appGet.cgi` / `applyapp.cgi` conventions rather than introducing a second auth model. |

---

## 3. Architecture

### 3.1 Data plane

Each VLAN is a Linux bridge with its own L3 interface:

```
        WAN (eth0 / ppp0)
             │  NAT
        ┌────┴───────────────────────────────────────────┐
        │                 router (netfilter)             │
        └─┬────────────┬────────────┬────────────┬───────┘
      br0 │        br10│        br20│        br30│
   (LAN,  │  (Trusted) │   (IoT)    │  (Guest)   │
    mgmt) │            │            │            │
  ┌───────┴──┐   ┌─────┴─────┐ ┌────┴─────┐ ┌────┴─────┐
  │ LAN1..3  │   │ lanX.v10  │ │ lanX.v20 │ │ lanX.v30 │  (tagged, trunk port)
  │ wl0/wl1  │   │  wl0.1    │ │  wl0.2   │ │  wl0.3   │  (VAPs)
  └──────────┘   └───────────┘ └──────────┘ └──────────┘
```

Components per VLAN *n*:

- A tagged sub-interface on the switch/trunk port, created with `vlanctl` (HND) or the 8021q module **[VERIFY]**.
- A bridge `br<n>` holding: the tagged sub-interface(s), plus any wireless VAPs assigned to that VLAN.
- An IPv4 address on `br<n>` acting as the gateway for that subnet.
- Optional access (untagged) LAN ports: the port's PVID is set to the VLAN ID so untagged client traffic lands in the right bridge.

Design rule: **the LAN ports that carry untagged client traffic are never trunk ports.** Only an explicitly designated uplink port carries tagged frames, which removes the classic VLAN-hopping vector at the source.

### 3.2 Address and name services

- **DHCPv4:** one `dnsmasq` configuration serving all bridges, using `interface=` + per-interface `dhcp-range=` stanzas, with `bind-interfaces` so no listener is exposed on unintended interfaces. A separate instance per VLAN is the fallback if the single-instance approach conflicts with ASUS's generated config. **[VERIFY]** how ASUS regenerates `/etc/dnsmasq.conf` and which hook (`dnsmasq.postconf`) survives service restarts.
- **DNS:** each VLAN resolves through the router. Guest/IoT VLANs get a forced-DNS rule (§5) so hardcoded resolvers in devices cannot bypass filtering.
- **IPv6:** if the ISP delegates a prefix shorter than /64 (typically /56), carve a /64 per VLAN and run RA/DHCPv6 per bridge. If only a /64 is delegated, IPv6 is enabled on the primary LAN only and other VLANs are IPv4-only — this must be surfaced in the UI, not silently degraded.
- **NTP:** the router's NTP service is allowed from all VLANs (or explicitly denied for lockdown profiles).

### 3.3 Wireless mapping

Each SSID is a VAP bound to exactly one bridge. Per-VLAN wireless hardening:

- `ap_isolate` on guest/IoT VLANs (blocks station-to-station traffic inside the same VAP).
- WPA3-SAE or WPA2/WPA3 transition mode; PMF required where the client mix allows it.
- Separate PSK per VLAN; never reuse the main LAN PSK.
- Band steering and 802.11r left off for IoT VLANs unless the device population is known to handle it.

### 3.4 AiMesh interaction (known limitation)

On the 3004.388 branch, custom VLANs do not traverse the AiMesh backhaul: clients connected to a mesh node land in the node's default LAN. Options:

1. **Accept and document** — VLAN SSIDs are broadcast by the router only (phase 1 default).
2. **Wired backhaul + trunk** — if the node is wired, tag the VLANs on the link and treat the node as a dumb AP with a matching bridge/VLAN config. Requires per-node customisation and is fragile across firmware updates.
3. **Move to the 3006 branch** where ASUS's SDN / Guest Network Pro provides VLAN-aware guest propagation. This is a significant rebase, evaluated separately.

The UI must state this limitation explicitly at VLAN creation time when AiMesh nodes are present.

### 3.5 Where the feature lives: firmware vs. script overlay

| Approach | Pros | Cons |
|---|---|---|
| **A. Script overlay** on stock firmware (`/jffs/scripts/`, `firewall-start`, `services-start`, `service-event`) | Fast iteration, no build cycle, easy rollback, no flash budget impact | No native UI, fights ASUS's own config generation, breaks silently after some `service restart_*` calls |
| **B. Firmware feature** (patch the source tree we cloned) | Native UI, correct lifecycle integration, survives service restarts, shippable | Long build cycle, flash budget, must track upstream merges |

**Recommendation:** build the logic script-first (A) to validate behaviour and the isolation matrix quickly on the live device, then port the validated logic into the firmware (B) behind the same NVRAM schema. The scripts become the reference implementation and the integration test harness.

Firmware touch points to confirm after checkout **[VERIFY]**:

- `release/src/router/rc/` — network/bridge bring-up, firewall generation, service lifecycle
- `release/src/router/shared/defaults.c` — NVRAM defaults for the new keys
- `release/src/router/httpd/` — `appGet.cgi` handlers for the new API
- `release/src/router/www/` — UI pages
- `release/src/router/others/` — Merlin-specific additions and script hook points

---

## 4. Detailed design

### 4.1 Bring-up order

1. Read and validate the VLAN config from NVRAM (§6). Reject malformed entries; never partially apply.
2. Create tagged interfaces and bridges; set PVIDs on access ports.
3. Assign addresses, set per-bridge sysctls (`rp_filter=1`, `arp_ignore=1`, `arp_announce=2`, `accept_redirects=0`, `send_redirects=0`, forwarding as required).
4. Bind VAPs to bridges.
5. Regenerate DHCP/DNS/RA config and restart those services.
6. Build the complete firewall rule set in memory and apply atomically (§4.3).
7. Emit a state file plus a log line summarising what was applied; expose it to the UI for diagnostics.

### 4.2 Failure handling

- Every step is idempotent — re-running the applier converges to the same state.
- On any failure, the applier restores the previous rule set (kept as a snapshot) and raises an error visible in the UI and syslog. The default in an unknown state is *deny*, not *allow*.
- A boot-time watchdog re-applies the config if the expected bridges or chains are missing (guards against races with ASUS's own service restarts).

### 4.3 Firewall model

Rules are generated into dedicated chains so they can be flushed and rebuilt without touching ASUS's own rules:

```
FORWARD  → VLAN_FWD   (jump inserted at the top)
INPUT    → VLAN_IN    (jump inserted at the top)
```

`VLAN_FWD` policy:

1. `ESTABLISHED,RELATED` accept.
2. Explicit inter-VLAN allow rules (per user policy), most-specific first.
3. VLAN → WAN accept, if internet is enabled for that VLAN.
4. Everything else: log (rate-limited) and drop.

`VLAN_IN` policy (router management plane):

1. From each VLAN: allow DHCP (67/68 UDP), DNS (53 UDP/TCP) and ICMP echo, and NTP if enabled.
2. Deny web UI (80/443/8443), SSH (22), Telnet, `9999`/`5000`-range ASUS services, UPnP/SSDP, SNMP, and everything else from untrusted VLANs.
3. Management access is allowed only from the management VLAN, and optionally restricted to a source IP allow-list.

Applied atomically via `iptables-restore` on a generated ruleset (build to a temp file, validate with `iptables-restore --test`, then commit), rather than a long sequence of individual `iptables -I` calls.

L2 controls with `ebtables`:

- Drop DHCP server responses (UDP 67 → 68) originating from client ports — kills rogue DHCP servers.
- ARP sanity: drop ARP frames whose sender MAC/IP disagree with the frame source; drop gratuitous ARP claiming the gateway address.
- Block cross-bridge multicast leakage for SSDP (`239.255.255.250`) and mDNS (`224.0.0.251` / `ff02::fb`) unless a service-discovery bridge rule is explicitly enabled.
- Optional MAC allow-list per VLAN for the most sensitive segments.

### 4.4 Forced DNS

For VLANs flagged `dns_force`:

- `PREROUTING` DNAT of UDP/TCP 53 to the bridge address.
- Drop outbound 853 (DoT) and known DoH bootstrap IPs if strict mode is on — with a clear UI warning that this breaks some client stacks.

### 4.5 IPv6

IPv6 rules mirror the IPv4 chains in `ip6tables`. Common failure: IPv4 isolation is correct and IPv6 traffic walks straight between segments. IPv6 rules are generated by the same code path, not written by hand, and the test matrix runs twice — once per family.

### 4.6 Hardware acceleration

- Determine empirically whether accelerated flows can bypass the `VLAN_FWD` chain once established. **[VERIFY]** with a bidirectional test: start an allowed flow, revoke the rule, confirm the flow dies.
- If bypass is confirmed, options are: exclude inter-VLAN traffic from acceleration (keeping WAN NAT acceleration intact), or disable acceleration entirely on affected paths and document the throughput cost.
- Whichever path is chosen, record the measured throughput before and after in the test report.

---

## 5. Threat model and isolation matrix

Assumed attacker: a compromised device inside a low-trust VLAN (an IoT camera, a guest laptop), with full L2 access to its own segment.

| Threat | Control |
|---|---|
| Lateral movement to trusted LAN | Default-deny `VLAN_FWD`, explicit allow-list only |
| Router management takeover | `VLAN_IN` denies UI/SSH from untrusted VLANs; management restricted to mgmt VLAN |
| VLAN hopping (double tagging) | No trunk on client-facing ports; native VLAN set to an unused ID; double-tagged frames dropped |
| Switch spoofing / dynamic trunking | Not applicable on this hardware, but ports are pinned to access mode explicitly |
| Rogue DHCP | `ebtables` drop of client-sourced DHCP offers |
| ARP/ND spoofing of the gateway | `ebtables` ARP checks + `arp_ignore`/`arp_announce` sysctls; ND handled by matching `ip6tables` rules |
| DNS exfiltration / bypass | Forced DNS, optional DoT/DoH blocking, per-VLAN logging |
| Multicast/service-discovery leakage | mDNS/SSDP blocked across bridges by default |
| Station-to-station attack inside a guest SSID | `ap_isolate` per VAP |
| Firmware/config rollback leaving VLANs open | Applier fails closed; watchdog re-applies; config version recorded in NVRAM |

**Isolation matrix** (default policy for a new VLAN):

| From \ To | Mgmt LAN | Other VLAN | Router mgmt ports | Router DHCP/DNS | WAN |
|---|---|---|---|---|---|
| Mgmt LAN | — | allow (configurable) | allow | allow | allow |
| Trusted VLAN | deny | deny | deny | allow | allow |
| IoT VLAN | deny | deny | deny | allow | configurable |
| Guest VLAN | deny | deny | deny | allow (forced) | allow |

---

## 6. Configuration schema (NVRAM)

Compact, versioned, one packed key per VLAN plus an index, to avoid NVRAM key explosion:

```
vlan_cfg_ver = 1
vlan_list    = 10 20 30
vlan_10      = name=IoT;subnet=10.0.10.1/24;dhcp=10.0.10.100-10.0.10.200;lease=86400;
               wan=1;dnsforce=1;isolate=1;ports=;vaps=wl0.2,wl1.2;v6=auto
vlan_rule_10 = allow:10->0:tcp:8123;allow:0->10:any
```

Rules:

- Every field validated on write (server side, never trusting the UI): VLAN ID 2–4093, subnet non-overlapping with WAN/LAN/VPN pools, DHCP range inside the subnet, VAP names from a known list.
- `vlan_cfg_ver` gates migration logic on firmware upgrade.
- Export/import as a single JSON blob for backup, with secrets (PSKs) handled separately.

---

## 7. API surface

Phase 1 reuses existing conventions:

- `appGet.cgi?hook=vlan_config()` — returns the parsed config plus live state (bridge up, client counts, applied-at timestamp).
- `applyapp.cgi?action_mode=apply&rc_service=restart_vlan` — writes NVRAM and triggers the applier.
- All writes require the existing session token and pass the existing referer/CSRF checks — no new auth path.
- Server-side validation is authoritative; the UI's validation is a convenience only.

---

## 8. Security review checklist (gate before merge)

- [ ] No shell interpolation of user-supplied strings without validation (VLAN names reach `iptables`/`dnsmasq` config — treat as hostile input; enforce a strict charset)
- [ ] Rule set applied atomically; `--test` before commit
- [ ] Fails closed on every error path
- [ ] IPv4 and IPv6 rules generated from one source of truth
- [ ] No management service reachable from an untrusted VLAN (verified by scan, not by reading code)
- [ ] Acceleration bypass tested and documented
- [ ] Secrets (PSKs) never written to logs or the state file
- [ ] Log volume rate-limited so a flood cannot fill flash or evict useful logs
- [ ] Config import validates before applying and rejects unknown schema versions
- [ ] `security-reviewer` agent pass on the final diff

---

## 9. Test plan

### 9.1 Unit / static

- `shellcheck` clean on all scripts; deterministic rule-generation tests (given config → expected `iptables-restore` text, byte-compared against golden files).
- Config parser fuzzing with malformed NVRAM values (oversized, injected `;`, backticks, unicode, out-of-range IDs).

### 9.2 Functional integration (per VLAN)

- Client gets the correct IP, gateway, DNS, lease time on wired access port and on the VAP.
- Internet reachable/blocked according to policy; DNS forced where configured.
- IPv6 addressing where a prefix is available.

### 9.3 Adversarial

- `nmap -sn` and `arp-scan` from each VLAN — must see only the gateway.
- Port scan of the gateway from each VLAN — only the intended ports respond.
- Double-tagged frame injection (scapy) from an access port — must not reach another VLAN.
- Rogue DHCP server on a guest port — clients must not accept the rogue lease.
- Gateway ARP spoof attempt — blocked; the victim's traffic keeps flowing to the real gateway.
- mDNS/SSDP discovery across VLANs — no cross-segment responses.
- Established-flow revocation test against hardware acceleration (§4.6).

### 9.4 Resilience

- Reboot; `service restart_firewall`; `service restart_net`; `service restart_wireless`; WAN flap; AiMesh node re-onboarding; firmware upgrade with config carried over. After each: re-run 9.3 in full.

### 9.5 Performance

- `iperf3` WAN↔LAN and LAN↔LAN, acceleration on and off, baseline vs. feature build. Record CPU utilisation. Accept no more than a documented, deliberate regression.

---

## 10. Rollout and rollback

1. Back up NVRAM and JFFS before the first apply; store the backup off-device.
2. Stage on a spare/lab XT8 if available; otherwise apply during a maintenance window with a wired management client on the mgmt VLAN.
3. Keep a serial/recovery path known-good: ASUS Rescue Mode + a stock firmware image on hand before flashing any custom build.
4. Rollback = restore NVRAM backup + reflash the previous image. Document the exact steps in the runbook before the first flash, not after.

---

## 11. Work breakdown

| Milestone | Content | Exit criterion |
|---|---|---|
| M0 | Source verification: resolve every **[VERIFY]** item; document actual tooling and hook points | Written findings doc |
| M0.5 | Commit-confirmed apply with auto-rollback (§12.1) | A deliberate lockout self-recovers without operator action |
| M1 | Script-based applier: config parser, bridge/VLAN creation, atomic firewall generation | One VLAN live, isolation matrix passes for it |
| M2 | DHCP/DNS/IPv6 integration, wireless VAP binding, forced DNS | Three VLANs (trusted/IoT/guest) live |
| M3 | Adversarial + resilience test pass; acceleration decision made and documented | Full §9 report green |
| M4 | Firmware port: NVRAM schema, rc integration, `httpd` API | Feature survives all service restarts natively |
| M5 | Web UI (first screens of the new design system — see the UI plan) | Create/edit/delete VLAN from the UI |
| M6 | Build, sign-off, security review, flash to production device | `security-reviewer` clean; runbook written |
| M7+ | Adjacent features in the order given in §12.8 | Each ships independently and revertibly |

---

## 12. Adjacent features built on the VLAN foundation

These extend the same config schema, applier, and firewall generator. They are sequenced after the core VLAN work except where noted — §12.1 lands **first**, before any VLAN change is applied to a production device.

### 12.1 Commit-confirmed apply with auto-rollback

**Priority: build this before the first VLAN apply.** JunOS-style safety: any change to VLANs, firewall policy, or interface configuration is applied with a rollback timer armed. If the operator does not confirm within N minutes (default 10, configurable 1–60), the previous configuration is automatically restored.

Design:

- Reuses the snapshot mechanism from §4.2. A snapshot captures NVRAM keys in the feature's namespace plus the generated rule set and interface state.
- Apply sequence: snapshot → apply → arm timer → UI shows a persistent confirm banner with countdown → operator clicks *Keep changes* → timer disarmed and snapshot retired.
- Rollback path runs from a detached process that survives the UI session dying and does not depend on the network state it is about to restore.
- Confirmation must come over an authenticated session; a page reload is not confirmation. If the operator loses management access, silence is the signal to roll back.
- Rollback events are logged with the diff that was reverted, so the failed change can be inspected afterwards.
- Escape hatch: a boot-time check for an un-confirmed pending change that reverts at startup, covering the case where the operator power-cycles a bricked-network router instead of waiting.

Dependencies: §4.2 applier snapshots. Effort: small. Risk: low — the failure mode of the safety net itself is "reverts a change you wanted", which is recoverable.

**Exit criterion:** deliberately apply a rule that blocks the management client; the device restores access without operator intervention.

### 12.2 Device onboarding and automatic VLAN assignment

Without this, segmentation is a one-time setup ritual that decays the moment a new device joins. With it, segmentation is an ongoing property of the network.

Design:

- A **quarantine VLAN** is the default landing zone for any unrecognised MAC: DHCP works, DNS works, internet is blocked, inter-VLAN is blocked.
- Classification rules, evaluated in order, each producing a target VLAN:
  1. Manual pin (operator assigned this MAC to this VLAN — always wins, never overridden)
  2. Hostname pattern match
  3. DHCP fingerprint (option 55 parameter request list, vendor class option 60) — a strong signal for device type
  4. MAC OUI lookup against a bundled vendor table
  5. Fallback: stay in quarantine
- Assignment is enforced at the bridge level. **[VERIFY]** whether the wireless driver on this SDK supports per-station VLAN steering on a single VAP; if not, wireless devices are assigned by moving them to the SSID bound to the target VLAN, which requires the operator to re-provision the client's credentials. This distinction must be explicit in the UI — an automatic wired reassignment and a manual wireless one are very different user experiences.
- MAC randomisation: iOS/Android private addresses break MAC-based identity. Track devices by a composite identity (MAC + DHCP fingerprint + hostname) and surface a "this looks like a device you already know" prompt rather than silently trusting a randomised MAC.
- Notification on every new device entering quarantine (ties to §12.4).

Dependencies: core VLAN feature, DHCP integration (§3.2). Effort: medium. Risk: medium — classification false positives put a device in the wrong segment; mitigate with quarantine-by-default and an explicit approval step for anything above the lowest trust tier.

### 12.3 Per-VLAN DNS policy with native blocklists

Third-party script suites (Diversion and similar) do this today but break on firmware upgrades and fight with ASUS's own dnsmasq config generation. Building it natively makes it survive upgrades and integrates with the forced-DNS rule in §4.4.

Design:

- Per-VLAN resolver choice: router-local, specified upstream, DoT, or DoH. Trusted VLAN and guest VLAN can resolve differently — that is the point.
- Per-VLAN blocklist sets, from subscribed lists (hosts-format and domain-list) plus manual allow/deny entries. Lists are refreshed on a schedule with a signature/size sanity check; a failed refresh keeps the previous list rather than falling open.
- Storage: compiled to dnsmasq server/address entries or an equivalent lookup structure; keep memory use bounded and measured — a large blocklist on a 512 MB device competes with everything else. Set an explicit entry-count ceiling and report usage in the UI.
- Query log with per-VLAN top-talkers, top-blocked, and per-client drill-down. Retention bounded, written to USB or RAM with periodic flush — never a high-write-rate log on internal flash.
- Privacy note in the UI: query logging records browsing behaviour of everyone on the network. Off by default, with retention configurable and a clear on-screen statement when enabled.

Dependencies: §3.2 DHCP/DNS integration, §4.4 forced DNS. Effort: medium. Risk: medium — the classic failure is a blocklist update taking DNS down for the whole house; hence fail-closed-to-previous-list and a watchdog that reverts to plain forwarding if resolution fails.

### 12.4 Metrics endpoint and alerting

Turns the router from a black box into a monitored system, and gives the VLAN work an evidence base — throughput and drop counters per segment.

Design:

- A Prometheus-format `/metrics` endpoint exposing: per-client and per-VLAN throughput and packet counts, firewall drop counters per chain, radio airtime and station counts, RSSI distribution, CPU/memory/temperature, WAN latency and uptime, DHCP lease usage per VLAN.
- The endpoint is served on the management interface only, requires a scoped read-only token, and is rate-limited. It is not exposed to any untrusted VLAN and never to WAN.
- Collection cost must stay marginal: sample on a fixed interval into a small in-memory structure, do not shell out per scrape.
- Alerting with pluggable transports (webhook, ntfy, Telegram, email) for: new device in quarantine, WAN down or degraded, AiMesh node offline, firewall drop-rate spike from a given segment, DNS resolution failure, unexpected configuration change, rollback triggered by §12.1.
- Alerts are rate-limited and de-duplicated — an alert channel that cries wolf gets muted, which is worse than no alerting.

Dependencies: none hard; benefits from VLAN counters existing. Effort: medium. Risk: low, provided the endpoint's auth and exposure rules are enforced (this is a new listener — it gets its own security review).

### 12.5 Internet quality monitor

Answers "is it the ISP or my network" with data instead of argument.

Design:

- Continuous low-rate latency/jitter/loss probing to a small set of anchors (ISP gateway, a public resolver, a well-known endpoint), so the failure can be localised to the last mile, the ISP, or beyond.
- Scheduled bufferbloat test (loaded latency under saturation) at a configurable off-peak time, with results kept as history rather than a one-shot number.
- History retained with bounded storage and rendered as a timeline, annotated with WAN events (link down, PPPoE reconnect, IP change).
- Export of a time-window summary suitable for pasting into an ISP support ticket.
- Probing must not itself distort measurements: low packet rate, and the saturation test never runs automatically during a period the operator marks as busy.

Dependencies: §12.4 for storage and alerting reuse. Effort: small to medium. Risk: low.

### 12.6 WireGuard policy routing

WireGuard client and server exist in the 388 base, but routing policy is awkward. With per-VLAN bridges in place, "route this whole segment through the tunnel" becomes a natural primitive. This supersedes the phase-2 "policy routing per VLAN" item in §1.

Design:

- Route selection at VLAN granularity (whole segment through a tunnel), with per-client overrides inside a segment.
- **Verified kill switch:** when a tunnel is down, traffic assigned to it is dropped, not silently sent out the WAN. Implemented as an explicit drop rule tied to tunnel state, and *tested* by taking the tunnel down and confirming the leak does not occur — including for IPv6 and for DNS.
- Split tunnelling by destination, with domain-based rules resolved to address sets rather than matched per packet on hostnames.
- DNS handling per tunnel: traffic routed through a tunnel resolves through that tunnel's resolver, otherwise the exit-node choice leaks in the DNS path even when the data path is correct.
- Interaction with hardware acceleration is the same open question as §4.6 and is tested the same way.

Dependencies: core VLAN bridges, §4.3 firewall chains, §12.3 for the DNS half. Effort: medium to large. Risk: medium — kill-switch correctness is a security property, not a convenience; it gets explicit adversarial tests in §9.3's style.

### 12.7 Guest access with time-limited credentials

The guest VLAN already exists in the core plan; this makes it usable the way a UniFi-style guest portal is.

Design:

- Time-limited access credentials: either expiring per-guest PSKs (where the driver supports multiple PSKs per SSID — **[VERIFY]**) or voucher codes redeemed on a local captive portal page.
- Per-guest bandwidth cap and session duration; automatic expiry and cleanup with no manual gardening.
- Scheduling: the guest SSID can be enabled only during defined windows.
- Guest devices remain fully isolated per the §5 matrix; expiry revokes access without touching other segments.
- If a captive portal is used, it is served locally, over the guest bridge only, with no external dependencies and no third-party analytics. It must not become a second authentication surface for the router — portal credentials grant network access only, never management access.

Dependencies: core VLAN feature, wireless VAP mapping (§3.3). Effort: medium. Risk: medium — captive portals interact badly with client OS probe behaviour; if multi-PSK is available in the driver, prefer it and skip the portal entirely.

### 12.8 Sequencing

| Order | Feature | Gate |
|---|---|---|
| 0 | §12.1 Commit-confirmed apply | Before any VLAN change reaches a production device |
| 1 | Core VLAN feature (M1–M4) | Isolation matrix green |
| 2 | §12.2 Onboarding / auto-assignment | Quarantine VLAN behaviour verified |
| 3 | §12.3 Per-VLAN DNS and blocklists | Memory ceiling measured; fail-safe verified |
| 4 | §12.4 Metrics and alerting | Endpoint auth reviewed |
| 5 | §12.5 Internet quality monitor | Reuses §12.4 storage |
| 6 | §12.6 WireGuard policy routing | Kill switch adversarially tested |
| 7 | §12.7 Guest credentials | Multi-PSK support determined |

Each item ships independently and is individually revertible. None of them should be started while the isolation matrix for the core feature is failing.

---

## 13. Open questions

1. Which VLAN tooling is actually functional in this build — `vlanctl`, 8021q, or both? (M0)
2. How many VAPs remain per radio after AiMesh and ASUS guest networks reserve theirs? (M0)
3. Does the flow accelerator bypass netfilter for inter-VLAN traffic on this SDK? (M3)
4. Is the 3006 branch (SDN / Guest Network Pro) a better long-term base, given AiMesh VLAN propagation? Decide before M4 — it changes the porting target.
5. Flash budget: how much free space remains in the XT8 image at this tag, after the UI work? (shared gate with the UI plan)
6. Does the wireless driver support per-station VLAN steering on a single VAP, or must each VLAN have its own SSID? Decides how §12.2 reassigns wireless devices. (M0)
7. Does the driver support multiple PSKs per SSID? If yes, §12.7 ships expiring per-guest keys and skips the captive portal entirely. (before §12.7)
8. What is the practical blocklist entry ceiling on 512 MB of RAM alongside everything else running? Measure before committing to a list-subscription UI. (before §12.3)
