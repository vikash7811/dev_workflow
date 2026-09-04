# Workflow

## Select a development context

Profiles define the repository and, when useful, ROS and Git identity expectations.
Select one once:

```bash
dev-use my_project
```

Then normal commands no longer need repeated repository/workspace arguments:

```bash
dev-git-update
dev-precheck
dev-ros-check --test
dev-validate
```

## Configure user Git identity once

```bash
dev-git-config set personal \
  --author-name "Jane Developer" \
  --author-email jane@example.com \
  --ssh-alias github-personal \
  --ssh-host github.com \
  --ssh-key ~/.ssh/id_ed25519_personal

dev-git-config use personal
```

Generate the SSH config stanza with:

```bash
dev-git-config ssh-snippet personal
```

Then validate the active repository:

```bash
dev-git-check
```

A project profile may pin `DEV_GIT_PROFILE="personal"` so the expected identity follows
the project automatically.

## Safe repository update

```bash
dev-git-update
```

This validates the configured identity, requires a clean tree, fetches/prunes, and
fast-forwards only. Diverged histories are refused unless you explicitly request:

```bash
dev-git-update --rebase
```

## Normal development cycle

```text
dev-use my_project
    ↓
dev-git-update
    ↓
edit/apply patch
    ↓
dev-precheck
    ↓
runtime/simulation/benchmark when behavior changed
    ↓
git add exact files
    ↓
dev-commit-push --message "..."
```

`dev-precheck` performs:

1. configured Git identity/remote validation;
2. Python cache cleanup;
3. Git executable/source hygiene;
4. changed-line formatting relative to `HEAD`;
5. standalone CMake tests when configured;
6. working-tree and staged whitespace checks;
7. automatic ROS1/ROS2 package build for repositories containing `package.xml`;
8. ROS package tests automatically when no standalone CMake test path exists.

If no Git identity profile is configured, the Git identity check reports `SKIP` and the
rest of the workflow remains usable.

Use `--skip-tests` to suppress tests while keeping the ROS build. Use `--no-ros` to
intentionally suppress the automatic ROS build.

## ROS auto-detection

`dev-ros-check` resolves the following automatically where possible:

```text
ROS generation -> package.xml build tool / build_type
workspace      -> containing WORKSPACE/src
packages       -> package.xml files under the selected repository
builder        -> ROS1: catkin/catkin_make, ROS2: colcon
```

Profiles should pin `DEV_ROS_SETUP` when multiple ROS distributions are installed.

### ROS1

```bash
dev-use ros1_project
dev-ros-check
dev-ros-check --test
```

`auto` prefers `catkin` when available and falls back to `catkin_make`.

### ROS2

```bash
dev-use ros2_project
dev-ros-check
dev-ros-check --test
```

ROS2 uses `colcon build --packages-up-to` so repository packages and their workspace
dependencies are built, while `colcon test --packages-select` keeps tests scoped to the
selected repository packages.

The ROS setup is sourced inside `dev-ros-check`, so switching project profiles does not
permanently change the parent shell.

## Before push / PR

```bash
dev-validate
```

`dev-validate` inherits Git identity validation and automatic ROS checking. It automatically
selects a comparison base from the configured remote's default branch (then `main`/`master`,
local default branches, tracking upstream, or `HEAD` for a new repository). Use `--base REF`
when you want an explicit comparison target. A project profile may set `DEV_VALIDATE_BASE`
when a repository has a non-standard integration branch.

`dev-commit-push` runs the same precheck before creating the commit. A configured Git
identity therefore blocks commit/push when the repository author, remote alias, or SSH
key routing is inconsistent.

## Docker

`dev-docker-check` is an optional ROS1 Docker reproduction path. It has no baked-in
organization-specific image. Configure `DEV_DOCKER_IMAGE` in a project profile or pass
`--image` explicitly.

## Project profile configuration

Use `dev-profile-config` for machine/user-specific project profiles instead of committing real
paths to `profiles/` in this repository. Profiles are stored under
`${XDG_CONFIG_HOME:-$HOME/.config}/dev-workflow/profiles/` and selected with `dev-use`.

`dev-use` persists only the selected profile name. Other `dev-*` commands load that profile on
each invocation, so `DEV_REPO`, `DEV_ROS_WORKSPACE`, and ROS variables do not need to be exported
into the parent shell.

## Project profile configuration

Use `dev-profile-config` for machine/user-specific project profiles instead of committing real
paths to `profiles/` in this repository. Profiles are stored under
`${XDG_CONFIG_HOME:-$HOME/.config}/dev-workflow/profiles/` and selected with `dev-use`.

`dev-use` persists only the selected profile name. Other `dev-*` commands load that profile on
each invocation, so `DEV_REPO`, `DEV_ROS_WORKSPACE`, and ROS variables do not need to be exported
into the parent shell.
