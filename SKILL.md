---
name: bjtu-hpc-submit
description: Use when an agent needs to refresh/save BJTU HPC portal auth, add or switch BJTU portal accounts/tokens, run a local transfer dashboard, upload code or data, submit CPU/GPU jobs, inspect job status including GPU counts, monitor resumable dataset uploads, or probe the runtime environment from a BJTU HPC helper workspace.
---

# BJTU HPC Submit

Tool-first workflow for BJTU HPC portal work from a local helper workspace. This public version is sanitized: replace placeholder paths, account names, and project directories with your own local values before use.

## Runtime Defaults

- Work from the helper workspace unless the project says otherwise:

```bash
PY=/path/to/python3
SLURM_DIR="/path/to/bjtu-hpc-helper"
PROJECT_DIR="/path/to/your/project"
```

- Prefer the same Python environment used to install the helper dependencies. If dependencies are missing, install the helper requirements and Playwright Chromium:

```bash
cd "$SLURM_DIR"
"$PY" -m pip install -r requirements.txt
"$PY" -m playwright install chromium
```

- When working from a project, save portal and Slurm evidence under a project-local log directory such as `$PROJECT_DIR/hpc_stdout/`.
- Use broad keywords for general queue checks. Use narrow keywords only for targeted follow-ups.

## Entry Points

- Start with `hpc_doctor.py --json`; it checks dependencies, account state, browser profile, and token validity without printing secrets.
- For portal-app jobs, prefer verified submit wrappers over raw submit scripts when available.
- For CPU/GRES-sensitive jobs, prefer uploaded native `sbatch` scripts over portal-generated PyTorch app scripts, then verify native Slurm allocation.
- For MCP clients, prefer tools that expose auth status, submit-and-verify, pending reason, allocation verification, stdout tailing, and SFTP info.

Useful status commands:

```bash
cd "$PROJECT_DIR"
"$PY" "$SLURM_DIR/hpc_jobs.py" list --keyword <keyword> --size 30 --paths
"$PY" "$SLURM_DIR/hpc_jobs.py" list --keyword <keyword> --size 30 --paths --json > hpc_stdout/bjtu_jobs_YYYYMMDD_HHMM.json
"$PY" "$SLURM_DIR/hpc_pending_reason.py" <slurm_job_id> --no-sinfo
```

## Auth

- Saved accounts usually live in `~/.bjtu_hpc_accounts.json`; a legacy single-token cache may live in `~/.bjtu_hpc_token`.
- Treat the saved account store as the source of truth; the legacy file is only a compatibility cache for older scripts.
- Select accounts with `--auth-account <name>` or `HPC_AUTH_ACCOUNT=<name>`.
- Never print portal tokens, cookies, temporary certificates, passwords, or captured browser storage.
- Treat portal codes `11009`, `11011`, and `11012` as expired or invalid auth.
- Treat portal HTTP `401`, token-validation transport errors, and missing profile tokens as auth-blocked for user-requested live status until a fresh validation succeeds. Stale snapshots may be reported only as `last trusted`.

### Auth Recovery State Machine

Use one integrated `hpc_refresh_flow.py` command that owns validation, profile probing, optional visible login, and post-login status collection. Do not manually bounce between doctor, job-list, and visible browser attempts unless the integrated command has exited and validation still fails.

1. For routine refreshes when no command is currently blocked:

```bash
cd "$SLURM_DIR" && "$PY" hpc_refresh_flow.py <account_name>
```

2. If invalid auth blocks a user-requested status check, progress check, pending-reason check, upload, or submit, run the integrated blocked-task flow in a PTY and keep it running:

```bash
cd "$SLURM_DIR" && "$PY" hpc_refresh_flow.py <account_name> --visible-only
```

3. For project progress checks, use the post-login status variant so the same command continues after any refresh or login:

```bash
cd "$PROJECT_DIR"
"$PY" "$SLURM_DIR/hpc_refresh_flow.py" <account_name> --visible-only \
  --after-jobs-keyword <keyword> --after-jobs-size 30 --after-jobs-paths \
  --after-snapshot-dir "$PROJECT_DIR/hpc_stdout" \
  --after-pending-job <slurm_job_id> --after-pending-no-sinfo
```

Interpret the integrated command by its output:

- `validate saved token ... ok`: token was already usable. Continue; do not open a browser.
- `refreshed ... headlessly` or `from the existing Playwright profile`: profile recovery succeeded. Continue; do not ask the user to log in.
- `[action] A Playwright Chromium window should open now`: only now ask the user to finish CAS/captcha, wait for the HPC portal home page to load, then close the Playwright window.
- A Playwright/Chromium window that opens and closes almost immediately after a recent successful login is usually normal profile validation. Keep the command running and wait for `[ok]`, a post-login job table, or an explicit validation error.

Operational rules:

- Run the refresh command in a PTY and keep it running while the user logs in.
- `--visible-only` does not blindly open a browser. It first validates the saved account token and briefly probes the selected Playwright profile.
- If there is no stdout for about 30 seconds, check whether `Google Chrome for Testing` or `hpc_refresh_flow` is running:

```bash
pgrep -afil "Google Chrome for Testing|playwright|hpc_refresh_flow"
```

- Do not screenshot or inspect login pages because they may contain account, CAPTCHA, or token material.
- If the command exits with `timed out waiting for token in visible browser`, first run `hpc_accounts.py validate <account_name>`. If validation succeeds, continue; if it still fails, rerun the integrated `--visible-only` flow once.
- Use `--force --visible-only --no-profile-probe-before-visible` only after one integrated attempt exits without a usable token and validation still fails, or when the user explicitly requests a visible login window. Do not use this as the first attempt.
- If the second visible attempt still fails to save a usable token, report the auth/token-save failure as the blocker and keep live status at the latest trusted snapshot.

## Job Rules

- Default single-process GPU shape on `cluster2`:

```text
--gpu 1 --ntasks 1 --cpus-per-task 8 --gres-flags disable-binding
```

- Native Slurm equivalent for one GPU:

```bash
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --gres=gpu:1
#SBATCH --gres-flags=disable-binding
```

- Request more GPUs only when the code actually uses them.
- Avoid `--gpu 1 --ntasks 8` without `--gres-flags disable-binding`; it can produce `BadConstraints`.
- After every submit, verify the portal job row. If the job is `PENDING`, report the native Slurm `Reason`, not just portal state.
- If CPU/GRES shape matters, verify native `NumCPUs`, `NumTasks`, `CPUs/Task`, and GPU TRES with `scontrol`; portal request fields are not enough.
- Do not cancel unrelated jobs. For per-user job-count limits, inspect existing jobs before canceling anything.
- Always run `sbatch --test-only` for a new native script or a new resource shape before real submission.

Known-good shapes on `cluster2`:

```text
1 GPU single process: --ntasks=1 --cpus-per-task=8 --gres=gpu:1 --gres-flags=disable-binding
2 GPU packed job:     --ntasks=1 --cpus-per-task=8 --gres=gpu:2 --gres-flags=disable-binding
```

## Native Slurm Packed Jobs

Use packed jobs only when one Slurm allocation intentionally launches multiple child experiments.

Checklist:

1. Request one batch allocation with the required GPU count, for example `--gres=gpu:2`, `--ntasks=1`, `--cpus-per-task=8`, and `--gres-flags=disable-binding`.
2. In the batch script, read allocation-provided `CUDA_VISIBLE_DEVICES` and split it into child lanes. Do not hardcode physical `0/1`.
3. For each child, set `CUDA_VISIBLE_DEVICES` to exactly one allocated id, run lightweight `nvidia-smi` and `torch.cuda.device_count()` checks, then launch the experiment.
4. Save a batch stdout plus one child log per lane.
5. After submission, run native checks:

```bash
cd "$PROJECT_DIR"
"$PY" "$SLURM_DIR/hpc_pending_reason.py" <job_id> --no-sinfo
```

6. Verify `JobState=RUNNING`, `Reason=None`, CPU fields, GPU TRES, and node name.
7. Tail child logs and verify that each child reports one visible CUDA device and has entered real training before calling the launch successful.

## Paths

- Portal SSH/SFTP should go through the helper's SFTP-info command. Do not hardcode one-time certificate tokens.
- Portal SSH uses a temporary certificate token, not the local SSH key.
- Reusable code can live under a user-controlled cluster path such as `$BJTU_HOME/code/<project>`.
- Portal job work/output directories usually live under a user-controlled cluster jobs directory.
- Trust job-side probes for runtime facts, not login-node inference.

Evidence examples:

- Portal snapshots: `$PROJECT_DIR/hpc_stdout/bjtu_jobs_YYYYMMDD_HHMM*.json`
- Native Slurm snapshots: `$PROJECT_DIR/hpc_stdout/bjtu_pending_reason_YYYYMMDD_HHMMSS*.json`
- Downloaded launch logs: `$PROJECT_DIR/hpc_stdout/bjtu_<jobid>_<shortname>_launch_YYYYMMDD.log`

Download pattern:

```bash
"$PY" "$SLURM_DIR/hpc_download.py" "<remote_log_path>" -o "$PROJECT_DIR/hpc_stdout/bjtu_<jobid>_<shortname>_launch_YYYYMMDD.log" --no-progress
```

## Dataset Upload

- Prefer resumable, chunked uploads for large datasets.
- Never run two upload workers writing the same `.part` file.
- When a source host is slow or unreliable, use cluster-side file size/progress as the source of truth.

## Post-Submit Evidence Checklist

Before reporting a job as running:

- Portal row is present and the expected job id, GPU count, CPU count, and node are recorded.
- Native Slurm reason was checked.
- CPU/GPU allocation matches the intended shape.
- Startup logs were downloaded or tailed locally.
- At least one real training/progress line was observed, not only environment setup.
- Evidence files were saved under the project log directory.

## Safety

- For cross-account dataset sharing, inspect ACLs first; do not apply ACL/chmod changes without explicit confirmation.
- Do not publish tokens, cookies, passwords, one-time certificate strings, local absolute paths, student ids, or project-specific job evidence.
