# Cockpit mcphost Integration

This directory contains playbooks that install Cockpit extensions backed by `mcphost`. Two experiences are available:

1. **Deploy + Chat GUI (`setup-cockpit-mcphost-deploy.yml`)** – surfaces every `deploy.sh` flag in a web form and adds a streaming `mcphost` chat console.
2. **RISU Diagnostics (`setup-cockpit-risu-standalone.yml`)** – legacy extension that runs the bundled RISU MCP servers through a generated `mcphost` script.

Both playbooks install Cockpit + prerequisites, copy the required assets, and restart `cockpit.socket`.

## Quick Start

### Option A – Deploy + Chat UI

```bash
cd ansible-cockpit
ansible-playbook setup-cockpit-mcphost-deploy.yml -K
```

What you get:

- `/usr/share/cockpit/ansible-ollama` extension with `manifest.json`, `index.html`, `app.js`, and `styles.css`
- `/usr/local/share/ansible-ollama_mcphost` bundle containing `deploy.sh`, `deploy.yml`, `ansible.cfg`, and the role tree
- `/usr/local/bin/ansible-ollama-deploy` wrapper that simply runs the bundled `deploy.sh`
- `/usr/share/cockpit/ansible-ollama/config.json` describing paths for the UI

The Cockpit page (Ansible Ollama) lets you:

- Set the model tag, GPU toggle/runtime, auto-discovery flag, directory-based MCP selection, API key inputs, and system prompt fields (1:1 with `deploy.sh`)
- Stream Ansible output in a log panel and stop a run if needed
- Refresh the MCP directory list (bundled role path + `/root/mcp_servers`)
- Send ad-hoc prompts to `mcphost` with optional `--model` overrides to validate the deployment

Customize defaults by passing `-e "default_model=qwen2.5:7b"` or overriding `deploy_bundle_root`, `cockpit_root`, etc.

### Option B – RISU-only Diagnostics

```bash
cd ansible-cockpit
ansible-playbook setup-cockpit-risu-standalone.yml -K
```

This keeps the original RISU-specific UI that dynamically writes a `mcphost script …` file and streams the formatted diagnostics back into Cockpit. Use it if you only need RISU coverage without the broader deployment interface.

## Architecture (Deploy + Chat UI)

```
Cockpit GUI → deploy wrapper (ansible-ollama-deploy) → deploy.sh → ansible-playbook deploy.yml
                         ↘︎ mcphost CLI (chat panel) → ~/.mcphost.yml
```

- Cockpit JS reads `/usr/share/cockpit/ansible-ollama/config.json` to know where the bundle and MCP directories live.
- Directory discovery is done via a short Python snippet executed with `cockpit.spawn`.
- Form submissions are translated back into the exact CLI flags that `deploy.sh` understands and the wrapper runs with `sudo`.
- The chat section shells out to `mcphost -p "…"` so you can test the resulting configuration without leaving the browser.

## Uninstall

The shared uninstall playbook removes whichever extension is installed:

```bash
cd ansible-cockpit
ansible-playbook uninstall-cockpit-extension.yml -K
```

## File Structure

```
ansible-cockpit/
├── setup-cockpit-mcphost-deploy.yml   # New deploy + chat GUI
├── setup-cockpit-risu-standalone.yml  # Legacy RISU-only GUI
├── uninstall-cockpit-extension.yml    # Removes Cockpit assets
├── templates/
│   ├── config.json.j2
│   ├── deploy-wrapper.sh.j2
│   └── mcphost-deploy/{manifest.json.j2,index.html.j2,app.js.j2,styles.css.j2}
├── README-SETUP.md
└── README.md
```

Older `.md` files in this directory are kept for historical reference.
