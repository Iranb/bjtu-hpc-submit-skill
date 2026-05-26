# BJTU HPC Submit Skill

A Codex skill for operating BJTU HPC portal workflows from a local helper workspace. It focuses on reliable portal authentication recovery, CPU/GRES-safe Slurm submission, packed GPU jobs, resumable uploads, and evidence-driven monitoring.

This repository contains a sanitized public version of the skill. It intentionally does not include personal account ids, local absolute paths, portal tokens, browser cache contents, project-specific job ids, or experiment logs.

## What It Covers

- BJTU portal token validation and recovery.
- Playwright profile-based token refresh with visible-login fallback.
- Portal job status checks and native Slurm pending-reason checks.
- CPU/GRES-safe GPU job submission rules.
- Packed multi-GPU Slurm jobs that respect allocated `CUDA_VISIBLE_DEVICES`.
- Evidence capture for job tables, native allocation snapshots, stdout tails, and launch logs.

## Install

Copy this directory into your Codex skills directory:

```bash
mkdir -p ~/.codex/skills
cp -R bjtu-hpc-submit ~/.codex/skills/
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
- `hpc_jobs.py`
- `hpc_pending_reason.py`
- `hpc_download.py`

The helper scripts are not included in this repository. Set these shell variables before using the examples:

```bash
PY=/path/to/python3
SLURM_DIR=/path/to/bjtu-hpc-helper
PROJECT_DIR=/path/to/your/project
```

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
- Use forced visible login only after the integrated flow fails and token validation still fails.

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
