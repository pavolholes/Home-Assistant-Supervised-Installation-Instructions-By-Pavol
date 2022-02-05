# Home Assistant Supervised Installation Instructions By Pavol
I've collected below notes when I installed Home Assistant Supervised to my Dell OptiPlex 3050 with Intel CPU. I'm not trying to cover here any other platforms (like RPi).

Because my installation is completed and working, below instructions may age-out so you'll need to update the steps which doesn't work. 
Last update: November 2021

## Install Debian
- Download Debian 11 x64 ISO: https://cdimage.debian.org/debian-cd/current/amd64/iso-dvd/
- Use Etcher to flash ISO to USB thumb drive: https://www.balena.io/etcher/
- Boot, install (I don't have any Desktop environment on my host OS, I use only CLI commands to manage it)
 
## Basic settings of Debian

### Enable sudo for `<user>`
  - login as `root` user and run below command to add `<userName>` to sudoers
    ```
    adduser <userName> sudo
    ```
  - logout from `root` user
    ```
    exit
    ```
  - login as `<user>`

### Make your session root
  ```
  sudo su -
  ```

### Update to the latest version
  ```
  apt update
  apt upgrade
  ```

### Update firmware (drivers) for hardware
  - Edit sources list
    ```
    nano /etc/apt/sources.list
    ```
  - To lines ending at `main` add:
    ```
    ... contrib non-free
    ```
  - Exit nano with saving changes by pressing `ctrl+c` > `y` > `enter`
  - Update firmware
    ```
    apt update
    apt install firmware-linux-nonfree
    apt install firmware-misc-nonfree
    update-initramfs –u
    reboot
    ```

### Make your session root again after reboot
  ```
  sudo su -
  ```

### Enable `ll` and `l` aliases to `ls` command
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

### Install SSH for remote connections to Debian
  - Install OpenSSH server:
    ```
    apt install openssh-server
    systemctl status ssh
    ```
  - Get IP address so you can connect by SSH:
    ```
    ip a
    ```
 
## Install Docker
  ```
  apt install ca-certificates curl gnupg lsb-release
  curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  apt update
  apt install docker-ce docker-ce-cli containerd.io
  docker run hello-world
  docker login
  ```
 
## Install Home Assistant Supervised prerequisites
Check the latest instructions here: https://github.com/home-assistant/supervised-installer
  ```
  apt install jq wget curl udisks2 libglib2.0-bin network-manager dbus apparmor
  ```

## Install Home Assistant OS-Agent
Check the latest instructions here: https://github.com/home-assistant/os-agent/tree/main#using-home-assistant-supervised-on-debian
  ```
  cd ~
  wget https://github.com/home-assistant/os-agent/releases/download/1.2.2/os-agent_1.2.2_linux_x86_64.deb
  dpkg -i os-agent_1.2.2_linux_x86_64.deb
  ```

Test the installation by:
  ```
  gdbus introspect --system --dest io.hass.os --object-path /io/hass/os
  ```
 
## Install Home Assistant Supervised
  ```
  cd ~
  wget https://github.com/home-assistant/supervised-installer/releases/latest/download/homeassistant-supervised.deb
  dpkg -i homeassistant-supervised.deb
  ```

## Open Home Assistant
  ```
  http://<host-ip-address>:8123
  ```

## Home Assistant data folders location
  ```
  /usr/share/hassio
  ```

## Install Portainer
### Create folder PortainerConfig in some convenient location so you can easily backup it
  ```
  cd ~
  mkdir DockerVolumes
  cd DockerVolumes
  mkdir PortainerConfig
  ```

### Install Portainer with a) self signed SSL certificate or b) with SSL certificate from Home Assistant
  
  a) with self-signed SSL cert:
  
  ```
  docker pull portainer/portainer-ce
  docker run -d -p 9443:9443 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v ~/DockerVolumes/PortainerConfig:/data  portainer/portainer-ce
  ```

  b) with SSL cert from Home Assistant:
  
  ```
  docker pull portainer/portainer-ce
  docker run -d -p 9443:9443 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v ~/DockerVolumes/PortainerConfig:/data -v /usr/share/hassio/ssl:/certs:ro portainer/portainer-ce --sslcert /certs/fullchain.pem --sslkey /certs/privkey.pem
  ```

If you'll need to update the Portainer later you can run these commands (update the `run` command to one of above to use or not the SSL cert from Home Assistant)
  ```
  docker stop portainer
  docker rm portainer
  docker pull portainer/portainer-ce
  docker run -d -p 9443:9443 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v ~/DockerVolumes/PortainerConfig:/data -v /usr/share/hassio/ssl:/certs:ro portainer/portainer-ce --sslcert /certs/fullchain.pem --sslkey /certs/privkey.pem
  ```

### Open Portainer web console at port 9443:
  ```
  https://<host-ip-address>:9443
  ```
    
## Install Cockpit for easier management of the Debian host OS
Installation
  ```
  echo 'deb http://deb.debian.org/debian bullseye-backports main' > \
   /etc/apt/sources.list.d/backports.list
  apt update

  apt install -t bullseye-backports cockpit
  ```
  
Connect to the Cockpit web console at port 9090:
  ```
  https://<host-ip-address>:9090
  ```

### Plugins installation
  - Add source for apt - mandatory for any plugin
    ```
    cd ~
    curl -sSL https://repo.45drives.com/setup -o setup-repo.sh
    bash setup-repo.sh
    apt update
    ```
    
  - Samba/NFS plugin
    ```
    apt install cockpit-file-sharing
    ```  
    - Configure `samba` for use of `Samba/NFS plugin`
      - Edit samba configuration file
        ```
        nano /etc/samba/smb.conf
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
