# OneClick

One-click diagnostics collector for a Simnovator perf-test run. SSHes to the rack hosts (UE / Simnovator / Callbox / app-server), time-windows everything to the resolved test iteration, and packages a `<testcase>_diagnostics_<TS>.zip`. Auto-generates `ANALYSIS.md` (per-area status + deep checks) and `SYSTEM.md` (cross-host inventory).

Ships a Flask UI on port **8080** with a Collector / Logs / Setup workflow + an **Update** button that pulls the latest from this repo with a single click.

## Layout

```
oneclick/
├── INSTALL.md                  full install guide (start here for a new site)
├── README.md                   this file
├── collect_perf_data.sh        bash collector — main entry point
├── analyze_bundle.py           post-collection heuristics → ANALYSIS.md
├── build_system_md.py          cross-host inventory → SYSTEM.md
├── beszel_screenshot.py        Playwright capture for the Beszel dashboard (optional)
├── simnovator_screenshot.py    Playwright capture for the Simnovator GUI tabs (optional)
├── setup.conf.example          copy to setup.conf and fill in IPs + creds
├── scripts/
│   ├── install.sh              one-shot installer (apt/dnf prereqs + venv + systemd)
│   └── fetch-vendor.sh         pre-stage Chromium for offline customer installs
└── ui/
    ├── app.py                  Flask UI (Collector / Logs / Setup tabs + Update button)
    ├── perf-qa-ui.service      systemd unit (PERFQA_PORT=8080 by default)
    ├── favicon.png
    ├── logo_light.svg
    └── logo_dark.svg
```

## Install at a customer site

See **`INSTALL.md`** for the full step-by-step. Short version:

```bash
tar xzf oneclick-*.tar.gz       # or: git clone https://github.com/nikhilsimnovus/oneclick.git
cd oneclick
sudo bash scripts/install.sh
```

The installer detects apt/dnf, installs prereqs, creates the service user + FHS-aligned dirs, builds the Python venv, plants the systemd unit, and starts the service. Open `http://<collector-host>:8080/` when it's done.

## Updates

Each Flask tab has a green **↓ Update** button in the top-right corner. One click pulls the latest tarball from this repo's main branch and re-runs `scripts/install.sh`. The service restarts automatically; the page reloads with the new code. No SSH, no manual file copies.

The Update flow uses a sudoers-whitelisted wrapper (`/usr/local/sbin/perfqa-update`) so the service user can self-update without being granted general root.

## What's in a bundle

```
<testcase>_diagnostics_<YYYYMMDD_HHMMSS>.zip   (~2-5 MB)
├── ANALYSIS.md            per-area status + cross-host + perf-tuning checks
├── SYSTEM.md              cross-host inventory (CPU / RAM / kernel / SDR / iperf3 ver)
├── <testcase>.testcase.json   re-importable via POST /v2/testcases/import
├── MANIFEST.txt + collect.log
├── ue/        system + cpu + net + UESIM logs + iperf logs + heat CSVs
├── simnovator/ container ps + stats + health + per-container logs + Beszel screenshot
│              ├── container_health.txt + systemd_units.txt   (quadlet hosts)
│              ├── journal/   per-unit systemd journal (windowed; quadlet hosts)
│              └── native_logs/  `simnovator logs` tar + CLI help (if CLI present)
├── callbox/   enb/mme/ims.cfg + sensors + amari monitor + heat CSVs
├── app_server/ network info
└── rest_api/  test definition + statistics + logs export + GUI screenshots
```

On hosts running the newer **podman quadlet** stack the collector auto-detects
systemd-managed containers and additionally captures each unit's systemd journal
(the authoritative log — survives restarts, records health-check + dependency
ordering) plus container health status. It degrades cleanly on older
podman-compose hosts. No configuration needed; `SIM_NATIVE_LOGS=0` opts out of
the native-CLI tar grab.

## CLI use

```bash
/opt/perf-qa/collect_perf_data.sh /opt/perf-qa/setup.conf
# → /var/lib/perf-qa/bundles/<testcase>_diagnostics_<TS>.zip
```
