> [中文文档](pi-agent.md)

# Agent workbench (Pi / Prime)

The Agent page supports personal CADD sessions powered by either Pi Agent Core or Prime Agent. Each WA-DD user can only see their own sessions, context, and model configuration; both runtimes share one encrypted model configuration.

![Agent / Pi page: manage sessions and model configuration on the left, execute CADD workflows in the current project through conversation on the right](images/agent-pi-overview.jpg)

On the left side of the page you can create, rename, or delete personal sessions, and open model configuration; the conversation on the right combines the proteins, ligands, pockets, and task results in the current project, asks for necessary inputs as needed, and then calls the allowed tools. Tool permissions match the current user and cannot access other users' projects or assets.

The first time you open the page, you will be asked to fill in the OpenAI-compatible model endpoint, model name, and API Key for the current account. Each user's API Key is encrypted and stored independently, and the page never echoes it back.

Pi and Prime only expose the controlled `wa_dd_api` tool: it reads projects, assets, and tasks within the current user's permissions, and submits allowed preparation, docking, or FEP API requests. Prime's built-in IPython and shell tools are disabled, and the Prime worker has no host directory, Docker socket, or other users' tokens.

Pi reuses the existing `WA_DD_DATA_HOST_DIR` mount; encrypted configuration and session context mirrors are persistently saved to `/data/pi` inside the container, which corresponds to `${WA_DD_DATA_HOST_DIR}/pi` on the host. PostgreSQL remains the authoritative record for permission checks and queries.

On first startup, a read-only deployment encryption master key is generated in this persistent directory. Typically no environment variable needs to be set; if you want it managed by a key management system, you may optionally set `WA_DD_PI_CONFIG_ENCRYPTION_KEY` to override the auto-generated key. It is not the user's model API Key.

Build the Pi worker:

```bash
./build_image.sh --profile amd --component pi-agent
```
