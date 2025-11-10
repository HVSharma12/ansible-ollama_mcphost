# Cockpit Setup Guide

This directory now ships **two** Cockpit experiences powered by `mcphost`. Both installers configure the required packages, install the web assets, and restart `cockpit.socket`.

| Playbook | Purpose |
|----------|---------|
| `setup-cockpit-mcphost-deploy.yml` | New UI that mirrors every `deploy.sh` flag and exposes an `mcphost` chat console. |
| `setup-cockpit-risu-standalone.yml` | Legacy RISU-only diagnostics that runs the bundled RISU agent via a generated `mcphost` script. |

Use the sections below to pick the workflow you need.

---

## Deploy + Chat UI (`setup-cockpit-mcphost-deploy.yml`)

### Install

```bash
cd ansible-cockpit
ansible-playbook setup-cockpit-mcphost-deploy.yml -K
```

What the playbook does:

- Installs Cockpit, firewalld, `ansible-core`, `mcphost`, Python, and helper CLI tools.
- Copies `deploy.sh`, `deploy.yml`, `ansible.cfg`, and the entire `roles/ansible-ollama_mcphost/` tree into `/usr/local/share/ansible-ollama_mcphost`.
- Writes `/usr/local/bin/ansible-ollama-deploy` which is a minimal wrapper around the bundled `deploy.sh`.
- Deploys the Cockpit extension in `/usr/share/cockpit/ansible-ollama` (HTML/JS/CSS + `config.json`).

### Using the UI

1. Open `https://<host>:9090` and sign in (Cockpit default port).
2. Click **Ansible Ollama** on the left sidebar.
3. The **Deployment settings** form maps 1:1 to `deploy.sh` flags:
   - Model tag, GPU toggle/runtime
   - Auto-discover flag and multi-select for MCP directories
   - API key env var vs inline value (inline value is masked in the displayed command preview)
   - System prompt file or multi-line inline prompt
4. Press **Run deploy.sh** to stream the Ansible output. The `Stop run` button sends a cancel signal by calling `close('cancelled')` on the spawned process.
5. Use **Refresh servers** to rescan `roles/…/mcp_servers` and `/root/mcp_servers`.
6. Use the **mcphost chat** section to run `mcphost -p "<prompt>"` (with optional `-m override`). Output is streamed with ANSI sequences stripped.

### Customising paths/defaults

- Override the bundle root or Cockpit path: `ansible-playbook setup-cockpit-mcphost-deploy.yml -e "deploy_bundle_root=/opt/mcphost-bundle" -e "cockpit_root=/usr/share/cockpit/ollama-ui"`.
- Change the default model placeholder: `-e "default_model=qwen2.5:7b"`.
- `user_mcp_dir` defaults to `/root/mcp_servers` but can be pointed elsewhere with `-e`.

### Validation checklist

```bash
sudo ls /usr/local/share/ansible-ollama_mcphost
sudo /usr/local/bin/ansible-ollama-deploy --help
sudo ls /usr/share/cockpit/ansible-ollama
sudo systemctl status cockpit.socket
```

If the UI fails to load, watch the browser console and verify `/usr/share/cockpit/ansible-ollama/app.js` exists.

---

## RISU-only Diagnostics (`setup-cockpit-risu-standalone.yml`)

This is the original extension and is still useful when you only need the RISU MCP agent.

### Install

```bash
cd ansible-cockpit
ansible-playbook setup-cockpit-risu-standalone.yml -K
```

### Workflow

1. Browse to `https://<host>:9090` → **Ansible Playbook**.
2. Choose **RISU Agent diagnostics** from the dropdown.
3. Optional: enter plugin filters.
4. Click **Run diagnostics** – the UI writes a temporary `mcphost script …` file that runs RISU, captures JSON, and asks the model for a structured summary.

The RISU UI bypasses Ansible entirely; everything runs through the generated script and `mcphost script <file> --compact`.

---

## Uninstalling Either Extension

```bash
cd ansible-cockpit
ansible-playbook uninstall-cockpit-extension.yml -K
```

This removes `/usr/share/cockpit/ansible-ollama*` (or `ansible-playbook/` for the legacy UI) and restarts `cockpit.socket`. The deployment bundle under `/usr/local/share/ansible-ollama_mcphost` is not removed so you can reinstall quickly.

---

## Troubleshooting Cheatsheet

| Issue | Checks |
|-------|--------|
| Cockpit page missing | `sudo systemctl status cockpit.socket`, ensure port 9090 reachable, verify `/usr/share/cockpit/ansible-ollama/manifest.json`. |
| Deploy button fails instantly | `sudo -n /usr/local/bin/ansible-ollama-deploy --help` (sudoers prompt?). Ensure `ansible-core` installed. |
| Server list empty | Ensure MCP directories exist: `sudo ls /usr/local/share/ansible-ollama_mcphost/roles/ansible-ollama_mcphost/mcp_servers`. |
| mcphost chat errors | Run `mcphost -p "ping"` directly to verify config and API keys. |
| js console errors | `sudo journalctl -u cockpit` for backend issues; `sudo tail -f /var/log/cockpit/*.log` for more detail. |

---

## File Structure

```
ansible-cockpit/
├── setup-cockpit-mcphost-deploy.yml
├── setup-cockpit-risu-standalone.yml
├── uninstall-cockpit-extension.yml
├── templates/
│   ├── config.json.j2
│   ├── deploy-wrapper.sh.j2
│   └── mcphost-deploy/{index.html.j2,app.js.j2,styles.css.j2,manifest.json.j2}
└── README-SETUP.md
```
