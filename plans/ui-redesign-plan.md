# UI / UX Redesign Plan — Asuswrt-Merlin (gnuton) on ZenWiFi XT8

**Base:** `gnuton/asuswrt-merlin.ng`, tag `3004.388.11_1-gnuton1`
**Design reference:** Ubiquiti UniFi Network — used as a *pattern and quality* reference only (see §13 on legal boundaries).
**Status:** Draft v1 — items marked **[VERIFY]** depend on the source tree currently being checked out.

---

## 1. Why replace the current UI

The ASUSWRT web UI is a ~2010-era server-rendered application: `.asp` pages with embedded template expressions rendered by ASUS's custom `httpd`, a large body of hand-written JavaScript with global state, table-driven layouts, iframe-based navigation, per-model page variants, and three visual themes (`default`, `tuf`, `rog` — visible in the CI build matrix). It works, but:

- Information density is poor: critical state (clients, throughput, security posture) is spread across many pages.
- No consistent component vocabulary — each page reinvents forms, tables, and dialogs.
- Mobile behaviour is an afterthought; touch targets and layout break below tablet width.
- Accessibility is largely absent (focus management, labels, contrast).
- Adding a feature like the VLAN manager means writing another bespoke page in the old idiom.

UniFi is the right reference point because it solves the same problem class — many networks, many clients, dense telemetry — with a calm, consistent, dark-first interface built around cards, drawers, and data tables rather than nested forms.

---

## 2. Constraints (these shape every decision below)

| Constraint | Implication |
|---|---|
| Flash budget | The firmware image must still fit the XT8 partition. Every KB of UI competes with firmware. Hard budget in §9; a size gate runs in CI. **[VERIFY]** current free space at this tag. |
| No internet at render time | No CDN, no Google Fonts, no runtime package fetch. Fonts, icons, and libraries ship in the image, subsetted. |
| Custom `httpd` | Limited MIME handling, no HTTP/2, possibly no automatic compression. **[VERIFY]** MIME table, gzip support (`.gz` sidecar convention), and cache headers. |
| Existing auth model | Session cookie/token plus referer and host checks. The new UI must fit it exactly — no second auth path, no weakening of CSRF protection. |
| Existing data API | `appGet.cgi?hook=…()` for reads, `applyapp.cgi` / `apply.cgi` for writes, plus service-restart actions. Reuse before extending. |
| Per-model page filtering | The build strips or swaps pages per model. Any new asset pipeline must respect that mechanism. **[VERIFY]** how `www/Makefile` and the model filters work. |
| Router CPU | Rendering must stay cheap; heavy client-side work is fine on a laptop but the router only serves static files, so the budget is really about payload size and parse time on the *client*. |
| ASUS mobile app + AiMesh | The app talks to the same CGI endpoints. Breaking or renaming endpoints breaks the app. Endpoints are additive-only. |

---

## 3. Strategy: reskin now, replace incrementally

| Option | Verdict |
|---|---|
| **A. Design-token reskin** — replace the stylesheets, keep the existing pages and markup | Ship first. Low risk, immediate visual gain, no API work. Ceiling: layout and interaction stay old. |
| **B. New SPA shell, page-by-page migration** — a modern app that owns navigation and the new screens, with legacy pages embedded until each is migrated | The real target. Strangler-fig migration keeps the device usable at every commit. |
| **C. Big-bang rewrite** | Rejected. Hundreds of model-specific pages; no safe cut-over point; guarantees regressions in features nobody remembers testing. |

**Plan: A → B.** Phase 1 delivers the token system and a reskin. Phase 2 stands up the shell and migrates screens in priority order, with the VLAN manager (from the companion plan) as the first native screen — a new feature is the cheapest place to prove the new stack.

---

## 4. Design direction

The reference qualities to hit, stated as design principles:

1. **Calm surface, dense data.** Neutral greys carry the interface; colour is reserved for state and one accent. No gradients, no chrome, no decorative iconography.
2. **One object model.** Everything is a *thing with a detail drawer*: a client, a network, a port, a rule. Click the row, the drawer opens, the list stays put — no full-page navigation for inspection.
3. **Status is glanceable.** Every screen answers "is anything wrong?" in the first 200 ms of looking, via a consistent status colour vocabulary.
4. **Progressive disclosure.** Defaults visible; advanced settings behind a clearly labelled section, never behind a mystery toggle.
5. **Dark-first, light-equal.** Both themes are first-class, driven by the same tokens.
6. **Motion is functional.** 120–180 ms transitions for drawers and toggles; nothing decorative; respects `prefers-reduced-motion`.

### Screen archetypes

- **Dashboard** — WAN/internet health, per-radio load, client count, throughput sparklines, alerts. Cards, not a wall of gauges.
- **Clients** — dense sortable table: name, IP, VLAN, connection (band/RSSI/port), throughput, uptime. Row → drawer with history, blocks, static-lease and VLAN assignment.
- **Networks (VLANs)** — list of segments with subnet, client count, policy summary, isolation state. Create/edit in a drawer with a live policy preview and a plain-language summary of what the VLAN can reach.
- **Wireless** — SSID list; per-SSID drawer with band, security, VLAN binding, isolation, schedule.
- **Firewall / Policies** — rules as readable sentences plus an expert view showing the generated rule set.
- **Topology** — router + AiMesh nodes + clients as a graph, link quality on edges.
- **Insights / Diagnostics** — logs, ping/trace, packet capture handoff, throughput tests.
- **System** — firmware, backup/restore, scripts, JFFS, reboot.

These are archetypes, not the full list. The complete screen inventory — every legacy page and where it lands, plus every new screen introduced by the VLAN plan — is §15. The data and control wiring for each screen is §16.

---

## 5. Design tokens

A single token file is the source of truth, exported as CSS custom properties and consumed by every component. Illustrative shape (values to be finalised in the design pass, contrast-checked to WCAG 2.2 AA):

```css
:root {
  /* neutrals — light */
  --bg-canvas: #f6f7f9;  --bg-surface: #ffffff;  --bg-sunken: #eceff3;
  --border-subtle: #e2e6ec; --border-strong: #c8d0da;
  --text-primary: #10151c; --text-secondary: #56606d; --text-tertiary: #8a949f;

  /* accent + state */
  --accent: #0b6bcb; --accent-hover: #0a5cb0; --accent-subtle: #e7f0fb;
  --ok: #1f9d5a; --warn: #c08117; --danger: #c0392b; --info: #4a7fbf;

  /* type */
  --font-ui: "InterVariable", system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
  --font-mono: "JetBrains Mono", ui-monospace, SFMono-Regular, Menlo, monospace;
  --fs-xs: 11px; --fs-sm: 12px; --fs-md: 14px; --fs-lg: 16px; --fs-xl: 20px; --fs-2xl: 28px;

  /* space, radius, elevation, motion */
  --sp-1: 4px; --sp-2: 8px; --sp-3: 12px; --sp-4: 16px; --sp-6: 24px; --sp-8: 32px;
  --r-sm: 4px; --r-md: 8px; --r-lg: 12px;
  --shadow-1: 0 1px 2px rgba(16,21,28,.08); --shadow-2: 0 8px 24px rgba(16,21,28,.12);
  --dur-fast: 120ms; --dur-med: 180ms; --ease: cubic-bezier(.2,.6,.3,1);
  --focus-ring: 0 0 0 2px var(--bg-surface), 0 0 0 4px var(--accent);
}

:root[data-theme="dark"] {
  --bg-canvas: #0f1319; --bg-surface: #161b23; --bg-sunken: #0b0e13;
  --border-subtle: #232a34; --border-strong: #333d4a;
  --text-primary: #e8edf3; --text-secondary: #a3aebc; --text-tertiary: #6f7b8a;
  --accent: #4c9aff; --accent-hover: #6aacff; --accent-subtle: #12233a;
  --ok: #35b877; --warn: #d9a441; --danger: #e05c4d;
}
```

Rules: no component defines a raw colour; theme switching only redefines tokens; every text/background pair is contrast-checked and recorded in the design docs.

---

## 6. Component library

Built once, used everywhere. Each component ships with states (default/hover/focus/active/disabled/loading/error), keyboard behaviour, and a dark-theme check.

Layout: app shell, icon nav rail, top bar (site/context switcher, search, alerts, user), page header with actions, content grid.
Data: data table (sticky header, sort, filter chips, column visibility, virtualised rows above ~200 entries), stat tile, sparkline, time-series chart, status dot/badge, empty state, skeleton loader.
Input: text/number/IP field with inline validation, select, combobox, toggle, radio card, slider, tag input, form section with description column.
Overlay: side drawer (primary detail surface), modal (destructive confirms only), popover, tooltip, toast.
Feedback: inline alert, banner, progress, diff/preview block (used for "what this rule will do").

Charting: one small library or hand-rolled SVG primitives — decided by the size budget (§9). No chart library that costs more than ~30 KB gzipped.

---

## 7. Front-end architecture (phase 2)

- **Framework:** Vue 3 + Vite, or Svelte. Both produce small bundles and suit a single-developer effort; Vue's ecosystem and template model make incremental migration from `.asp` markup easier. Decide at M1 with a spike measuring real bundle size for the Clients screen in both.
- **Routing:** hash-based client routing inside one served HTML file, so `httpd` needs no rewrite rules.
- **State:** a thin store per domain (clients, networks, wireless, system) with polling (adaptive interval, backoff on error, pause when the tab is hidden — a router UI left open should not hammer the device).
- **API layer:** one typed client wrapping `appGet.cgi` / `applyapp.cgi`, with token handling, session-expiry detection and a single error surface. All legacy quirks (odd JSON, string booleans, locale strings) are normalised here, not in components.
- **Legacy bridge:** unmigrated pages render inside the shell in an iframe, styled by the phase-1 reskin, so navigation stays coherent throughout the migration.
- **i18n:** reuse the existing language dictionaries rather than re-translating. **[VERIFY]** their format and load mechanism.

---

## 8. Build and delivery

- Node/Vite build runs in CI, not on the router. Output is a small set of hashed, gzip-precompressed assets placed into the firmware's `www` directory.
- The build list in `www/Makefile` (or whatever the actual mechanism is — **[VERIFY]**) is updated so new assets ship and stale ones do not.
- Assets are pre-gzipped and served as `.gz` sidecars if `httpd` supports it; otherwise compression is dropped and the size budget tightens accordingly.
- Fonts subsetted to Latin + the character sets of shipped translations; icons as a single inline SVG sprite, no icon font.
- CI gates: bundle-size budget, `eslint`/`stylelint`, unit tests for the API layer, a11y checks (axe) on the component library, and a firmware-image-size check.

---

## 9. Budgets

| Budget | Target |
|---|---|
| Initial JS (gzipped) | ≤ 150 KB |
| Initial CSS (gzipped) | ≤ 30 KB |
| Fonts | ≤ 60 KB total (one variable font, subsetted) |
| Total new UI payload in image | ≤ 600 KB uncompressed on flash |
| Time to interactive on a LAN client | ≤ 1.0 s |
| Firmware image growth vs. baseline | ≤ 0 KB net — offset by removing legacy assets as pages migrate |

If the image-size gate fails, migration slows down but never ships a build that cannot flash.

---

## 10. Accessibility

WCAG 2.2 AA as the floor: visible focus on every interactive element, full keyboard operation (drawers trap and restore focus), labels and descriptions wired to inputs, live regions for async results, contrast checked in both themes, `prefers-reduced-motion` honoured, table semantics preserved for screen readers. The `a11y-architect` agent reviews the component library before it is adopted.

---

## 11. Migration order

| Wave | Screens | Rationale |
|---|---|---|
| 0 | Tokens + reskin of existing pages | Immediate visual lift, zero API risk |
| 1 | App shell, nav, Dashboard | Establishes the frame everything else lives in |
| 2 | **Networks / VLAN manager** | New feature, new stack, no legacy behaviour to preserve |
| 3 | Clients (+ drawer, blocks, static leases) | Highest-traffic screen |
| 4 | Wireless, incl. per-SSID VLAN binding | Completes the VLAN story end to end |
| 5 | Firewall / policies, Topology | Dense screens that benefit most from the new components |
| 6 | Diagnostics, System, Scripts/JFFS | Long tail; retire legacy assets as each lands |

Every wave ships independently and leaves the device fully functional.

---

## 12. Risks

| Risk | Mitigation |
|---|---|
| Firmware image overflow | Size gate in CI from day one; retire legacy assets in step with migration |
| `httpd` cannot serve modern assets (MIME, compression, caching) | Resolve in M0 spike; adjust the pipeline before writing UI code |
| Breaking the ASUS mobile app or AiMesh flows | Endpoints are additive-only; regression-test the app against each build |
| Model/theme variants (`tuf`, `rog`) diverge | Themes become token overrides, not forked stylesheets |
| Upstream merges conflict with UI changes | Keep new UI in new files; touch legacy pages minimally; rebase per upstream release |
| Solo-maintainer bandwidth | Wave-based delivery; each wave is independently valuable and revertible |

---

## 13. Legal boundary

UniFi is a reference for interaction patterns and visual discipline — layout conventions, density, dark-first neutrals, drawer-based detail views. Do **not** copy Ubiquiti's assets, icon set, fonts, illustrations, or CSS, and do not use their trademarks or imply affiliation. Everything shipped is original work or permissively licensed (fonts under OFL, icons under MIT or equivalent), with licences recorded in the repo.

---

## 14. Deliverables per milestone

| Milestone | Deliverable |
|---|---|
| M0 | Source and platform findings: `www` build mechanism, `httpd` MIME/compression behaviour, i18n format, image size headroom; **reconciliation of §15's page inventory against the real `www` file list, including per-model variants**, and the confirmed hook/endpoint names behind §16 |
| M1 | Token file + framework spike with measured bundle sizes; framework decision recorded |
| M2 | Reskin of existing pages shipped and flashed |
| M3 | App shell + Dashboard + API layer + component library v1 |
| M4 | VLAN manager screens (paired with the VLAN plan's M5) |
| M5 | Clients + Wireless |
| M6 | Firewall, Topology, Diagnostics, System; legacy assets retired; final size and a11y report |

---

## 15. Complete screen inventory

Two rules govern this table:

1. **Nothing is orphaned.** Every legacy page either has a destination screen, is explicitly retired, or stays reachable through the legacy bridge (§7) until it has one. A page that nobody has mapped is a page that silently disappears in a migration.
2. **Legacy filenames are unverified.** The names below are from the known ASUSWRT/Merlin structure and are marked **[VERIFY]** as a group — reconciling them against the actual `release/src/router/www` file list, including per-model variants, is an M0 deliverable. Expect the real list to contain pages not named here (model-specific, carrier-specific, and hidden diagnostic pages).

### 15.1 Legacy pages → new screens

| New screen | Replaces (legacy, **[VERIFY]**) | Wave |
|---|---|---|
| **Dashboard** | `index.asp` (Network Map), CPU/RAM widgets, WAN status blocks | 1 |
| **Topology** | Network Map client/node views, AiMesh node pages | 5 |
| **Clients** | Network Map client list, `Main_DHCPStatus_Content.asp`, client blocking/time-scheduling UIs | 3 |
| **Client detail (drawer)** | Per-client dialogs in Network Map, static lease editor in `Advanced_DHCP_Content.asp`, parental-control entries | 3 |
| **Networks / VLANs** | `Advanced_LAN_Content.asp`, `Advanced_DHCP_Content.asp`, `Advanced_IPTV_Content.asp`, `Advanced_SwitchCtrl_Content.asp`, `Guest_network.asp` | 2 |
| **Routing** | `Advanced_GWStaticRoute_Content.asp`, policy-routing tabs inside the VPN pages | 5 |
| **Wireless** | `Advanced_Wireless_Content.asp`, `Advanced_WAdvanced_Content.asp`, `Advanced_WMode_Content.asp`, `Advanced_ACL_Content.asp`, `Advanced_WSecurity_Content.asp`, `Advanced_WWPS_Content.asp` | 4 |
| **Guest access** | `Guest_network.asp` (guest half), guest scheduling | 4 |
| **Internet / WAN** | `Advanced_WAN_Content.asp`, `Advanced_WANPort_Content.asp` (Dual WAN), `Advanced_Modem_Content.asp`, `Advanced_ASUSDDNS_Content.asp`, NAT pass-through | 5 |
| **IPv6** | `Advanced_IPv6_Content.asp`, `Main_IPV6Status_Content.asp` | 5 |
| **Port forwarding & NAT** | `Advanced_VirtualServer_Content.asp`, `Advanced_PortTrigger_Content.asp`, `Advanced_Exposed_Content.asp` (DMZ), UPnP settings | 5 |
| **Firewall / Policies** | `Advanced_Firewall_Content.asp`, `Advanced_IPv6Firewall_Content.asp`, `Advanced_URLFilter_Content.asp`, `Advanced_KeywordFilter_Content.asp`, network-services filter | 5 |
| **DNS** | DNS fields scattered across WAN/LAN pages, `DNSFilter.asp`, DoT settings | 2 (with VLAN work) |
| **VPN** | `Advanced_VPNClient_Content.asp`, `Advanced_VPNServer_Content.asp`, OpenVPN/WireGuard/IPsec tabs, VPN status pages, Instant Guard | 5 |
| **Protection** | AiProtection pages (`AiProtection_HomeProtection.asp`, malicious-site blocking, IPS, infected-device prevention), parental controls | 5 |
| **QoS & Traffic** | `QoS_EZQoS.asp`, bandwidth limiter, bandwidth monitor, `Main_TrafficMonitor_realtime.asp`, Traffic Analyzer, web history | 5 |
| **USB & Storage** | `Advanced_AiDisk_samba.asp`, media server, printer server, Time Machine, 3G/4G modem, `APP_Installation.asp` (Download Master / Entware entry points) | 6 |
| **Cloud & integrations** | AiCloud pages (`cloud_main.asp`, sync, settings), Alexa/IFTTT, DDNS-adjacent cloud features | 6 |
| **Diagnostics** | `Main_Analysis_Content.asp` (ping/trace/nslookup), `Main_Netstat_Content.asp`, `Main_RouteStatus_Content.asp`, WOL, spectrum/site survey, `Tools_Sysinfo.asp` | 6 |
| **Logs** | `Main_LogStatus_Content.asp`, wireless log, connection log, VPN logs | 6 |
| **System** | `Advanced_System_Content.asp`, `Advanced_OperationMode_Content.asp`, `Advanced_FirmwareUpgrade_Content.asp`, `Advanced_SettingBackup_Content.asp`, `Tools_OtherSettings.asp`, JFFS/scripts, feedback/privacy | 6 |
| **AiMesh** | AiMesh node management, backhaul settings, node detail | 5 |

Pages deliberately **retired** rather than migrated (decision required at M0, each with a one-line rationale in the findings doc): feedback/telemetry submission, any page whose feature is unavailable on the XT8, and duplicate entry points that exist only for menu navigation.

### 15.2 New screens introduced by the VLAN plan

| Screen | Source | Wave | Notes |
|---|---|---|---|
| **Networks list** | VLAN core | 2 | Segments with subnet, client count, policy summary, isolation state, health |
| **Network editor (drawer)** | VLAN core | 2 | Subnet, DHCP, DNS, IPv6, port and SSID membership; live plain-language summary of what this segment can reach |
| **Policy editor** | VLAN §4.3, §5 | 2 | Inter-VLAN and management-plane rules as sentences, with an expert view showing the generated rule set before apply |
| **Pending change banner + diff** | VLAN §12.1 | 2 | Global chrome, not a page: countdown, *Keep changes* / *Revert now*, and the diff being confirmed |
| **Rollback history** | VLAN §12.1 | 2 | Past applies, who/when, confirmed or auto-reverted, with the reverted diff retained for inspection |
| **Onboarding queue** | VLAN §12.2 | 3 | Devices sitting in quarantine, with classification evidence (OUI, DHCP fingerprint, hostname) and one-click assignment |
| **Classification rules** | VLAN §12.2 | 3 | Ordered rule list with test-against-current-clients preview |
| **DNS policy** | VLAN §12.3 | 2–3 | Per-VLAN resolver choice, DoT/DoH config, forced-DNS toggle |
| **Blocklists** | VLAN §12.3 | 3 | Subscriptions, refresh status, entry count against the measured memory ceiling, manual allow/deny |
| **DNS insights** | VLAN §12.3 | 3 | Top talkers, top blocked, per-client drill-down; off by default with the privacy statement shown when enabled |
| **Metrics & tokens** | VLAN §12.4 | 4 | `/metrics` enable/disable, scoped token issue and revoke, exposure summary (which interfaces can reach it) |
| **Alert channels** | VLAN §12.4 | 4 | Webhook/ntfy/Telegram/email config, per-event subscription, test-send, de-duplication window |
| **Alerts feed** | VLAN §12.4 | 4 | Chronological events with acknowledge/mute; feeds the top-bar alert indicator |
| **Internet quality** | VLAN §12.5 | 4 | Latency/jitter/loss timeline per anchor, bufferbloat history, WAN event annotations, ISP-report export |
| **VPN policy routing** | VLAN §12.6 | 5 | VLAN→tunnel mapping, per-client overrides, kill-switch state with last verification time, split-tunnel rules |
| **Guest access** | VLAN §12.7 | 4 | Voucher/expiring-PSK issue, active guests with time remaining, per-guest caps, SSID schedule |

### 15.3 Global chrome (present on every screen)

- Top bar: context/site name, global search (clients, networks, rules, settings), alert indicator, session/user menu.
- **Pending-change banner** (§12.1) — outranks everything else on the page when armed; never dismissible without a decision.
- Connectivity indicator: the UI must distinguish "the router is applying a change" from "I have lost contact with the router", because during VLAN work the second one happens constantly and looks like the first.
- Toast region for command results; skeleton loaders for first paint; a single error surface for API failures.

---

## 16. Wiring

### 16.1 Conventions

- **Reads** go through `appGet.cgi?hook=<name>()` where a hook exists, otherwise through a new additive hook in the feature namespace. **[VERIFY]** the exact hook names and response shapes at M0 — the names below are the intended targets, not confirmed strings.
- **Writes** go through the existing apply endpoint (`applyapp.cgi` / `apply.cgi`) with `action_mode=apply` and an `rc_service` naming the services to restart, carrying the session token and passing the existing referer/host checks.
- **No new auth path.** The only exception is the read-only metrics token (§12.4), which is scoped, revocable, and never accepted on the UI endpoints.
- **Server-side validation is authoritative.** UI validation is a convenience; the applier re-validates everything and rejects malformed input regardless of what the UI sent.
- **Normalisation happens in the API layer**, not in components: string booleans, locale strings, odd list encodings, and per-model field presence are all resolved before data reaches a screen.

### 16.2 Per-screen wiring

| Screen | Reads | Writes / restart action | Cadence |
|---|---|---|---|
| Dashboard | `cpu_ram_status`, `netdev` throughput, WAN status, client count, alert summary | — | 3–5 s while visible; paused when tab hidden |
| Clients | client list, DHCP leases, wireless station list (per radio/VAP), per-client throughput | client rename, block, static lease, VLAN assignment → DHCP/firewall restart | 5 s |
| Client detail | as above + connection history, per-client DNS stats (if enabled) | as above | on open, then 5 s |
| Topology | AiMesh node list, backhaul quality, client-to-node mapping | node config actions | 10 s |
| Networks list | `vlan_config` + `vlan_state` (bridge up, client counts, applied-at, last error) | — | 5 s |
| Network editor | `vlan_config`, available VAPs, available ports, IPv6 prefix availability | `restart_vlan` (bridges, DHCP/DNS, firewall) | on open / on save |
| Policy editor | `vlan_config` rules, generated-ruleset preview | `restart_vlan` | on save |
| Pending change banner | pending-change status: armed, deadline (server-authoritative), diff summary | confirm / revert-now actions | 1 s while armed |
| Rollback history | apply log with outcomes and diffs | — | on open |
| Onboarding queue | quarantine device list with classification evidence | assign device → VLAN membership + DHCP/firewall restart | 10 s |
| Classification rules | rule list, dry-run result against current clients | rule create/reorder/delete → re-evaluation | on save |
| DNS policy | per-VLAN resolver config | DNS/DHCP service restart | on save |
| Blocklists | subscription list, refresh timestamps, entry counts, memory usage | subscribe / refresh / manual entries → list rebuild | on open; refresh status 5 s during a rebuild |
| DNS insights | aggregated query stats per VLAN and client | enable/disable logging, retention | 10 s |
| Wireless | per-radio and per-VAP config, station counts | wireless restart (**disruptive — warn before apply**) | on open |
| Guest access | active guests with expiry, voucher/PSK list, caps, schedule | issue/revoke credential, schedule change | 10 s |
| Internet / WAN | WAN state, dual-WAN status, DDNS state | WAN restart (**disruptive**) | 5 s |
| IPv6 | IPv6 config and delegated prefix state | IPv6/WAN restart | on open |
| Port forwarding & NAT | rule lists, UPnP mappings | firewall restart | on open |
| Firewall / Policies | rule lists (v4 and v6), generated ruleset | firewall restart | on open |
| VPN + policy routing | tunnel configs, tunnel state, kill-switch state and last verification, VLAN→tunnel map | tunnel up/down, routing policy → firewall + routing restart | 5 s |
| Protection | AiProtection state and event counts | feature toggles | on open |
| QoS & Traffic | QoS config, live per-client bandwidth, historical traffic | QoS restart | 3–5 s live, on open for history |
| Metrics & tokens | metrics config, token list with last-used, exposure summary | enable/disable, issue/revoke token | on open |
| Alert channels / feed | channel config, event feed | channel CRUD, test-send, acknowledge | feed 10 s |
| Internet quality | probe history, bufferbloat results, WAN events | probe config, run-test-now | 30 s; live during a running test |
| USB & Storage | mount state, share config, service state | service restarts | on open |
| Diagnostics | tool output (ping/trace/netstat/route), sysinfo | run-tool actions (long-running, streamed or polled) | per run |
| Logs | syslog tail, per-service logs | clear/rotate | 5 s while visible |
| System | firmware state, config backup, JFFS/scripts, operation mode | firmware upgrade, backup/restore, reboot (**all disruptive**) | on open |

### 16.3 The apply pipeline

Every network-affecting change follows one path, and the UI never invents a shortcut around it:

```
validate (server)  →  preview diff  →  operator confirms intent
      →  apply + snapshot  →  arm rollback timer (§12.1)
      →  banner countdown  →  operator confirms success  →  snapshot retired
                          ↘  silence / lost contact  →  automatic revert
```

Wiring requirements:

- The countdown deadline comes from the server, not from a client-side timer. A reloaded page, a second browser tab, or a phone picking up mid-window must all see the same remaining time.
- The confirm action requires a live authenticated session. Rendering the page is not proof the operator still has working access.
- If the browser loses contact after an apply, the UI shows *waiting for the router to come back* with the deadline, not a generic error — the operator needs to know that doing nothing is a safe outcome.
- After a revert, the next page load surfaces what was rolled back and why, rather than quietly showing the old configuration.

### 16.4 Optimistic UI rules

- **Never optimistic** for anything that changes networking: VLANs, firewall, wireless, WAN, routing, VPN. The UI shows the requested state as *pending* and only shows it as current when the router confirms it.
- **Optimistic is fine** for local, non-network state: renaming a client, acknowledging an alert, toggling a UI preference.
- Any screen whose apply triggers a wireless or WAN restart warns before applying, naming what will drop.

### 16.5 Polling and lifecycle

One polling manager owns all periodic reads: per-screen cadence, pause on hidden tab, exponential backoff on error, immediate refetch on focus, and a global "router unreachable" state shared by every screen so ten components do not each render their own error. Long-running operations (firmware upgrade, reboot, blocklist rebuild, bufferbloat test) get progress plus reconnect detection rather than a spinner that lies.

### 16.6 New endpoints (additive)

`vlan_config`, `vlan_state`, `pending_change` (+ confirm / revert), `rollback_history`, `onboarding_queue`, `classification_rules`, `dns_policy`, `dns_blocklists`, `dns_stats`, `metrics_config`, `metrics_tokens`, `alert_channels`, `alert_feed`, `netquality_history`, `vpn_policy`, `guest_credentials`.

All additive — no existing endpoint is renamed or removed, because the ASUS mobile app and AiMesh flows depend on them (§2). Each new endpoint gets the same session/CSRF treatment as the existing ones and is covered by the security review that gates the VLAN feature.
