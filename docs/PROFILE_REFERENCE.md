# Profile reference

Project profiles are sourced shell files stored under `profiles/` in the external
dev-workflow repository. User-specific Git identity profiles are stored separately
under `~/.config/dev-workflow/git-profiles/`.

## Project profile settings

```bash
# Repository
DEV_REPO="$HOME/path/to/project"

# C/C++
DEV_CPP_FORMATTER="git-clang-format-14"
DEV_CPP_EXTENSIONS="c,cc,cpp,cxx,h,hh,hpp,hxx"

# Python
DEV_PYTHON_FORMATTER="darker"

# Optional standalone CMake tests
DEV_CMAKE_TEST_FLAG="-DBUILD_TESTING=ON"
DEV_GTEST_GLOB="*_tests"
DEV_TMP_BUILD_NAME="project_test_all"

# ROS
DEV_ROS_WORKSPACE="$HOME/ws"              # optional if repo is under ws/src
DEV_ROS_VERSION="2"                       # auto, 1, or 2
DEV_ROS_SETUP="/opt/ros/humble/setup.bash"
DEV_ROS_DISTRO="humble"                   # alternative to DEV_ROS_SETUP
DEV_ROS_PACKAGE="my_package"              # optional; empty = auto-discover
DEV_ROS_PACKAGES="pkg_a pkg_b"            # optional multi-package override
DEV_ROS_BUILDER="auto"                    # catkin/catkin_make/colcon

# Optional per-user Git identity reference
DEV_GIT_PROFILE="personal"

# Optional Docker reproduction image
DEV_DOCKER_IMAGE="your-registry/ros1-build-image:tag"
```

`DEV_ROS_SETUP` is the most deterministic choice on a machine carrying multiple ROS
distributions. The setup is sourced only inside the check process.

If package selection is empty, every `package.xml` under the repository is discovered.
If workspace is empty, the tool walks upward looking for the workspace whose `src`
directory contains the repository.

For a pure ROS repository, standalone CMake tests are skipped unless explicitly
configured. ROS package tests then run through `dev-ros-check` automatically. Set:

```bash
DEV_STANDALONE_TESTS=1
```

only when a ROS repository's top-level `CMakeLists.txt` is intentionally usable as a
standalone test build.

## Git identity profile settings

Create these with `dev-git-config set`; do not commit them to the repository.

```bash
DEV_GIT_AUTHOR_NAME="Jane Developer"
DEV_GIT_AUTHOR_EMAIL="jane@example.com"
DEV_GIT_SSH_ALIAS="github-personal"
DEV_GIT_SSH_HOST="github.com"
DEV_GIT_SSH_USER="git"
DEV_GIT_SSH_KEY="$HOME/.ssh/id_ed25519_personal"
DEV_GIT_REMOTE="origin"
```

The SSH alias is the host that should appear in the Git remote, while `DEV_GIT_SSH_HOST`
is the real service host configured in `~/.ssh/config`.

## Active project profile

`dev-use PROFILE` stores only the selected project profile name under the user's config
directory. It does not modify target repositories.

Project resolution priority:

1. command-line values such as `--repo`, `--workspace`, `--package`, `--builder`,
   `--ros-version`, or `--ros-setup`;
2. explicit `--profile`;
3. active profile selected with `dev-use`;
4. automatic repository/workspace/package/ROS detection.

Git identity resolution priority:

1. `--git-profile`;
2. `DEV_GIT_PROFILE` in the project profile;
3. active identity selected with `dev-git-config use`.
