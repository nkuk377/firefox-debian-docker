# Firefox Browser - Docker Image Based on Debian

## Description

This Docker image provides a lightweight and secure environment to run the **"Firefox Browser"** Based on **"Debian Linux"**, it is designed for users who prioritize speed and efficiency while maintaining privacy and security while browsing the web.

## Requirements:

- **Podman** (recommended) - In the .sh file below `podman` is used instead of `docker`.
>_Why Podman?: It offers a DAEMONLESS architecture and supports running containers WITHOUT root privileges, which can improve both security and system stability. Since its CLI is largely compatible with Docker, most commands can be used without modification._  
- **Docker** (compatible) - Ensure to replace `podman` with `docker` in the .sh file if you want to use it instead.  
- **X11 Server** -- You need to have an **X11** server running for the graphical interface.  

⚠️ _It does not work on **Wayland !**_

>_(to check what you using - run: `echo $XDG_SESSION_TYPE` on Linux)_    

## ⚠️ Before starting to set it up - please read "Security Considerations" section at the end of this page!  

---

## Usage:

👣 Create a folder **"firefox-debian"**  
👣 Create two files in that folder: **"Dockerfile"** & **"run-firefox.sh"** with the code below...

---

#### Dockerfile: _(use capital "D" here!)_

```
# Use the Debian Bookworm Slim base image
FROM debian:bookworm-slim

# Set environment variable to avoid interactive prompts during package installation
ENV DEBIAN_FRONTEND=noninteractive

# Update apt and install essential tools for debugging
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    apt-utils \
    ffmpeg \
    pulseaudio \
    sudo \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Install Firefox and other dependencies separately for better error tracking
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    firefox-esr \
    dbus-x11 \
    xvfb \
    xauth \
    bash \
    mesa-utils \
    libdbus-glib-1-2 && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Add a non-root user for running Firefox
RUN useradd -ms /bin/bash firefox
RUN chown -R firefox:firefox /home/firefox

# Add non-root user to the sudoers file for sudo access without a password
RUN echo "firefox ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers

# Switch to the torbrowser user
USER firefox
WORKDIR /home/firefox/firefox-browser

# Set the environment variable to forward the X display
ENV DISPLAY=:99

# Create a startup script to run Firefox with Xvfb
RUN echo -e '#!/bin/bash\n\
Xvfb :99 -screen 0 1280x1024x16 &\n\
firefox' > /home/firefox/start-firefox.sh

# Make the script executable
RUN chmod +x /home/firefox/start-firefox.sh

# Set the entrypoint to start the script
ENTRYPOINT ["/bin/bash", "/home/firefox/start-firefox.sh"]


```

---

#### run-firefox.sh:

```
#!/bin/bash

# Ensure script runs in the Dockerfile directory
cd "$(dirname "$0")"

# Build the Podman image for Firefox Browser on Debian Slim
podman build -t localhost/dockerfos/firefox-debian:latest .

# Allow the current user to access the X server
xhost +local:$(id -un)

# PulseAudio paths
PULSE_SERVER="unix:/run/user/$(id -u)/pulse/native"
PULSE_SOCKET="/run/user/$(id -u)/pulse/native"
PULSE_COOKIE="$HOME/.config/pulse/cookie"

# Run the Firefox container with X11, PulseAudio, and restricted network
echo "Running Firefox inside Podman..."
podman run -it --rm \
--network="slirp4netns:allow_host_loopback=false" \
-e DISPLAY="$DISPLAY" \
-e PULSE_SERVER="$PULSE_SERVER" \
-v /tmp/.X11-unix:/tmp/.X11-unix:rw \
-v "$PULSE_SOCKET:$PULSE_SOCKET:rw" \
-v "$PULSE_COOKIE:/home/firefox/.config/pulse/cookie:ro" \
localhost/dockerfos/firefox-debian:latest

# Revoke X server access after container stops
echo "Revoking Podman access to X11 server..."
xhost -local:$(id -un)
```

---

👣 Make **"run-firefox.sh"** executable:

```
chmod +x run-firefox.sh
```

👣 No need to build the image - just run the script:

```
./run-firefox.sh
```

---

## Key Features:

**You can interactively enter the running container:**

👣 Open a new terminal window or tab when container is running, keep the browser open and run:

`docker exec -it <container_name_or_id> bash`  

> _If you close the browser running from the container - the container process exites!_

- You can install additional packages as "sudo" is installed (no password needed)

> _You can do so adding the packages in the package list in the "Dockerfile" before run it!_

---

## Security Considerations:

- This container is designed to run in a sandboxed environment with minimal exposure to the host system.

- It's recommended to run this image using non-privileged user permissions and limit access to sensitive directories.

- Remember: The docker image behind the browser is running with sudo installed with no-password!

## Disclaimer:

This Docker image is provided **as-is**, without any warranties or guarantees of any kind. Use of this image is **entirely at your own risk**.
