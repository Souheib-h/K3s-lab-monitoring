# ADR additions — to append to DECISIONS.md

## ADR-011 — Ansible control node on Alpine Linux

**Date:** 2026-07-18 · **Status:** Accepted

**Context.** Phase 5 requires a dedicated control node. The initial Ubuntu VM
consumed 4 GiB RAM for a role with no heavy dependency; host RAM is the lab's
scarcest resource (31 GiB physical).

**Decision.** Rebuild the control node on Alpine Linux (1 GiB). Ansible is
installed from the Alpine repository (`apk add ansible`, full bundle: core +
community collections including community.zabbix). SSH key passphrase is
served by keychain in the shell profile.

**Consequences.** Minimal footprint, consistent with load-srv (already
Alpine). Trade-offs accepted: OpenRC quirks (services suffixed `-d`), Python
packages managed via apk/pipx, and the control node itself runs the 4.8.2
Wazuh agent (see ADR-010 amendment).

---

## ADR-012 — Ansible scope: OPNsense and hypervisor excluded

**Date:** 2026-07-19 · **Status:** Accepted

**Context.** With 13 hosts under Ansible, two machines remained candidates:
the OPNsense firewall VM and the Arch hypervisor.

**Decision.** Both are intentionally excluded from the Ansible perimeter.

- **OPNsense**: FreeBSD-based, no standard Python/SSH management path; its
  configuration is managed through its own UI/config.xml and documented
  separately. A `connection: local` placeholder in the inventory produced
  misleading SUCCESS results and was removed.
- **Hypervisor (My-pc)**: trust hierarchy. The control node holds SSH keys and
  passwordless sudo on the entire lab; granting it root on the machine that
  *runs* the lab would invert the containment model — a compromised control
  node must not yield the host. The hypervisor already runs both monitoring
  agents (installed manually in earlier phases) and is monitored like any
  other host.

**Consequences.** `ansible all` covers exactly the 13 lab hosts. Firewall and
hypervisor changes remain manual and documented.

---

## ADR-010 — Amendment: Alpine agents on 4.8.2

**Date:** 2026-07-18 · **Status:** Accepted exception

ADR-010 freezes the Wazuh repository at 4.14.6 to keep manager and agents in
lockstep. The Wazuh apk repository for Alpine stops at **4.8.2** — no 4.14
build exists. The two Alpine hosts (load-srv, ansible-srv) therefore run
agent 4.8.2 against the 4.14 manager: protocol-compatible, flagged "outdated"
in the dashboard, and **excluded from Vulnerability Detection scans**. Accepted
as-is; revisit if Wazuh publishes newer Alpine builds.
