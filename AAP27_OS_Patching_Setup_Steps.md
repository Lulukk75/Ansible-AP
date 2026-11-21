# AAP 2.7 OS Patching: Setup Steps and Status

Written for: the platform admin who will run and maintain the OS patching job in AAP 2.7.

## 1. Environment

| Item | Value |
| --- | --- |
| AAP web / API | `https://172.17.152.208` (gateway reports version `2.7`, controller reports `4.8.14`) |
| AAP host (SSH) | `172.17.152.208` (hostname `aap.example.com`) |
| AAP SSH login used | `daddy` (in `wheel`, passwordless sudo) |
| Managed RHEL host in AAP | Inventory host `RHEL` (id 2) in `Demo Inventory` (id 1) |
| Execution environment | `Default execution environment` / `ee-supported-rhel9` |
| Organization | `Default` (id 1) |

## 2. Deliverables

Folder: `AAP 2.7/os_patching/`

| File | Purpose |
| --- | --- |
| `os_patching.yml` | The playbook. Uses `ansible.builtin` only, supports dnf and yum, and runs in rolling batches. |
| `README.md` | Generic step-by-step guide for AAP 2.7. |
| `AAP27_OS_Patching_Setup_Steps.md` | This file. |

## 3. Steps completed

### Step 1. Check the playbook locally

- Parsed the YAML and all 30 Jinja expressions with Python. Both are valid.
- Fixed a bug in the disk-space check: `df -P` cannot be combined with `--output`. The command is now `df -m --output=avail <mount>`.
- `ansible-playbook` cannot run natively on this Windows machine, so the playbook has not been run against a real host yet.

### Step 2. Check the AAP API

Read-only checks using the admin account:

- Gateway ping: version `2.7`.
- Controller ping: version `4.8.14`.
- Existing objects found: 5 execution environments, 1 demo project, 1 demo job template, 1 demo inventory, 3 credentials (including `RHEL-Host-SSH-Key`, username `daddy`), and 2 hosts (`localhost` and `RHEL`).
- No jobs had been run before.

### Step 3. Authorize SSH access to the AAP host (option 2)

1. The personal key `~/.ssh/id_ed25519` was installed for `daddy` with `ssh-copy-id`. It is passphrase-protected.
2. Verified with a read-only command: the login returned `aap.example.com`, the user is `daddy` in `wheel`, `git` is installed, and sudo is passwordless.
3. The key line appears twice in `~daddy/.ssh/authorized_keys`. This is harmless. To tidy it, run on the AAP host:
   ```bash
   sort -u -o ~/.ssh/authorized_keys ~/.ssh/authorized_keys
   ```

### Step 4. Create the Git repository on the AAP host

On the AAP host, as `daddy`:

```bash
mkdir -p ~/git/os_patching.git && git init --bare -b main ~/git/os_patching.git
```

From this PC, in Git Bash:

```bash
cd "/c/Users/LulukKristiawan/OneDrive - IBM/Documents/2026/2026 Danantara/AAP 2.7/os_patching"
git init -b main && git add os_patching.yml README.md && git commit -m "OS patching playbook"
git remote add origin daddy@172.17.152.208:git/os_patching.git
git push -u origin main
```

Result: commit `830e85e` (`OS patching playbook`) is on `main`.

### Step 5. Create a restricted key for the AAP controller

Your personal key has a passphrase, which the controller cannot use. A separate key was created for AAP:

- Private key: `~/.ssh/aap-scm/aap_scm_ed25519` on this PC (never share it).
- Public key: `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIB2Q76jOZ4lS4UEubLFEJdzC1aOA261lJAOzomKQUI1H aap-controller-scm-readonly`
- Fingerprint: `SHA256:w7iui+cAh9PjDLn4PqdRA6Ye06sZewmRuSH08X5jnxM`

The public key was installed on the AAP host in `~daddy/.ssh/authorized_keys` with a restriction:

```text
restrict,command="git-upload-pack /home/daddy/git/os_patching.git" ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIB2Q76jOZ4lS4UEubLFEJdzC1aOA261lJAOzomKQUI1H aap-controller-scm-readonly
```

This key can only read that one repository. It cannot get a shell, forward ports, or push.

### Step 6. Create the AAP objects (API)

| Object | ID | Details |
| --- | --- | --- |
| Source Control credential `OS-Patching-Git-Key` | 4 | Type `Source Control`, username `daddy`, uses the restricted key |
| Project `OS Patching` | 8 | SCM URL `ssh://daddy@172.17.152.208/home/daddy/git/os_patching.git`, branch `main`, credential 4 |
| Project sync | 6 | Status `successful`. AAP found `os_patching.yml`. |
| Job template `OS-Patching-Linux` | 9 | Job type `run`, inventory `Demo Inventory`, project 8, playbook `os_patching.yml`, limit `RHEL`, become enabled, timeout 7200s, concurrent jobs off, prompt on launch for limit and job type |
| Credentials on template 9 | | `RHEL-Host-SSH-Key` (id 3) |
| Survey on template 9 | | Enabled. Questions: `patch_mode`, `reboot_policy`, `patch_batch_size`, `patch_packages`, `exclude_packages` |

The launch check shows that the job needs one password at launch: `become_password` (the sudo password for `RHEL`). It is not stored in AAP.

## 4. Current status

| Item | Status |
| --- | --- |
| Playbook syntax (YAML and Jinja) | Passed locally |
| Git repository on the AAP host | Done |
| Project sync | Successful |
| Job template and survey | Created |
| Check run against `RHEL` | **Not launched yet** |
| Real patching run | **Not started** (requires your approval) |

## 5. How to run the Check job

1. In the AAP UI, go to **Automation Execution → Templates → OS-Patching-Linux**, then click **Launch**.
2. Set **Job type** to **Check**. Keep **Limit** as `RHEL`.
3. Keep the survey defaults: `security`, `if_needed`, `25%`.
4. Enter the sudo password when prompted, then launch.

Expected result: the job completes without changing the host. Package tasks report what would change.

Possible failure points:

- **Unreachable**: the host `RHEL` has no `ansible_host` set. Set it to the managed host's IP address in the host's variables.
- **Sudo failure**: the sudo password entered at launch is wrong.
- **yum-utils or df failure**: the managed host has an unexpected configuration. Send the job output for diagnosis.

## 6. Security notes

- The AAP admin password was used in plain text in shell commands during setup, and it is written in `AAP 2.7/AAP 27at RH9.md`. **Change it** in AAP, then remove it from that file. This document intentionally does not contain the password.
- The SSH password for `daddy` was typed by you and is not stored in these files.
- The `RHEL-Host-SSH-Key` credential stores the username `daddy` and a password for the managed host. Only the encrypted form lives in AAP.
- Do not commit private keys to Git. Only the playbook and docs are in the repository.

## 7. Cleanup checklist (after testing)

- [ ] Remove the `aap-controller-scm-readonly` line from `~daddy/.ssh/authorized_keys` on `172.17.152.208`. This revokes AAP's read access to the repository. Remove it only after testing, because the project depends on it.
- [ ] Optionally remove the duplicate `lulukkristiawan` line from `authorized_keys`, and tidy the file with `sort -u`.
- [ ] Rotate the AAP admin password and remove it from `AAP 2.7/AAP 27at RH9.md`.
- [ ] Delete `~/.ssh/aap-scm/` on this PC if AAP no longer needs to pull from the repo.
- [ ] Decide whether `daddy` should keep a password-based sudo or passwordless sudo for patching.

## 8. Useful links and IDs

- Controller API base: `https://172.17.152.208/api/controller/v2/`
- Project: `/projects/8/` (`OS Patching`)
- Job template: `/job_templates/9/` (`OS-Patching-Linux`)
- Project sync log: `/project_updates/6/stdout/?format=txt`
