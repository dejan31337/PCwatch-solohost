# PC Watch — SoloHost starter project

A read-only Windows 11 PC monitoring dashboard intended to run as a SoloHost app. The web app runs in a container; a separate Windows agent reads host metrics and posts them to the app over localhost.

## Important: not install-ready until you publish the image

SoloHost packages reference a **pre-built public image**. They do not build your source code. Before installing the package, replace `YOUR_GITHUB_USERNAME` in `docker-compose.yml`, build the image, and publish it to GitHub Container Registry (GHCR). The included source is a starter implementation, not an already-published service.

The package follows the SoloHost v0 contract: `docker-compose.yml` + `config_options.yml`, one primary UI service, one loopback-published UI port, and all Compose variables declared in the config form.

## Architecture

- `app/`: FastAPI web dashboard and authenticated metrics endpoint.
- `agent/`: Windows agent that reads CPU, RAM, disk, network and process metrics.
- `docker-compose.yml`: SoloHost package template.
- `config_options.yml`: SoloHost install form.

The container cannot reliably inspect Windows host processes/sensors itself. SoloHost intentionally blocks privileged mode, host PID namespaces, devices, and host bind mounts. The Windows agent is therefore needed for actual host metrics.

## 1. Make a strong token

Generate one in PowerShell:
```powershell
python -c "import secrets; print(secrets.token_urlsafe(32))"
```
Use this value as `APP_TOKEN` in SoloHost, and as `PCWATCH_TOKEN` for the Windows agent. Do not commit it to GitHub or paste it into public issue trackers.

## 2. Build and publish the image

Install Docker Desktop and sign in to GitHub Container Registry. Create a GitHub repository for this project and upload the `app/` folder. In PowerShell from the project folder:
```powershell
docker login ghcr.io -u YOUR_GITHUB_USERNAME
docker build -t ghcr.io/YOUR_GITHUB_USERNAME/pcwatch:0.1.0 .\app
docker push ghcr.io/YOUR_GITHUB_USERNAME/pcwatch:0.1.0
```
Ensure the GHCR package is public/pullable. Change the image reference in `docker-compose.yml` to your actual lower-case GitHub username and package path. Keep a versioned tag.

## 3. Install in SoloHost

1. Open Pi Desktop 0.6.4 → SoloHost.
2. Create/publish your app using the contents of `docker-compose.yml` and `config_options.yml`.
3. Set `APP_TOKEN` to the random token you generated.
4. Install/start the app and open its embedded UI.

The published UI port is bound to `127.0.0.1:18642` on purpose. Do not change it to `0.0.0.0` or forward it on your router. Use the supported Pi Desktop 0.6.4 remote connection path for remote access; do not assume that direct public access to this localhost port works.

## 4. Install the Windows agent

On the same Windows PC that runs Pi Desktop, open PowerShell:
```powershell
cd path\to\pcwatch\agent
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
$env:PCWATCH_URL = "http://127.0.0.1:18642/api/metrics"
$env:PCWATCH_TOKEN = "PASTE_THE_SAME_TOKEN_HERE"
python .\agent.py
```
Leave the terminal open while testing. Open the dashboard, paste the same token in the token box, and click Connect. Metrics should appear after the agent's first successful post.

If PowerShell blocks activation, run the agent with the virtual environment's Python directly:
```powershell
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe .\agent.py
```
Set the environment variables in that same terminal before launching the script.

## Temperature support

CPU temperature is not consistently exposed by Windows through `psutil`. The agent returns `Unavailable` unless it can query the WMI namespace exposed by LibreHardwareMonitor. To enable it, run LibreHardwareMonitor with its sensor WMI support enabled, then install the optional WMI Python package:
```powershell
.\.venv\Scripts\python.exe -m pip install WMI
```
The sensor namespace may require elevated access and differs by hardware/driver. Do not run the monitoring agent as administrator unless needed and understood.

## Security notes

- The dashboard requires a shared token for metrics ingestion and the status API.
- The UI port is loopback-only; use Pi Desktop's supported remote access, not public router port forwarding.
- Anyone with the token can view the metrics and process names. Do not share it.
- This starter is read-only: no shell execution, file browsing, or remote shutdown.
- Use a dedicated Windows account where practical. Keep Python/Docker updated.
- The dashboard has no built-in user account system, HTTPS terminates outside this app, and its token is not a substitute for a proper multi-user auth system. Use only through trusted Pi Desktop remote access.
- Before publishing for others, add rate limiting, stronger session handling, audit/security review, and a documented token rotation flow.

## Troubleshooting

- `401 Unauthorized`: token mismatch between SoloHost `APP_TOKEN`, agent `PCWATCH_TOKEN`, and dashboard token.
- No metrics: verify the agent runs on the same PC and `http://127.0.0.1:18642/health` opens locally.
- App will not install: make sure the image is public, versioned, and your actual username replaced the placeholder.
- Blank embedded UI: check the SoloHost app logs and ensure the service has `pi.ui.primary: "true"` and exactly one loopback-published UI port.
- Windows temperature is missing: expected on many systems; see temperature section.
