# Development Workflow Setup Examples

This guide shows how to configure `dev_workflow` for four common project types without hard-coding any organization-specific or personal information.

The examples intentionally use generic repository names, Git identities, SSH aliases, paths, and Docker images. Replace them with values appropriate for your own machine.

## 1. One-time workflow installation

Assume the workflow repository is installed at:

```bash
$HOME/tools/dev_workflow
```

Add the following to `~/.bashrc`:

```bash
export DEV_WORKFLOW_HOME="$HOME/tools/dev_workflow"
export PATH="$DEV_WORKFLOW_HOME/bin:$PATH"
```

Reload the shell:

```bash
source ~/.bashrc
hash -r
```

Verify the installation:

```bash
echo "$DEV_WORKFLOW_HOME"

command -v dev-use
command -v dev-profile-config
command -v dev-git-config
command -v dev-git-check
command -v dev-git-update
command -v dev-ros-check
command -v dev-docker-check
command -v dev-precheck
command -v dev-validate
```

Project and Git identity configuration is stored outside the repository under:

```text
~/.config/dev-workflow/
```

This keeps user-specific paths, email addresses, SSH keys, and private project configuration out of the public workflow repository.

---

# 2. Configure Git identities

A project profile can reference a Git identity profile. This allows different repositories to use different authors and SSH keys without editing global Git state manually.

## Personal Git identity

Example:

```bash
dev-git-config set personal \
  --author-name "Developer Name" \
  --author-email "developer@example.com" \
  --ssh-alias github-personal \
  --ssh-host github.com \
  --ssh-key ~/.ssh/id_ed25519_github_personal
```

Select it:

```bash
dev-git-config use personal
dev-git-config show personal
```

Example SSH configuration:

```text
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github_personal
    IdentitiesOnly yes
```

Verify the effective SSH configuration:

```bash
ssh -G github-personal | grep -E \
'^(hostname|user|identityfile|identitiesonly) '
```

Test authentication:

```bash
ssh -T git@github-personal
```

A repository using this identity would normally use a remote such as:

```text
git@github-personal:my-user/my-project.git
```

## Work Git identity

Example:

```bash
dev-git-config set work \
  --author-name "Developer Name" \
  --author-email "developer@work.example" \
  --ssh-alias github-work \
  --ssh-host github.com \
  --ssh-key ~/.ssh/id_ed25519_github_work
```

Example SSH configuration:

```text
Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github_work
    IdentitiesOnly yes
```

A work repository would then use a remote such as:

```text
git@github-work:my-org/my-project.git
```

Validate any active project's Git identity with:

```bash
dev-git-check
```

If only the repository-local author or remote host is wrong:

```bash
dev-git-check --fix
```

The tool should not silently rewrite `~/.ssh/config`. SSH configuration remains an explicit user action.

---

# 3. Case A — Generic non-ROS repository

Use this pattern for a normal tooling, library, documentation, or application repository that does not require ROS or Docker for its main validation path.

Example repository:

```text
$HOME/projects/my_tool
```

Configure:

```bash
dev-profile-config set my_tool \
  --repo "$HOME/projects/my_tool" \
  --git-profile personal
```

Activate:

```bash
dev-use my_tool
dev-use --show
```

Validate Git:

```bash
dev-git-check
```

Run normal repository checks:

```bash
dev-hygiene --check
dev-format --check
dev-precheck --check-format
dev-validate
```

Safely update from the configured remote:

```bash
dev-git-update
```

Typical flow:

```text
dev-use my_tool
    ↓
dev-git-check
    ↓
dev-git-update
    ↓
dev-precheck --check-format
    ↓
dev-validate
```

---

# 4. Case B — Native ROS2 / colcon project

Example workspace:

```text
$HOME/workspaces/ros2_ws
```

Example repository:

```text
$HOME/workspaces/ros2_ws/src/my_ros2_project
```

Configure:

```bash
dev-profile-config set my_ros2_project \
  --repo "$HOME/workspaces/ros2_ws/src/my_ros2_project" \
  --workspace "$HOME/workspaces/ros2_ws" \
  --ros-version 2 \
  --ros-setup /opt/ros/humble/setup.bash \
  --builder auto \
  --git-profile personal
```

Activate:

```bash
dev-use my_ros2_project
dev-use --show
```

Expected profile characteristics:

```text
ROS version : 2
ROS setup   : /opt/ros/humble/setup.bash
Builder     : auto
Git profile : personal
```

Validate Git:

```bash
dev-git-check
```

Build:

```bash
dev-ros-check
```

Build and test:

```bash
dev-ros-check --test
```

Run the combined workflow:

```bash
dev-precheck --check-format
dev-validate
```

Safely update the repository:

```bash
dev-git-update
```

The ROS2 environment is sourced by the workflow for the check itself. The user does not need to manually run:

```bash
source /opt/ros/humble/setup.bash
```

before every validation.

---

# 5. Case C — Native ROS1 / catkin project

Example workspace:

```text
$HOME/workspaces/ros1_ws
```

Example repository:

```text
$HOME/workspaces/ros1_ws/src/my_ros1_project
```

Configure:

```bash
dev-profile-config set my_ros1_project \
  --repo "$HOME/workspaces/ros1_ws/src/my_ros1_project" \
  --workspace "$HOME/workspaces/ros1_ws" \
  --ros-version 1 \
  --ros-setup /opt/ros/noetic/setup.bash \
  --ros-package my_ros1_package \
  --builder auto \
  --git-profile work
```

Activate:

```bash
dev-use my_ros1_project
dev-use --show
```

Expected profile characteristics:

```text
ROS version : 1
ROS setup   : /opt/ros/noetic/setup.bash
ROS package : my_ros1_package
Builder     : auto
Git profile : work
```

Validate Git:

```bash
dev-git-check
```

Build:

```bash
dev-ros-check
```

Build and test:

```bash
dev-ros-check --test
```

Run the full local workflow:

```bash
dev-precheck --check-format
dev-validate
```

Safely update:

```bash
dev-git-update
```

As with ROS2, the ROS1 setup is sourced internally by the workflow. Manual shell sourcing is not required for every check.

---

# 6. Case D — Docker + ROS1/catkin project

Use this pattern when the authoritative build environment is a Docker image rather than the host machine.

Example repository:

```text
$HOME/projects/robot_stack
```

Example Docker image:

```text
example/robot-build-base:validated-v1
```

Configure:

```bash
dev-profile-config set robot_stack \
  --repo "$HOME/projects/robot_stack" \
  --builder catkin \
  --docker-image example/robot-build-base:validated-v1 \
  --git-profile work
```

If the project should normally validate a specific consumer package rather than the entire workspace, configure the package:

```bash
dev-profile-config set robot_stack \
  --repo "$HOME/projects/robot_stack" \
  --builder catkin \
  --docker-image example/robot-build-base:validated-v1 \
  --ros-package my_consumer_package \
  --git-profile work
```

Activate:

```bash
dev-use robot_stack
dev-use --show
dev-git-check
```

Verify that the Docker image exists:

```bash
docker image inspect example/robot-build-base:validated-v1 >/dev/null \
  && echo "PASS: Docker image available" \
  || echo "ERROR: Docker image missing"
```

## Full workspace build

If no ROS package is configured:

```bash
dev-docker-check
```

The effective build is:

```bash
catkin build \
  --no-status \
  --cmake-args -DCATKIN_ENABLE_TESTING=OFF
```

## Targeted package build

To build only the expected package and its catkin dependency closure:

```bash
dev-docker-check --package my_consumer_package
```

This is useful when the repository contains many packages but only one integration target needs validation.

## Build and test

For the configured target:

```bash
dev-docker-check --test
```

Or explicitly:

```bash
dev-docker-check \
  --package my_consumer_package \
  --test
```

The expected sequence is:

```text
catkin build <target>
    ↓
catkin run_tests <target>
    ↓
catkin_test_results
```

## Keep the Docker shell open

For debugging after a build/test run:

```bash
dev-docker-check \
  --package my_consumer_package \
  --test \
  --shell
```

This allows additional inspection inside the same container:

```bash
cd /ws

catkin build another_package
catkin run_tests another_package
catkin_test_results --verbose
```

Exit when finished:

```bash
exit
```

---

# 7. Switching between projects

Once the profiles are configured, switching should be simple.

ROS2 project:

```bash
dev-use my_ros2_project
dev-use --show
dev-git-check
dev-ros-check
```

ROS1 project:

```bash
dev-use my_ros1_project
dev-use --show
dev-git-check
dev-ros-check
```

Docker/catkin project:

```bash
dev-use robot_stack
dev-use --show
dev-git-check
dev-docker-check
```

Generic project:

```bash
dev-use my_tool
dev-use --show
dev-git-check
dev-precheck
```

The active profile is persisted by the workflow. The tool commands load the selected profile internally.

Do not expect commands such as:

```bash
echo "$DEV_REPO"
```

to necessarily reflect the active profile. The workflow intentionally avoids exporting all project settings into the parent shell.

---

# 8. Profile discovery

List configured user profiles:

```bash
dev-profile-config list
```

List all profiles visible to `dev-use`:

```bash
dev-use --list
```

Inspect a specific profile:

```bash
dev-profile-config show my_ros2_project
```

Inspect the active profile:

```bash
dev-use --show
```

User-created project profiles are stored under:

```text
~/.config/dev-workflow/profiles/
```

User-created Git profiles are stored under:

```text
~/.config/dev-workflow/git-profiles/
```

These files should not be committed to the public workflow repository.

---

# 9. Recommended validation sequence

For a normal native project:

```bash
dev-use <profile>

dev-git-check
dev-git-update

dev-hygiene --check
dev-format --check
dev-precheck --check-format
dev-validate
```

For a Docker-authoritative project:

```bash
dev-use <profile>

dev-git-check
dev-git-update

dev-hygiene --check
dev-format --check

dev-docker-check --package <target>
dev-docker-check --package <target> --test
```

Use `--shell` when interactive post-build inspection is useful:

```bash
dev-docker-check \
  --package <target> \
  --test \
  --shell
```

---

# 10. Summary of the four patterns

| Case | Environment | Build command | Git profile example |
|---|---|---|---|
| Generic repository | Host | repository-specific checks | `personal` |
| Native ROS2 | Host ROS2 | `colcon` through `dev-ros-check` | `personal` |
| Native ROS1 | Host ROS1 | `catkin` through `dev-ros-check` | `work` |
| Docker ROS1/catkin | Docker | `catkin build` through `dev-docker-check` | `work` |

The project profile determines how the repository is built and validated. The Git identity profile determines how it authenticates and authors commits. Keeping those concerns separate allows the same workflow installation to safely handle personal, work, ROS1, ROS2, and Docker-based repositories.
