# ludus_detonator_agent
:warning: Various parts of the code are written by Claude. Use at own risk.

An [Ludus](https://ludus.cloud) Ansible role that turns a Windows VM into a
detonation target for [**dobin/detonator**](https://github.com/dobin/detonator):
it builds [**dobin/DetonatorAgent**](https://github.com/dobin/DetonatorAgent)
from source.

## Requirements

- Ludus, and a Windows template (developed on `win11-25h2-x64-enterprise`).
- Collections: `ansible.windows`, `community.windows`, `chocolatey.chocolatey`
  (all present on a Ludus server).
- ~4 GB RAM / 2 vCPU. The .NET build is the slow part (several minutes).
- Internet from the VM at deploy time — it clones from GitHub, pulls Chocolatey packages and restores NuGet. Deploy before you enable ludus testing.

## Install

```bash
# the DIRECTORY name becomes the Ludus role name -- meta/main.yml is ignored.
# Cloning into anything else installs a role your range config cannot find.
git clone https://github.com/curi0xity/ludus_detonator_agent
ludus ansible role add -d ./ludus_detonator_agent
# after local edits -- Ludus caches roles server-side:
ludus ansible role add -d ./ludus_detonator_agent --force
```

## Minimal range config

```yaml
ludus:
  - vm_name: "{{ range_id }}-win11-edr"
    hostname: "{{ range_id }}-win11-edr"
    template: win11-25h2-x64-enterprise-template
    vlan: 10
    ip_last_octet: 21
    ram_gb: 4
    cpus: 2
    windows:
      sysprep: true
    roles:
      - ludus_detonator_agent
    role_vars:
      ludus_detonator_agent_edr: "defender"
      ludus_detonator_agent_install_autoit: true
      ludus_detonator_agent_reboot: true
```

```bash
ludus range config set -f range-config.yml
ludus range deploy
```

Verify — and probe `/api/lock/status`, **not** `/`:

```bash
curl http://10.X.10.21:8080/api/lock/status
```

`/` only returns 200 while `publish\wwwroot` exists, so it reports healthy on a
broken agent and unhealthy on a working one. `/api/lock/status` is the honest
one.

Then take a snapshot you can revert to between detonations:

```bash
ludus snapshots create clean -n <proxmox-vmid>
```

## Variables

| variable | default | notes |
|---|---|---|
| `ludus_detonator_agent_repo_url` | `https://github.com/dobin/DetonatorAgent.git` | |
| `ludus_detonator_agent_repo_version` | `83b8f3d` | **Pinned.** HEAD had broken FlaUI 5.0 / .NET 8 compatibility. Bump deliberately — the patches below are matched to this commit |
| `ludus_detonator_agent_install_dir` | `C:\Tools\DetonatorAgent` | |
| `ludus_detonator_agent_port` | `8080` | |
| `ludus_detonator_agent_edr` | `defender` | `defender` \| `fibratus` \| `logfile` \| `none`. Use `none` when a cloud plugin on the Detonator side provides the verdict |
| `ludus_detonator_agent_install_autoit` | `true` | required for `--execution-mode autoit` / `clickfix` |
| `ludus_detonator_agent_run_at_logon` | `true` | autologon + logon-triggered task. Turn off only if you do not need interactive execution |
| `ludus_detonator_agent_autologon_user` | `localuser` | |
| `ludus_detonator_agent_autologon_password` | `password` | Ludus default — **override it** |
| `ludus_detonator_agent_autologon_domain` | `''` | empty = the computer name, which is what a LOCAL account needs. Set only to log on as a domain user |
| `ludus_detonator_agent_reboot` | `true` | needed for autologon to take effect |
| `ludus_detonator_agent_open_firewall` | `true` | |
| `ludus_detonator_agent_install_rededr` | `false` | needs `..._rededr_release_url` |
| `ludus_detonator_agent_time_source_url` | `''` | a URL whose HTTP `Date` header is trustworthy. Empty = no clock healing. See [the clock trap](#traps) |

Full list with comments: [`defaults/main.yml`](defaults/main.yml).

---

## Using it with dobin/detonator

Detonator runs on a Linux host that can reach the VM's port 8080.

```bash
git clone https://github.com/dobin/detonator && cd detonator
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
```

Write `profiles_init.yaml`, import it, start the server:

```yaml
win11_defender:
  connector: Live          # Live = use the VM as-is. Proxmox = revert per run
  comment: Win11 + Defender
  vm_ip: 10.X.10.21       # the Ludus VM
  port: 8080
```

No `edr_*` key here on purpose: with `ludus_detonator_agent_edr: defender` the
verdict comes from the agent's own `/api/logs/edr`, not from a cloud plugin.
Declaring `edr_defender:` would be an *unclaimed* key — see the rule below.

```bash
.venv/bin/python migrate_profiles_yaml.py
.venv/bin/python -m detonator both      # API :8000, web UI :5000
.venv/bin/python -m detonatorcmd --profile win11_defender ./payload.exe
```

> **Put EDR config under `data:`.** Upstream's `profiles_init.yaml.sample`
> shows `edr_elastic:` at the profile's top level for the `Live` profiles, but
> `migrate_profiles_yaml.py` reads `profile_config.get('data', {})` — so a
> top-level block is silently ignored, no alerts are collected, and the run
> reports `not_detected`. That is indistinguishable from real evasion. The
> `test_proxmox` entry in the same sample has it right.

### Clean state between detonations

A payload that ran changes the VM. Without a revert, run N+1 is measured on a
dirty machine. Two options:

- **`connector: Proxmox`** — Detonator reverts the VM to a named snapshot at
  the end of every run. Needs `detonatorapi/connectors/proxmox.yaml` (copy the
  `.sample`) and a Proxmox API token with **privilege separation OFF**, or
  snapshot operations are refused.

  ```yaml
  win11_defender:
    connector: Proxmox
    vm_ip: 10.X.10.21
    port: 8080
    data:
      proxmox_id: 162
      proxmox_snapshot: clean
  ```

  Proxmox VM IDs **change when you rebuild a range**. Resolve `proxmox_id` from
  the live VM name rather than hand-maintaining it.

- **`ludus testing start` / `stop`** — reverts *every* VM in the range and
  blocks internet. Good as an outer containment envelope, wrong for the
  per-detonation loop: see [the clock trap](#traps).

### Cloud-console EDRs

For an EDR whose verdict lives in a vendor console (Defender for Endpoint,
Elastic, CrowdStrike…), set `ludus_detonator_agent_edr: "none"` and let a
Detonator cloud plugin collect the alerts instead. Routing is by the presence of
an `edr_<vendor>` key in the profile's `data`. See `doc/implement-edr.md` in
the Detonator repo.

One rule worth copying: **never declare an `edr_*` key whose plugin does not
exist.** The collection block is skipped, zero alerts come back, and the verdict
is `not_detected` — identical to a real evasion.

A worked Elastic Defend profile, as upstream's plugin actually wants it:

```yaml
win11_elastic:
  connector: Live
  vm_ip: 10.X.10.21
  port: 8080
  data:
    edr_elastic:
      elastic_url: "https://10.X.20.1:9200"
      elastic_apikey: "<base64 'encoded' field from POST /_security/api_key>"
      hostname: "myrange-win11-edr"  # must match the agent's host.name in Elastic
```

Upstream's `InitializeClient` requires exactly `elastic_url`, `elastic_apikey`
and `hostname`. It takes an **API key, not user/password** — if you are copying
a profile from a fork that uses `elastic_user`/`elastic_password`, the plugin
will refuse to initialise. Mint a key with:

```bash
curl -sk -u elastic:<pass> -XPOST "https://<elastic>:9200/_security/api_key" \
  -H 'Content-Type: application/json' -d '{"name":"detonator"}'
```

`hostname` is matched against the host as Elastic knows it, which is the Windows
computer name — not the Ludus VM name, if you have renamed anything.

---

## Patches this role applies

| file | what | why |
|---|---|---|
| `WindowsExecutionServiceClickfix.cs` | `AutoItX.Send(cmd)` → `Send(cmd, 1)` | Default send mode treats `! ^ + # { }` as modifier keys, so any real command containing them is silently mangled into keystrokes. Flag 1 sends raw |
| `WindowsExecutionServiceAutoit.cs` | log the dialog probe before the early return | Distinguishes "no dialog at all" from "a dialog we don't recognise". Win11's SmartScreen prompt is XAML, so `[CLASS:#32770]` cannot see it |
| `WindowsExecutionServiceAutoit.cs` | Win+E → `AutoItX.Run("explorer.exe \"<file>\"")` | **This one changed results.** The upstream keystroke sequence (Win+E, wait, Alt+D, type path, Enter, type name, Enter) assumes a painted, focused Explorer. On 2 vCPU it is not — the active window was still `Home - File Explorer` twelve seconds in, so `--execution-mode autoit` wrote the payload and never ran it, while still producing a verdict. ShellExecute via the running shell keeps `explorer.exe` as the parent and still does the zone check, so MOTW stays measurable |
| `WindowsExecutionServiceAutoit.cs` | an unconfirmed PID is no longer a failure | The agent looks the process up **by name** ~1.5s after launch, so anything that exits fast returns pid 0 and upstream aborts the submission before any alert is collected. Every short-lived benign control hit this |
| `ExecuteController.cs` | `execution_mode` suffix `:motw` writes `Zone.Identifier` | Nothing in the Detonator path marks a dropped file, so every detonation tested the no-MOTW case only. The suffix rides on a string that is already plumbed server→agent and recorded in the report |

If you want stock upstream behaviour, delete the patch tasks from
`tasks/main.yml`. Keep the first and fourth at minimum; without them `autoit`
mode produces verdicts for payloads that never ran.

## License

MIT. DetonatorAgent, Detonator and RedEdr are dobin's, under their own licenses.

## Acknowledgments

Thanks to [dobin](https://github.com/dobin) and [Erik Hunstad](https://ludus.cloud/about) for making testing of payloads easier.
