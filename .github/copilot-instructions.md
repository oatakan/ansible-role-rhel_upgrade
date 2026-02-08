# Copilot instructions (ansible-role-rhel_upgrade)

## What this repo is
- This is an Ansible Galaxy role (`meta/main.yml`) intended to bring an EL host (RHEL/CentOS) up to date via package updates, then optionally reboot when a newer kernel is installed.
- Role entrypoint is `tasks/main.yml`.

## Execution flow (big picture)
`tasks/main.yml` runs in this order:
1. Optional CentOS EOL repo fix (`tasks/centos_eol_fix.yml`) when `ansible_distribution == 'CentOS'` and `centos_eol_vault_fix: true`.
2. Captures pre-update OS facts into `os_info_pre_update` and prints them.
3. Collects package facts (`package_facts`) and prints them at verbosity 3.
4. Shows pending updates via `yum: list=updates`.
5. Includes `tasks/update.yml` to perform the actual update.
6. Re-checks pending updates, collects package facts again, computes `latest_kernel_version`, and sets `system_reboot_needed`.
7. If `update_reboot_kernel: true` and reboot is needed, reboots and prints post-reboot OS facts.

## Update behavior (important)
- The update is implemented in `tasks/update.yml` as a `block`/`rescue`:
  - First attempt: `ansible.builtin.yum` with `allowerasing: false`.
  - On dependency conflict: if `force_allow_erasing: true`, runs `dnf clean all` then retries with `allowerasing: true`.
  - If `force_allow_erasing` is not enabled, the role fails with a clear message.
- Keep this "strict-first, permissive-rescue" behavior if you change update logic.

## Role variables (defaults)
Defined in `defaults/main.yml`:
- `update_distro_packages` (default `"*"`) and `update_distro_packages_excludes` (list) control the package selection.
- `update_reboot_kernel` (bool) and `reboot_timeout` control reboot behavior.
- `centos_eol_vault_fix` toggles the CentOS vault baseurl rewrite.
- `force_allow_erasing` toggles the rescue path in `tasks/update.yml`.

## Conventions used in tasks
- Prefer fully-qualified builtins (e.g., `ansible.builtin.yum`, `ansible.builtin.include_tasks`, `ansible.builtin.set_fact`).
- Package operations run with `become: true`.
- Debug output is sometimes gated with `verbosity: 3`.
- The role intentionally inspects/prints update lists before and after running updates.

## Developer workflows
- Minimal smoke run (local): `ansible-playbook -i tests/inventory tests/test.yml --become -K`
  - Note: `tests/test.yml` currently targets `localhost` and runs role `oatakan.rhel_upgrade`.
- Linting: `ansible-lint`
  - Config in `.ansible-lint` skips rules like `package-latest` and `ignore-errors`; don’t “fix” those patterns unless you’re changing behavior intentionally.
- CI (GitHub Actions): `.github/workflows/ci.yml` runs `yamllint`, `ansible-lint`, and a playbook `--syntax-check` on PRs.
- PR enrichment (GitHub Actions): `.github/workflows/pr-enrichment.yml` posts an “AI Pull Request Analysis” comment and responds to `/ai ...` commands on PR comments.
  - Commands: `/ai review`, `/ai test`, `/ai changelog`, `/ai docs`, `/ai improve`, `/ai help`.
  - Optional secrets: `OPENAI_API_KEY` and/or `ANTHROPIC_API_KEY` (falls back to rule-based output if unset).
- VM integration testing: `molecule/vagrant` is intended for self-hosted runners (see `.github/workflows/vm-test.yml`).

## Common sharp edges
- Kernel comparison relies on `package_facts` containing `ansible_facts.packages.kernel[-1]`; keep this consistent if you refactor kernel logic.
- `dnf clean all` in `tasks/update.yml` assumes a DNF-based system; changes should preserve EL compatibility.
