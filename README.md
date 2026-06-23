# BJTU HPC Submit Skill

A Codex skill for operating BJTU HPC portal workflows from a local helper workspace. It focuses on reliable portal authentication recovery, multi-account job scheduling, CPU/GRES-safe Slurm submission, packed GPU jobs, reusable datasets, account-local runtime environments, resumable uploads, and evidence-driven monitoring.

This repository contains a sanitized public version of the skill. It intentionally does not include personal account ids, local absolute paths, portal tokens, browser cache contents, project-specific job ids, or experiment logs.

GitHub: [Iranb/bjtu-hpc-submit-skill](https://github.com/Iranb/bjtu-hpc-submit-skill)

## What It Covers

- BJTU portal token validation and recovery.
- Multi-account token management through `hpc_accounts.py`.
- Playwright profile-based token refresh with visible-login fallback.
- Local dashboard and Token Guardian operating rules.
- Two run-slot experiments plus two queued follow-up experiments per saved account.
- Dataset reuse across cluster accounts through verified filesystem permissions or ACLs.
- Account-local runtime environment copies for cross-account runs.
- Portal job status checks and native Slurm pending-reason checks.
- CPU/GRES-safe GPU job submission rules, including native Slurm verification, forced `--gres-flags=disable-binding`, emergency packed-job CPU fallback, and 2GPU-to-1GPU compatibility fallback.
- Fast native queue summaries across saved accounts through `hpc_queue_summary.py`.
- Packed multi-GPU Slurm jobs that respect allocated `CUDA_VISIBLE_DEVICES`.
- Safe `sbatch --hold` submit-cap probes that are immediately cancelled and do not start work.
- Evidence capture for job tables, native allocation snapshots, stdout tails, and launch logs.

## Install

Clone the public repository and install it into your Codex skills directory:

```bash
mkdir -p ~/.codex/skills
git clone https://github.com/Iranb/bjtu-hpc-submit-skill.git
cp -R bjtu-hpc-submit-skill ~/.codex/skills/bjtu-hpc-submit
```

Or keep it as a repository and symlink it:

```bash
ln -s /path/to/bjtu-hpc-submit-skill ~/.codex/skills/bjtu-hpc-submit
```

## Required Local Helper Scripts

The skill assumes you have a local BJTU helper workspace that provides scripts such as:

- `hpc_doctor.py`
- `hpc_accounts.py`
- `hpc_refresh_flow.py`
- `hpc_transfer_web.py`
- `hpc_jobs.py`
- `hpc_pending_reason.py`
- `hpc_submit.py`
- `hpc_submit_verified.py`
- `hpc_upload.py`
- `hpc_download.py`
- `hpc_winscp_info.py`
- `hpc_share_check.py`

The helper scripts are not included in this repository. Set these shell variables before using the examples:

```bash
PY=/path/to/python3
SLURM_DIR=/path/to/bjtu-hpc-helper
PROJECT_DIR=/path/to/your/project
```

## Dashboard And Token Guardian

The local dashboard can manage saved CAS logins, refresh and validate portal tokens, create resumable upload tasks, launch uploads, show cluster-side progress, and list portal jobs. Saved passwords must stay in a local credentials store with restrictive permissions; the UI should only show whether a password exists.

Token Guardian is a background validator and best-effort headless refresher. Use a conservative default of a 300 second validation interval and a 1800 second refresh threshold. Treat a headless refresh as actually successful only when validation succeeds and the saved token changes or the account `token_updated_at` advances. If headless refresh times out while final validation still succeeds, the old token is still usable but renewal was not proven.

In multi-account workflows, upload tasks should include `auth_account` so launch commands, SFTP certificate lookup, and progress checks use the correct saved account instead of hardcoded defaults. If the dashboard is installed as a user service, service status output should be redacted before printing raw environment or service-manager details.

## Recommended Auth Flow

For routine refresh:

```bash
cd "$SLURM_DIR" && "$PY" hpc_refresh_flow.py <account_name>
```

For blocked status/submit/upload tasks:

```bash
cd "$SLURM_DIR" && "$PY" hpc_refresh_flow.py <account_name> --visible-only
```

Important behavior:

- `--visible-only` first validates the saved token and probes the Playwright profile.
- A browser should only be expected after the command prints the `[action]` line.
- If profile recovery succeeds, do not ask the user to log in again.
- If the user closes the visible browser and the command has not finished, validate the same account or run a headless profile refresh before opening another visible browser.
- Use forced visible login only after the integrated flow fails and token validation still fails.

## Multi-Account Runs

For batch experiments, the skill uses this default cap:

```text
per saved account: 2 run-slot experiments + 2 queued follow-up experiments
```

Use `--auth-account <name>` for every submit and status command, keep code/output/environment paths inside the corresponding cluster OS account home, and share only datasets through verified filesystem permissions or symlinks.

When checking whether an account can submit one more job, distinguish submit acceptance from run-slot availability. A job may be accepted by `sbatch` but remain pending with a native reason such as `QOSMaxJobsPerUserLimit`. Use a unique held probe when you need to test submit capacity without disturbing existing runs:

```bash
sbatch --test-only <probe>.sbatch
PROBE_JOB_ID=$(sbatch --hold --parsable <probe>.sbatch)
squeue -j "$PROBE_JOB_ID"
scancel "$PROBE_JOB_ID"
squeue -j "$PROBE_JOB_ID"
```

The probe should have a unique name, short time limit, and the same resource shape being tested. It should be cancelled immediately and should not replace or cancel any unrelated job.

Target native Slurm GPU training shape:

```bash
--gpu 1 --ntasks 1 --cpus-per-task 16 --gres-flags disable-binding
```

For normal training, keep `--gres-flags=disable-binding` and start with `16` CPU cores per training task. If `sbatch --test-only` or scheduler constraints reject `16`, retry with `12`, then `8`; treat `8` as the ordinary minimum for evidence-producing GPU training. For native packed `2GPU` jobs only, if `2GPU/16CPU` is still blocked by `Resources`, reservation constraints, or node CPU availability, test emergency `2GPU/8CPU` with `--ntasks=2 --cpus-per-task=4 --gres=gpu:2` before giving up or switching accounts.

The packed GPU fallback ladder is:

```text
2GPU/32CPU: --ntasks=2 --cpus-per-task=16 --gres=gpu:2
2GPU/24CPU: --ntasks=2 --cpus-per-task=12 --gres=gpu:2
2GPU/16CPU: --ntasks=2 --cpus-per-task=8  --gres=gpu:2
2GPU/8CPU:  --ntasks=2 --cpus-per-task=4  --gres=gpu:2  (emergency Resources/reservation fallback)
```

Before replacing a pending packed job, inspect native Slurm reason and node availability. A `Resources` blocker can come from same-node CPU shortage or an active reservation even when GPU summaries appear to show free devices. A pure `Priority` blocker should normally keep its queue position unless the user explicitly accepts a lower-CPU retry.

CPU/GRES-sensitive training should use a native `sbatch` script and then verify `NumCPUs`, `NumTasks`, `CPUs/Task`, and GPU TRES with `scontrol show job <job_id>`. Portal PyTorch/GPU app templates may accept CPU/GRES fields in the helper payload but drop those directives from the generated Slurm script, so portal request fields are not proof of allocation.

If you must use a portal app wrapper, use a verified submit path and require it to resolve the native Slurm id from either the immediate job row or `wait.job` when `--wait` is used. Missing Slurm id or allocation mismatch should be treated as a failed launch.

## Sanitization

Before publishing, the skill was stripped of:

- local user paths
- student/account ids
- cluster home paths tied to a specific user
- project-specific experiment names and job ids
- tokens, cookies, passwords, and one-time certificate strings

If you fork this repository, keep those constraints in place before committing local modifications.

## License

MIT License. See [LICENSE](LICENSE).
