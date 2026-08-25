# Ansible playbook for droid NUC
This repository contains the Ansible playbook that sets up polymetis on a NUC so it can be used to control the robot. It can be rerun to pull the latest [droid](https://github.com/BasisResearch/droid) repo, rebuild the image, and restart the containers (services are only restarted when something actually changed). The playbook assumes the host runs AlmaLinux 10; it should also work on RHEL 10 and Rocky Linux 10. AlmaLinux 10 has security support until May 31, 2035, so you don't need to upgrade unless you want a newer, faster kernel.

The ansible playbook has been tested on a freshly installed AlmaLinux 10 VM and an NUC with 8 cores.

## Preparing the machine

### Installing the operating system
Download an AlmaLinux 10 ISO from the [official site](https://almalinux.org/) — the DVD or the Boot image are both fine — and install it. The installation should be straightforward: you can mostly keep clicking next. If there's an existing Windows installation on the NUC, wipe it completely and reuse the space.

For network access, this guide assumes you use WiFi for Internet and reserve the ethernet port for communication with the robot. (It's possible to do everything through the ethernet port, but that requires a slightly more complicated network setup, which I won't go over.)

The installer will prompt you to create a user in the `wheel` group, which lets you run `sudo`. The root account is locked by default, which is fine to keep as is.

### Configuring ssh access for Ansible
Ansible is a tool that automates configuration. You install it on your laptop, set up ssh access on the NUC, and the Ansible runtime sshes into the NUC and performs the configuration specified in `playbook.yaml`. For this section, connect the NUC to a monitor and keyboard.

First, let your admin account run `sudo` without a password: run `sudo visudo` and add the following line, replacing `USER` with your user name (see `whoami`):
```
USER ALL=(ALL:ALL) NOPASSWD: ALL
```

Next, your laptop needs to reach the NUC over the network. WiFi access points very often block communication between devices, and the NUC's ethernet card needs a static local IP to talk to the robot anyway, so we use the wired link for ssh as well: manually configure the NUC's wired ethernet interface to `172.16.0.2/24` and your laptop's ethernet card (or dongle) to `172.16.0.7/24` (anything of the form `172.16.0.x` where `x` is not `2` or `4` works — `4` is the robot controller). Then try `ssh YOUR-USER-NAME@172.16.0.2`. Once that works, run the same command with `ssh` replaced by `ssh-copy-id` so you can log in without a password. You might also want to use `ssh-add` so you don't have to retype your key's passphrase.

## Running the ansible playbook
Create a file named `inventory.ini` in the project root, replacing `USERNAME` with your user name on the NUC:
```ini
[nuc]
172.16.0.2 ansible_user=USERNAME
```

Turn on the NUC, connect your laptop, and run `ping 172.16.0.2` to make sure the two machines can talk. Then run `uv sync` to install Ansible, and run
```
uv run ansible-playbook playbook.yaml
```
to configure the NUC. The first run builds a container image, so it will take a while. If the playbook fails due to a transient network issue, simply rerun it. If the failure persists, create an issue with the error messages and ping me at `yiyun@basis.ai`.

## Updating the docker image
If you ever make changes to the `docker` branch of the Basis Droid fork, you can make the change available on the NUC by rerunning `uv run ansible-playbook playbook.yaml`. It'll automatically pull the latest commit and build an up-to-date image.

Note: You should never make any changes to the checked out repo `/var/lib/droid` on NUC manually. There's nothing morally wrong with doing that, but it's better practice to keep the changes visible as commits and apply them wholesale on the NUC through the ansible automation script.

## Validating the setup
This section doesn't cover turning on and unlocking the robot. For detailed instructions, refer to the Notion documents. Note that the NUC now works as an appliance in the sense that you probably never need to ssh into it if it works properly. If the robot settings change and the robot arm and gripper control process crash, the service is configured to autostart after a few seconds. You only need to log in to validate the setup

### Making sure the realtime kernel is running
The playbook might fail to make the realtime kernel the default. Run `uname -a` and check that `PREEMPT_RT` appears in the output. If not, refer to [RHEL's official documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux_for_real_time/10/html/installing_rhel_for_real_time/specifying-the-rhel-kernel-to-run) to configure the default kernel.

### Checking for listened ports
SSH into the NUC and run `ss -tulpn`. Port `4242` should be listening; with the robot turned on, ports `50051` and `50052` should appear as well.

### Communication test
First stop the arm and gripper services: `sudo systemctl stop droid-robot droid-gripper`. Turn on the robot, then run the following on the NUC:
```
sudo podman exec -it polymetis /app/droid/fairo/polymetis/polymetis/src/clients/franka_panda_client/third_party/libfranka/build/examples/communication_test 172.16.0.4
```
Replace `172.16.0.4` with the actual controller IP if you changed it through the controller's GUI. I was able to get 5-20ish dropped packets per run with close to 98% to 99% success rate of executing commands. If too many packets get dropped, the communication test will complain.

Once you are done with the communication test, make sure that you restart both `droid-robot` and `droid-gripper` for more in-depth tests:
```sh
sudo systemctl start droid-robot droid-gripper
```

### Droid test
This is a more thorough test that drives the robot arm and gripper through the droid library. You can run it on the NUC or on your laptop; the latter is recommended, since in a realistic workflow you talk to the NUC remotely from a beefier machine that handles learning and perception. Make sure the robot is turned on and the NUC services are running.

On your laptop, clone [droid](https://github.com/BasisResearch/droid), check out the `docker` branch, and run `uv sync --extra dm-robotics` in the repo root. Then save the following as `test.py`:
```python
"""Move the Franka Panda to the DROID home (reset) pose via the NUC server."""
import numpy as np
import time

from droid.misc.parameters import nuc_ip
from droid.misc.server_interface import ServerInterface

# Same home pose as RobotEnv.reset_joints in droid/robot_env.py
HOME_JOINTS = np.array([0, -np.pi / 5, 0, -4 * np.pi / 5, 0, 3 * np.pi / 5, 1.0])


def main():
    print(f"Connecting to NUC at {nuc_ip}...")
    robot = ServerInterface(ip_address=nuc_ip, launch=False)

    try:
        print("Current joint positions:", robot.get_joint_positions())
    except Exception as e:
        print("Binding handles...")
        robot.launch_robot()
        print("Current joint positions:", robot.get_joint_positions())

    print("Opening gripper...")
    robot.update_gripper(0, velocity=False, blocking=True)
    time.sleep(1)
    print("Closing gripper...")
    robot.update_gripper(1, velocity=False, blocking=True)

    print("Moving to home pose...")
    robot.update_joints(HOME_JOINTS, velocity=False, blocking=True)

    print("Done. Joint positions:", robot.get_joint_positions())
    print("End-effector pose:", robot.get_ee_pose())


if __name__ == "__main__":
    main()
```
Run it with `uv run python test.py`. If everything works, the gripper opens and then closes while the arm moves to its home position.

## Debugging
The polymetis server (`polymetis`), the robot arm (`droid-robot`), and the gripper control (`droid-gripper`) each run as their own services. You can, for example, run `systemctl status polymetis` to query the status of the polymetis server. You can run `journalctl -u polymetis -u droid-robot -u droid-gripper -f` to follow the live status of all three services, and you can remove the `-u SERVICE` for the service that you are not interested in. You can also remove the `-f` to see the complete journal, including the past runs. If you append `--invocation=0`, you'll be able to view only the logs from the most recent invocations.
