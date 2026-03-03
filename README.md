# Android World — Guide

---

# Part 1: Installation (One-Time Setup)

## Step 1: System Dependencies

```bash
cd ~
sudo apt update
sudo apt install -y \
  openjdk-17-jdk qemu-kvm ffmpeg unzip wget curl \
  libnss3 libasound2t64 xvfb x11vnc novnc

# KVM permissions (required on Ubuntu 24)
sudo usermod -aG kvm $USER
newgrp kvm

# Fix X11 socket directory
sudo mkdir -p /tmp/.X11-unix
sudo chmod 1777 /tmp/.X11-unix
```

## Step 2: Android SDK & Emulator

```bash
export ANDROID_SDK_ROOT=$HOME/.local/share/android/sdk
mkdir -p $ANDROID_SDK_ROOT/cmdline-tools

wget https://dl.google.com/android/repository/commandlinetools-linux-10406996_latest.zip
unzip commandlinetools-linux-10406996_latest.zip -d $ANDROID_SDK_ROOT/cmdline-tools
mv $ANDROID_SDK_ROOT/cmdline-tools/cmdline-tools $ANDROID_SDK_ROOT/cmdline-tools/latest
```

### Set Environment Variables

```bash
cat >> ~/.bashrc << 'EOF'
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export ANDROID_SDK_ROOT=$HOME/.local/share/android/sdk
export PATH=$PATH:$ANDROID_SDK_ROOT/cmdline-tools/latest/bin:$ANDROID_SDK_ROOT/platform-tools:$ANDROID_SDK_ROOT/emulator
EOF
source ~/.bashrc
```

### Install SDK Components & Create AVD

```bash
sdkmanager --licenses
#sdkmanager "system-images;android-34;google_apis;x86_64"
sdkmanager "emulator" "platform-tools" "system-images;android-33;google_apis;x86_64"

avdmanager create avd --force \
  --name AndroidWorldAvd \
  --device "pixel_6" \
  --package "system-images;android-33;google_apis;x86_64"
```

## Step 3: Install Android World

```bash
conda create -n android_world python=3.11.8
conda activate android_world

git clone https://github.com/google-research/android_world.git
cd android_world

pip install -r requirements.txt
python setup.py install
```

## Step 4: Create Startup Script

```bash
cat > ~/start_emulator.sh << 'EOF'
#!/bin/bash
export DISPLAY=:99
Xvfb :99 -screen 0 1280x800x24 &
sleep 1
x11vnc -display :99 -forever -nopw -listen 0.0.0.0 -rfbport 5900 &
websockify --web /usr/share/novnc 6080 localhost:5900 &
sleep 1
$ANDROID_SDK_ROOT/emulator/emulator \
  -avd AndroidWorldAvd \
  -no-snapshot -no-boot-anim \
  -gpu swiftshader_indirect \
  -grpc 8554 &
EOF
chmod +x ~/start_emulator.sh
```

---

# Part 2: Restart & Run (Every Session)

## On the Remote Server

```bash
# 1. Start emulator + VNC
~/start_emulator.sh

# 2. Wait for emulator to boot (~30s), then verify
adb devices

# 3. Activate environment and run
conda activate android_world
cd /mnt/nvme0n1/ruofan/git_space/android_world
```

## On Your Local Device (to view emulator)

```bash
# SSH with port forwarding
ssh -p <port to connect remote host, if any> -L 5900:localhost:5900 -L 6080:localhost:6080 <remote-username>@<remote-ip>
```

Open browser: http://localhost:6080/vnc.html

## Minimal Task
Run the `minimal_task_runner.py` script to see the basic mechanics of
AndroidWorld components. It initializes the environment, sets up a task, and
runs the default agent, M3A, on it.
```bash
python minimal_task_runner.py --task=ContactsAddContact
```

If you don't specify a task, a random task will be selected. *NOTE: If you want
to try open-source apps, i.e. not included with Android OS, please run
`--perform_emulator_setup` in the script below.*

## Run the benchmark

Note: **Task Step Limits Update**
As of 11/18/2024, the max_steps/step_budget for each task in AndroidWorld have been updated to approximately
**2x the human average completion time**. 
This adjustment ensures agents have sufficient time to complete tasks, 
while also reducing overhead of running thebenchmark. 
[Here](https://docs.google.com/spreadsheets/d/1KF-vY0Uy47o0mnursvs-HmS6hreU6U3rPrAjgEfjMK4/edit?usp=sharing) are the per-task updates.

```bash
python run.py \
  --suite_family=android_world \
  --output_path android_world/runs \
  --agent_name=m3a_gpt4v \
  --tasks=RecipeAddMultipleRecipesFromMarkor,SimpleSmsSendClipboardContent \  # Optional: Just run on a subset.
```

**Note:
The first time you run this script, you must install the necessary apps and set
permissions by specifying `--perform_emulator_setup`. 
This is a one-time setup. It may take several minutes depending on the connection speed.**

Above we specify the optional `--tasks` flag to run on a subset of tasks. Leave
it empty to run on the entire AndroidWorld suite.

The `n_task_combinations` argument specifies how many parameter permutations to
use for each task. 
For example, for an SMS task, it would correspond to different phone number/message combinations for each run.

If a run fails part-way through, you can resume it by re-running the script with
the `--checkpoint_dir` flag pointing to the output directory from the original run.

## Running MiniWoB++ tasks
To run the MiniWoB++ web-based tasks in AndroidWorld, simply set
`--suite_family=miniwob` and `--perform_emulator_setup` in the command above.

A key advantage of running MiniWoB++ tasks is that common input elements are
rendered as native, commonly used Android UI widgets, rather than as HTML. Thus
agents must learn to use universal widgets such as time- and date-pickers:
<p align="center">
   <img src="assets/miniwob.png" style="width:30%">
</p>

# Part 3: Stop 
Remember to clean up everything
```bash
./stop_emulator.sh
```


---

# Troubleshooting

### `ERROR | adb protocol fault`

```bash
pkill -9 qemu
pkill -9 emulator
adb kill-server
adb start-server
# Then re-run: ~/start_emulator.sh
```

### Proxy errors (`cannot parse value of 'http_proxy'`)

```bash
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890

# Or disable entirely:
unset http_proxy https_proxy
```

### `ModuleNotFoundError: No module named 'pkg_resources'`

In `setup.py`, replace:

```python
# Before
import pkg_resources
grpc_protos_include = pkg_resources.resource_filename('grpc_tools', '_proto')

# After
import grpc_tools, os
grpc_protos_include = os.path.join(os.path.dirname(grpc_tools.__file__), '_proto')
```