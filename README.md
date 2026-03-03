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
sdkmanager "emulator" "platform-tools" "system-images;android-34;google_apis;x86_64"

avdmanager create avd --force \
  --name AndroidWorldAvd \
  --device "pixel_6" \
  --package "system-images;android-34;google_apis;x86_64"
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