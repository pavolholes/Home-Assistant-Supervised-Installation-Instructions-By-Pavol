# Home Assistant Supervised Installation Instructions By Pavol
I've collected below notes when I installed Home Assistant Supervised to my Dell OptiPlex 3050 with Intel CPU. I'm not trying to cover here any other platforms (like RPi).

Because my installation is completed and working, below instructions may age-out so you'll need to update the steps which doesn't work. 
Last update: November 2021

## Install Debian
- Download Debian 11 x64 ISO: https://cdimage.debian.org/debian-cd/current/amd64/iso-dvd/
- Use Etcher to flash ISO to USB thumb drive: https://www.balena.io/etcher/
- Boot, install (I don't have any Desktop environment on my host OS, I use only CLI commands to manage it)
 
## Basic settings of Debian
- Make your session root
  ```
  sudo su -
  ```
- Update to the latest version
  ```
  apt update
  apt upgrade
  ```

- Update firmware (drivers) for hardware
  - Edit sources list
    ```
    nano /etc/apt/sources.list
    ```
  - To lines ending at `main` add:
    ```
    ... contrib non-free
    ```
  - At the end add two new lines:
    ```
    deb http://ftp.us.debian.org/debian/ testing main non-free contrib
    deb-src http://ftp.us.debian.org/debian/ testing main non-free contrib
    ```
  - Exit nano with saving changes
  - Update firmware
    ```
    apt update
    apt install  firmware-linux-nonfre
    apt install  firmware-misc-nonfree
    update-initramfs –u
    reboot
    ```
- Enable `ll` and `l` aliases to `ls` command
  - Edit profile:
    ```
    nano ~/.bashrc
    ```
  - Uncomment below lines and edit the `ll` and `l` commands as seen below:
    ```
    export LS_OPTIONS='--color=auto'
    eval "$(dircolors)"
    alias ls='ls $LS_OPTIONS'
    alias ll='ls $LS_OPTIONS -lAhF'
    alias l='ls $LS_OPTIONS -lAhF'
    ```
  - Apply settings to the profile:
    ```
    . ~/.bashrc
    ```
- Install SSH for remote connections to Debian
  - Install OpenSSH server:
    ```
    apt install openssh-server
    systemctl status ssh
    ```
  - Get IP address by:
    ```
    nmcli –p device show
    ```
 
## Install Docker
  ```
  apt-get install curl
  curl -fsSL get.docker.com | CHANNEL=stable sh
  docker run hello-world
  docker login
  ```
 
## Install Home Assistant prerequisites
Check the latest instructions here: https://github.com/home-assistant/supervised-installer
  ```
  apt-get install jq wget curl udisks2 libglib2.0-bin network-manager dbus –y
  apt install apparmor -y
  ```

## Install Home Assistant OS-Agent
Check the latest instructions here: https://github.com/home-assistant/os-agent/tree/main#using-home-assistant-supervised-on-debian
  ```
  apt install libglib2.0-bin
  cd ~
  wget https://github.com/home-assistant/os-agent/releases/download/1.2.0/os-agent_1.2.0_linux_x86_64.deb
  dpkg -i os-agent_1.2.0_linux_x86_64.deb
  ```
Test installation by:
  ```
  gdbus introspect --system --dest io.hass.os --object-path /io/hass/os
  ```
 
## Install Home Assistant Supervised
  ```
  wget https://github.com/home-assistant/supervised-installer/releases/latest/download/homeassistant-supervised.deb
  dpkg -i homeassistant-supervised.deb
  ```
Press `y` to replace /etc/network/interfaces

## Install Portainer
  - Create folder PortainerConfig in some convenient location so you can easily backup it
    ```
    cd ~
    mkdir DockerVolumes
    cd DockerVolumes
    mkdir PortainerConfig    
    ```
  - Install Portainer
  
    a) with self-signed SSL cert:
      ```
      docker pull portainer/portainer-ce
      docker run -d -p 9000:9000 -p 9443:9443 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v ~/DockerVolumes/PortainerConfig:/data  portainer/portainer-ce
      ```
    b) with SSL cert from Home Assistant:
      ```
      docker pull portainer/portainer-ce
      docker run -d -p 9000:9000 -p 9443:9443 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v ~/DockerVolumes/PortainerConfig:/data -v /usr/share/hassio/ssl:/certs:ro portainer/portainer-ce --sslcert /certs/fullchain.pem --sslkey /certs/privkey.pem
      ```
  - If you'll need to update the Portainer later you can run these commands (update the `run` command to one of above to use or not the SSL cert from Home Assistant)
    ```
    docker stop portainer
    docker rm portainer
    docker pull portainer/portainer-ce
    docker run -d -p 9000:9000 -p 9443:9443 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v ~/DockerVolumes/PortainerConfig:/data -v /usr/share/hassio/ssl:/certs:ro portainer/portainer-ce --sslcert /certs/fullchain.pem --sslkey /certs/privkey.pem
    ```
  - Open Portainer web console at port 9443:
    ```
    https://<host-ip-address>:9443
    ```
    
## Install Cockpit for easier management of the Debian host OS
- Installation
  ```
  echo 'deb http://deb.debian.org/debian bullseye-backports main' > \
   /etc/apt/sources.list.d/backports.list
  apt update

  apt install -t bullseye-backports cockpit
  ```
  
- Connect to the Cockpit web console at port 9090:
  ```
  https://<host-ip-address>:9090
  ```

- Samba/NFS plugin
  ```
  wget -qO - http://images.45drives.com/repo/keys/aptpubkey.asc | apt-key add -
  curl -o /etc/apt/sources.list.d/45drives.list http://images.45drives.com/repo/debian/45drives.list
  apt update
  apt install cockpit-file-sharing
  ```
  
  - Configure `samba` for use of `Samba/NFS plugin`
    - Edit samba configuration file
      ```
      nano /ect/samba/smb.conf
      ```
    - Under `[global]` add these lines:
      ```
      # enable configuration from Cockpit
      include = registry
      # disable guest access
      restrict anonymous = 2
      # disable SMB1 and SMB2 for security purposes
      min protocol = SMB3
      ```
    - Find below line and comment it
      ```
      # Windows attempts to login with the current users as default.
      # If that doesn't work, it gets mapped as guest on the server side,
      # something that the latest Windows versions do not allow.
      # map to guest = bad user
      ```
    - Restart the samba service
      ```
      systemctl restart smbd.service
      ```

- Navigator plugin
  ```
  apt install cockpit-navigator
  ```

- Virtual Machines plugin
  ```
  apt install cockpit-machines
  ```
  - Fix for: ERROR Requested operation is not valid: network 'default' is not active
    ```
    systemctl restart libvirtd
    virsh net-start default
    ```
  - If you would like to use Bridge network in the VMs please let me know and I'll copy my notes to here. It's little complicated and took me several days to find the proper combination for the cooperation of `KVM`, `Home Assistant Supervised` and `Docker`.
