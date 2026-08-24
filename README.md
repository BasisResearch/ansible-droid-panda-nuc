# Ansible playbook for droid NUC
The repository contains the ansible playbook that sets up polymetis on an NUC so it can be used to control the robot. It can be rerun to pull from the latest [droid](https://github.com/BasisResearch/droid) repo, rebuild the image, and restart the containers. The playbook assumes that the host runs AlmaLinux 10. It should be compatible with any RHEL and RockyLinux as well. The security support of AlmaLinux 10 should last until May 31, 2035 so you don't need to update unless you want to use a newer, faster kernel.

## Preparing the machine

### Installing the operating system
Before you run the playbook, you need to first install the operating system. To do that, go to [Almalinux](https://almalinux.org/)'s official site and download an ISO image for AlmaLinux 10. There are multiple image options. At the time of the writing, you can choose a full-sized DVD, a small Boot image, or a Minimal image. I recommend either downloading the DVD or Boot image.

The installation should be straightforward. You should be able to just keep clicking next. If there's an existing Windows installation on the NUC, simply wipe it completely and reuse the space for the Linux installation.

For the network access, you could connect to an ethernet cable to a NUC, but for this guide, I'll assume that you are using wireless access for Internet and reserving the ethernet port for communication with the robot. (It's possible to do everything through the ethernet port, but it'll require slightly more complicated network setup, which I won't go over)

By default, the AlmaLinux installer will prompt you to create a user that lives in the `wheel` group, which lets you run `sudo` when you logged in as the user. The root user will be locked, which is fine to keep as is.


### Configuring ssh accesss for ansible
Ansible is a tool that automates configuration. You install it on your laptop, setup ssh access on the NUC, and then the ansible runtime will ssh into the NUC machine for you and perform the configurations specified in `playbook.yaml`. For this section, make sure your NUC is connected to an external monitor and keyboard.
Good news is that `ssh` is installed by default on AlmaLinux, but you need to make the NUC's IP reachable from your laptop for it to work.

First, you want your admin account to be able to run `sudo` without a password. To do so, run `visudo` (possibly with `sudo`) and then add the following line after replacing `USER` with your user name, which you can retrieve with `whoami`:
```
USER ALL=(ALL:ALL) NOPASSWD: ALL
```


Next, you want to make sure your laptop is on the same network as the NUC, so it can ssh into it. If your laptop and the NUC are on the same network, you might be able to `ssh` into the laptop directly. However, WiFi access points very often block communication between devices so it's likely not going to work. Also, since we need to configure the ethernet card to a local IP address to communicate with the robot anyways, we'll configure a local IP for the ethernet port and the laptop and use `ssh` on that local IP. I won't go into details, but you should manually configure the IP of the wired ethernet card to `172.16.0.2/24` and your own laptop's ethernet card (or external dongle) to `172.16.0.7/24` (or anything of the form 172.16.0.x where `x` is not `2`). After that, try `ssh YOUR-USER-NAME@172.16.0.2`. If that works, run the same command with `ssh` replaced by `ssh-copy-id` so you can later `ssh` into the machine without a password. Note that you might also want to set up `ssh-add` to cache your local `ssh` key so you don't have to type another passphrase to unlock it.

If everything works, you should be able to `ssh` into the NUC without any passwords.

### Configuring the firewall
The `ssh` port should be opened by default so you don't need to do anything with the firewall. However, since the communication test script involves the robot controller sending an unsolicited packet, it's most convenient to set up the firewall so the subnet (`172.16.0.0/24`) is trusted and all traffic is let through.

To do that, run `ip link show`, and find a device name that starts with `enp*`. On the Intel NUC, the device is named `enp89s0` but it might be something else if you buy a different brand of NUC or have a newer version of Linux. (The one that starts with `w` is likely the WLAN device that you should leave alone)

Next, run the following command to add the device to the trusted zone after replacing `enp89s0` with your actual device name.
```sh
sudo firewall-cmd --zone=trusted --change-interface=enp89s0
```


## Running the ansible playbook
Before you can run the ansible playbook, create a file named `inventory.ini` in the project root with the following content.
```ini
[nuc]
172.16.0.2 ansible_user=USERNAME
```
Replace USERNAME with your user name. The file specifies the NUC's IP address (attached to the ethernet port), and the user you want to run as on the remote machine.

Next, turned on the NUC, and connected your laptop to the NUC, and run `ping 172.16.0.2` to make sure your machine can talk to the NUC. Next, run `uv sync` to install ansible, and run `uv run ansible-playbook -i inventory.ini playbook.yaml` to actually configure the NUC. The script may fail due to temporary networking issues. If that happens, simply rerun the previous command and see if it resolves. If not, create an issue, post the error messages, and ping me.

The playbook will build a docker image, so it'll take a while for the build to finish.


## Validating the setup
The validation setup doesn't include the full details about turning on the robot and unlocking it. For more detailed instructions, refer to the notion documents.

### Making sure the realtime kernel is running
The playbook might fail to switch to the realtime kernel as the default. To see if the realtime kernel is running, run `uname -a` and see if `PREEMPT_RT` is part of the output. If not, refer to [RHEL's official documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux_for_real_time/10/html/installing_rhel_for_real_time/specifying-the-rhel-kernel-to-run) to configure the default kernel.

### Checking for listened ports
SSH into the machine and run `ss -tulpn`. You should be able to see the port `4242` being listened. If the robot is turned on, you should be able to see ports `50051` and `50052` as well.


### Communication test
Before you run the communication test, run `sudo systemctl stop droid-robot droid-gripper` to stop the robot arm and gripper instance.
Turn on the robot, and then run the following command on the NUC:
```
sudo podman exec -it polymetis /app/droid/fairo/polymetis/polymetis/src/clients/franka_panda_client/third_party/libfranka/build/examples/communication_test 172.16.0.4
```
Replace `172.16.0.4` with the actual robot controller IP address if you changed it through the controller GUI interface.


### Droid test
This is a more thorough test that drives the robot arm and gripper through the droid library. You can do this either on the NUC or on the laptop. The latter is recommended as in a realistic workflow you would talk to the NUC remotely from a beefier machine that's capable of learning and perception. Make sure you turn on the robot and the

On your laptop, clone [droid](https://github.com/BasisResearch/droid) and check out the `docker` branch. First, enter the repo root and run `uv sync`. Next, create a file with the following content.
```
"""Move the Franka Panda to the DROID home (reset) pose via the NUC server."""
import argparse
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
Save the script as `test.py` and then run `uv run python test.py`. If everything works correctly, the gripper should first open and then close while moving to its home position.
