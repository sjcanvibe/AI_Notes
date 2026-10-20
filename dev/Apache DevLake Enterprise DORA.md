# Project Handoff: Apache DevLake Enterprise DORA & 45-Org Provisioning

Please read `/home/ngsj/Documents/AIworkspace/devlake/AI_HANDOFF_GUIDE.md` first. Below is the complete context of what has been built, the environment details, and what remains to be executed.

## 1. Objective
Automate Apache DevLake (`v1.0.2`) to:
1. Delete all legacy projects that were bound to the old GitHub App connection (`devops-demo`, Connection `#1`).
2. Bind all **455 valid repositories** (across **45 GitHub organizations**, sourced strictly from `/home/ngsj/Documents/AIworkspace/devlake/all_456_repositories.json`) and the `Fenix-DORA-Config` Scope Config to the **GitHub PAT connection named `sj-test-devlake`**.
3. Create **45 1:1 Organization Projects** (`accurics`, `appx`, ..., `zaproxy`) bound to `sj-test-devlake` with `skipOnFail: True` and DORA metrics enabled.
4. Automatically trigger the Blueprint data collection pipelines (`POST /blueprints/{bp_id}/trigger` — equivalent to clicking "Collect Data") for all 45 organizations on both `https://devlake.duck.web` and `http://localhost:8080`.
5. **Constraint:** 100% pure DevLake REST API calls only (Zero direct MySQL access / no `docker exec devlake-mysql`).

---

## 2. Environment, Credentials & Network Topology

| Target Environment | Base URL | API Prefix | Basic Auth User | Basic Auth Password | Active PAT Connection |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Enterprise DevLake (`duck.web`)** | `https://devlake.duck.web` | `/api` | `appxadmin` | `appxpa$$w0rd` | **Connection `#2`:** `sj-test-devlake` (`AccessToken`)<br>*(Connection `#1` is legacy `devops-demo`)* |
| **Local Docker DevLake (`localhost`)** | `http://localhost:8080` | `""` *(root)* | *(None)* | *(None)* | **Connection `#1`:** `sj-test-devlake` (`AccessToken`) |
| **Enterprise GitHub (GHES)** | `https://github.fenix.dev/api/v3/` | — | — | PAT in `sj-test-devlake` | Internal IP: `10.252.1.101:443` |

### Critical Technical Pitfalls Already Solved
1. **Dual Wi-Fi Network Switching (`Darumaka` vs `Moltres`):**
   - The laptop switches between **`Darumaka`** (internet Wi-Fi, DNS `8.8.8.8`) and **`Moltres`** (air-gapped enterprise Wi-Fi, `192.168.254.x`).
   - When on `Darumaka`, `devlake.duck.web` resolves to `127.0.53.53` (ICANN Name Collision IP), which hits a local Rancher/Traefik container on port 443 and returns `HTTP 404`.
   - Commands targeting `https://devlake.duck.web` must be run while connected to **`Moltres`**.
2. **Bash/Zsh `$$` PID Expansion in Password (`appxpa$$w0rd`):**
   - Passing `"appxpa$$w0rd"` in **double quotes** in Bash/Zsh expands `$$` into the shell's PID (e.g., `appxpa679483w0rd`), causing `HTTP 401 Unauthorized`.
   - All scripts now have `appxpa$$w0rd` hardcoded as the default and include `sanitize_shell_password()` to auto-restore `appxpa$$w0rd` if shell PID expansion occurs. Always use **single quotes** (`'appxpa$$w0rd'`) in shell commands.
3. **Strict Catalog Usage (`all_456_repositories.json`):**
   - **Do NOT crawl GitHub `/organizations` or `/remote-scopes`** to discover orgs/repos (the PAT does not have access to all enterprise orgs).
   - Strictly use `/home/ngsj/Documents/AIworkspace/devlake/all_456_repositories.json`, which contains the exact **455 valid repositories across 45 organizations** (excluding deleted repo `githubId: 237` / `htx-devops/my-first-project`).

---

## 3. Workspace Files & Scripts (`/home/ngsj/Documents/AIworkspace/devlake/`)

| File Path | Purpose & Status |
| :--- | :--- |
| `/home/ngsj/Documents/AIworkspace/devlake/auto_pull_all_org_data.py` | **Main End-to-End Script (Ready).** Connects to `duck.web` (`--duck`), `localhost:8080` (`--localhost`), or both (`--target both`). Purges old projects (`--clean`), upserts `Fenix-DORA-Config`, uploads all 455 repos from `all_456_repositories.json` in a single `<1s` `PUT /scopes` call, prunes any non-catalog scopes, creates all 45 org projects bound to `sj-test-devlake`, and triggers `POST /blueprints/{bp_id}/trigger` for every org. |
| `/home/ngsj/Documents/AIworkspace/devlake/reset_and_recreate_projects.py` | **Project Reset & Provisioning Script (Ready).** Supports `--mode org` (45 org projects) or `--mode single` (1 consolidated project). Purges old projects, uploads 455 scopes to `sj-test-devlake`, creates projects/blueprints, and triggers data pull. |
| `/home/ngsj/Documents/AIworkspace/devlake/clear_all_projects.py` | **Standalone Purge Script (Ready).** Deletes all projects (`DELETE /projects/{name}`) and orphan blueprints (`DELETE /blueprints/{id}`). Supports default (`duck.web`) or `--localhost`. |
| `/home/ngsj/Documents/AIworkspace/devlake/all_456_repositories.json` | **Verified Golden Catalog (Ready).** Contains the exact 455 valid repositories across 45 organizations with `HTMLUrl` and `cloneUrl` populated. |
| `/home/ngsj/Documents/AIworkspace/devlake/AI_HANDOFF_GUIDE.md` | Full technical reference document stored in the workspace. |

---

## 4. Current State & Exact Steps to Finish

### Current State
- On `https://devlake.duck.web`: All 45 old projects (previously bound to `devops-demo`) were deleted in Phase 1 of the last run before `Ctrl+C` was pressed.
- Both `auto_pull_all_org_data.py` and `reset_and_recreate_projects.py` have been updated to:
  1. Read strictly from `all_456_repositories.json` (zero GitHub crawling).
  2. Upload all 455 repos in a single `< 1s` `PUT /scopes` call and prune any extra scopes from Connection `#2` (`sj-test-devlake`).
  3. Recreate all 45 organization projects bound to Connection `#2` (`sj-test-devlake`) and immediately trigger `POST /blueprints/{bp_id}/trigger` for all 45 orgs.

### Action Required to Finish
1. Ensure Wi-Fi is connected to **`Moltres`** (so `https://devlake.duck.web` and `https://github.fenix.dev` are reachable).
2. Run:
   ```bash
   python3 /home/ngsj/Documents/AIworkspace/devlake/auto_pull_all_org_data.py --target both --clean
   ```
3. Verify that all 45 projects are created and their pipelines are running/queued on `https://devlake.duck.web`:
   ```bash
   curl -sk -u 'appxadmin:appxpa$$w0rd' 'https://devlake.duck.web/api/pipelines?pageSize=20' | python3 -m json.tool | head -n 40
   ```
