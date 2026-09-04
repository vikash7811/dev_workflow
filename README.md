# Dev Workflow

Portable profile-based development tooling for Git, CMake, ROS1, and ROS2 repositories.

The toolkit lives outside project repositories. Project profiles select repositories,
workspaces, ROS setup, optional test settings, and an optional Git identity profile.
User-specific Git names, emails, SSH aliases, and key paths are stored separately under
`~/.config/dev-workflow`.

## Layout

```text
~/dev_workflow/
├── bin/
├── profiles/
│   └── example.conf
├── docs/
└── README.md
```

## First-time setup

```bash
git clone <your-dev-workflow-repository> ~/dev_workflow

export DEV_WORKFLOW_HOME="$HOME/dev_workflow"
export PATH="$DEV_WORKFLOW_HOME/bin:$PATH"
hash -r
```

Add the exports to `~/.bashrc`:

```bash
export DEV_WORKFLOW_HOME="$HOME/dev_workflow"
export PATH="$DEV_WORKFLOW_HOME/bin:$PATH"
```

Reload and verify:

```bash
source ~/.bashrc

echo "$DEV_WORKFLOW_HOME"
command -v dev-use
command -v dev-precheck
command -v dev-git-check
dev-use --list
```

Expected path resolution for this layout:

```text
DEV_WORKFLOW_HOME=/home/<user>/dev_workflow
command -v dev-use -> /home/<user>/dev_workflow/bin/dev-use
```

## Configure a project profile

Project profiles are user-specific and should normally live outside this repository under:

```text
${XDG_CONFIG_HOME:-$HOME/.config}/dev-workflow/profiles/
```

Create or update them with `dev-profile-config`:

```bash
dev-profile-config set my_project \
  --repo ~/ws/src/my_project \
  --workspace ~/ws \
  --ros-version 2 \
  --ros-setup /opt/ros/humble/setup.bash \
  --git-profile personal

dev-profile-config list
dev-profile-config show my_project
```

Select it once:

```bash
dev-use my_project
dev-use --show
```

`profiles/example.conf` is a reference template only. Keeping real project paths in
per-user configuration prevents machine-specific or private paths from leaking into the
public dev-workflow repository.

The selected project profile name is stored at:

```text
${XDG_CONFIG_HOME:-$HOME/.config}/dev-workflow/active_profile
```

## Configure Git identities

Git identities are deliberately user-specific and are not committed to the workflow
repository.

Create one:

```bash
dev-git-config set personal \
  --author-name "Jane Developer" \
  --author-email jane@example.com \
  --ssh-alias github-personal \
  --ssh-host github.com \
  --ssh-key ~/.ssh/id_ed25519_personal
```

Select it as the user default:

```bash
dev-git-config use personal
dev-git-config show
```

Identity profiles are stored at:

```text
${XDG_CONFIG_HOME:-$HOME/.config}/dev-workflow/git-profiles/
```

A project can pin an identity with:

```bash
DEV_GIT_PROFILE="personal"
```

Resolution order is:

```text
--git-profile
    ↓
project DEV_GIT_PROFILE
    ↓
active identity selected by dev-git-config use
```

### Configure the SSH alias

Generate the matching `~/.ssh/config` stanza:

```bash
dev-git-config ssh-snippet personal
```

Example output:

```text
Host github-personal
    HostName github.com
    User git
    IdentityFile /home/<user>/.ssh/id_ed25519_personal
    IdentitiesOnly yes
```

Add that stanza to `~/.ssh/config`, then validate a repository:

```bash
dev-git-check --repo ~/ws/src/my_project --git-profile personal
```

If the repository-local author is wrong:

```bash
dev-git-check --repo ~/ws/src/my_project --git-profile personal --fix-author
```

If `origin` points directly at the service host instead of the configured SSH alias:

```bash
dev-git-check --repo ~/ws/src/my_project --git-profile personal --fix-remote
```

`--fix-remote` preserves the repository path and changes only the SSH routing needed to
use the configured identity. It refuses unrelated remote hosts. `--fix-author` updates
only repository-local `user.name` and `user.email`. `dev-git-check` never edits
`~/.ssh/config`.

## Safe Git update workflow

Update the active project's current branch safely:

```bash
dev-git-update
```

The command:

1. validates the configured Git identity when one is configured;
2. requires a clean worktree/index;
3. fetches the selected remote with pruning;
4. fast-forwards when possible;
5. refuses diverged history by default.

If you deliberately want to rebase a diverged local branch:

```bash
dev-git-update --rebase
```

## Automatic ROS1 / ROS2 checks

`dev-precheck` detects repositories containing `package.xml` and runs the appropriate
ROS check automatically.

ROS version can be inferred from package metadata or pinned with `DEV_ROS_VERSION`.
Supported builders are:

```text
ROS1: catkin, catkin_make
ROS2: colcon
```

Workspace detection walks upward from the repository and finds the containing
`WORKSPACE/src`. Packages are read from the profile when configured; otherwise all ROS
packages inside the repository are discovered automatically.

Examples:

```bash
# ROS1 project
dev-use ros1_project
dev-precheck

# ROS2 project
dev-use ros2_project
dev-precheck
```

The configured ROS setup is sourced inside the check process, so switching project
profiles does not require manually sourcing a different ROS distribution in the parent
shell.

Use `--no-ros` only when intentionally suppressing the automatic ROS build:

```bash
dev-precheck --no-ros
```

Use `--ros-test` to force ROS package tests when standalone tests are also configured:

```bash
dev-precheck --ros-test
```

For pure ROS repositories without a standalone top-level CMake test path,
`dev-precheck` automatically runs package tests through catkin/colcon.

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

When a Git identity is configured, `dev-precheck`, `dev-validate`, `dev-git-update`, and
`dev-commit-push` validate it before repository operations continue.

## Commands

| Command | Purpose |
|---|---|
| `dev-use` | Select/show/switch the active project profile |
| `dev-profile-config` | Create/update/list user-local project profiles |
| `dev-git-config` | Create/select/show user Git identity profiles |
| `dev-git-check` | Validate Git author, remote SSH alias, and configured key |
| `dev-git-update` | Fetch and safely fast-forward/rebase a branch |
| `dev-format` | Format/check changed C/C++ and Python lines |
| `dev-test` | Clean temporary standalone CMake build + CTest + discovered GTests |
| `dev-hygiene` | Remove Python caches and verify Git/source hygiene |
| `dev-precheck` | Normal pre-commit gate, including Git and automatic ROS checks |
| `dev-validate` | Validate complete branch delta from a Git base |
| `dev-ros-check` | Auto-detected ROS1/ROS2 package-scoped build/test |
| `dev-docker-check` | Optional configured ROS1 Docker catkin build/test |
| `dev-commit-push` | Validate staged changes, commit, and push |

## One-off overrides

Explicit values override active profiles:

```bash
dev-precheck --repo ~/other_ws/src/project --git-profile work
dev-ros-check --workspace ~/other_ws --package another_package
dev-ros-check --ros-version 2 --ros-distro humble
dev-ros-check --ros-setup /opt/ros/humble/setup.bash
```

The Docker check intentionally has no organization-specific default image. Configure
one per project or pass it explicitly:

```bash
DEV_DOCKER_IMAGE="your-registry/ros1-build-image:tag"
# or
dev-docker-check --image your-registry/ros1-build-image:tag
```

See `docs/WORKFLOW.md` and `docs/PROFILE_REFERENCE.md` for details.
