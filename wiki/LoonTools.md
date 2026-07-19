# LoonTools

LoonTools is a wrapper for various commands
featuring diagnostics.

Usage: LoonTools [options]

Options:
    -d, --wtf, --doctor  Run the Loon Doctor diagnostics
    -mtu, --mtu     Check network MTU only
    --restart-zed   Attempt to bring the ZED daemon back online
    -c, --colcon    Build the colcon workspace (run inside the container)
    -h, --help      Show help
    -v, --verbose   Enable verbose mode, shows passing checks too
    --dry-run       Do not execute changes
    --version       print version
    --ignore-defensive-checks  Skip defensive checks (use with caution!) Allows building in unsupported environments

## Loon Doctor

`LoonTools --doctor` is the single health check for the stack. It replaces the
old `ros2 run loone doctor` entry point, so it covers both the host layer and
the ROS 2 runtime layer.

By default only checks that need attention are printed. Pass `-v` to see the
passing ones too. The command exits non-zero if any check fails, so it can be
used from CI and from other scripts.

### What it checks

| Layer | Checks |
| --- | --- |
| Environment | `RMW_IMPLEMENTATION`, `ROS_DOMAIN_ID`, `ROS_LOCALHOST_ONLY`, `ROS_DISTRO`, ZED/L4T versions, X11 forwarding vars |
| File structure | `LoonE_ws` workspace layout and the docker compose/Dockerfiles |
| Commands | Required tooling on PATH, plus optional tooling (`i2cdetect`, `adb`, `colcon`, `nvidia-smi`) |
| APT packages | Host build packages, Jetson/L4T packages, and the ROS 2 runtime packages the stack depends on |
| Python packages | `numpy` and the Adafruit motor/PCA9685/Blinka modules, checked by importing them |
| ROS nodes | `thrust_mixer`, `robot_state_publisher`, `joint_state_broadcaster`, `controller_manager`, plus the nav2, ZED and SLAM nodes |
| ROS topics | The `/cmd_vel` → `/asv_forward_controller/commands` → `/asv/joint_commands` chain, plus `/scan`, `/map`, odom and diagnostics |
| TF tree | `map` → `odom` → `zedx_camera_link` → `base_link` → prop/rudder links |
| Parameters | `cmd_timeout`, `prop_neutral` and `rudder_center` must agree between `thrust_mixer` and `pca9685_driver` |
| Network | Physical interfaces at MTU 9000, required for ZED X GMSL streaming |
| Hardware | I2C bus 1 and the PCA9685 at `0x40` |

### Reading the output

| Symbol | Meaning |
| --- | --- |
| `✓` PASS | Check succeeded. Hidden unless `-v` is passed. |
| `✗` FAIL | Something is broken. Sets the non-zero exit code. |
| `!` WARN | Degraded but not fatal, e.g. nav2 not brought up yet, or MTU below 9000. |
| `-` SKIP | Not applicable here, e.g. I2C checks off the SBC or ROS packages on the host. |

Layers are ordered outside in, so the first failure printed is usually the root
cause. In particular, a wrong `ROS_DOMAIN_ID` or `RMW_IMPLEMENTATION` makes
every node, topic and TF check fail at once — fix the environment variables
before reading anything below them.

The doctor never changes the system. If the ZED daemon is reported offline, use
`LoonTools --restart-zed` to act on it.
