## mt5-pi-setup
MetaTrader5 is designed to run only on Windows 64-bit machines. In order for it to run on a Raspberry Pi, some steps were taken as a hack of this process.   

1. The first step is to install the required tools for setting up chroot. `Debootstrap` is a tool used to bootstrap a basic Debian or Ubuntu system into a directory, without needing a Debian installation medium. `qemu-user-static` allows running executables compiled for different CPU architectures on a host system, even if the host and target architectures are different. It does this by translating the instruction set of a target architecture to the host architecture, providing user-mode emulation. `binfmt-support` provides tools for managing and extending the kernel's ability to handle different binary file formats, allowing the system to recognise and execute files based on their format
   ```
   sudo apt update
   sudo apt install debootstrap qemu-user-static binfmt-support
2. Next, we create a root for amd64, which is a "workable" architecture for mt5 on Linux. For this, I used the command below to install a minimalist bootstrap of Debian Bookworm operating system for amd64, setting /opt/amd64-bookworm as the directory.
   ```
   sudo mkdir -p /opt/amd64-bookworm
   sudo debootstrap --arch=amd64 --foreign bookworm /opt/amd64-bookworm https://ftp.debian.org/debian/
3. Copy QEMU static binary into the refroot, we only copy the x86_64 static as we only plan to emulate amd64 binary instruction set
   ```
   sudo cp /usr/bin/qemu-x86_64-static /opt/amd64-bookworm/usr/bin/
4. We can now create a chroot jail and finish bootstrapping. We now have a minimal Debian Bookworm amd64 root filesystem ready.
   ```
   sudo chroot /opt/amd64-bookworm /debootstrap/debootstrap --second-stage
5. Next, we enter our new chroot using the below command and update the package list and install any missing dependencies we would need, pointing to the right source list for this distribution. I take an install when needed approach here!
   ```
   sudo chroot /opt/amd64-bookworm

   echo "deb https://deb.debian.org/debian bookworm main contrib non-free non-free-firmware" > /etc/apt/sources.list
   echo "deb https://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware" >> /etc/apt/sources.list
   echo "deb https://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware" >> /etc/apt/sources.list

   apt update
   apt install <any_essential_package_missing>
6. It is possible that the minimalist bootstrap does not contain apt or wget, which makes it difficult to make any other installation. A walk-around for this is to download wget and apt outside the chroot jail and then copy the dpkg files to the chroot jail, which we can install with dpkg. The problem with this approach is that there tend to be more packages or libraries that the installed dpkg depends on, so we end up having to download those outside of our chroot jail and copy them into our chroot to get this working (we just found out why package managers are so great :)). Once we sort the dependencies for apt and wget installation of other packages becomes easier.
   ```
   # registry to find Debian packages directly to download
   https://deb.debian.org/debian/pool/main/
   # For example, if we want to install version 2.6.1 of apt we can find a link to it in the provided debian package registry
   wget http://deb.debian.org/debian/pool/main/a/apt/apt_2.6.1_amd64.deb
   dpkg -i apt_2.6.1_amd64.deb
7. We now have all the initial setup done, now we can install wine so we can be able to run Windows applications in amd64. For this, we follow the step outlines on [wine-hq](https://gitlab.winehq.org/wine/wine/-/wikis/Debian-Ubuntu). Once this is done, we should confirm we have both wine 32-bit and wine 64-bit so we can run both 32-bit and 64-bit applications. If wine64 does not get installed, we would have to install it as a separate step with `apt install wine64`.
8. Now that we have Wine installed, we can run Windows 64-bit applications on our Pi. To install mt5 we would need to have some form of display as we would have to click some buttons when trying to install. It was not designed to run headless. To do this, we would be trying to remotely access graphic feeds over SSH from a machine with GUI support. We make use of VNC for this.
   ```
   sudo apt install tightvncserver
   # Start the vnc server and set a password when requested
   vncserver :1
   ```
   we have to install [VNC-viewer](https://www.realvnc.com/en/connect/download/viewer/) on the machine we are SSH-ing from and connect it to our pi. Now to be able to access the pi via a GUI by entering the IP address or hostname of our Raspberry Pi and the display in VNC Viewer "server address search". We would enter the password we had set earlier when creating the VNC server. Now we can see the Pi's desktop in a window on my PC (We can chose to skip the entire VNC installation if we can display content from the PI on a monitor with an HDMI, this did not work for me, hence why I went the remote display route)
   ```
   <raspberry_pi_ip_or_hostname>:1
   ```
   we should see a plan grey screen on our VNC viewer, this is the default screen when using VNC viewer before configuration.
9. Next step is to edit the VNC startup file to launch a desktop environment. We would need to download a lightweight desktop like LXDE. 
    ```
    sudo apt update
    sudo apt install lxde-core
    ```
    Then we replace the content in the VNC startup file to the content provided below.
   ```
   # Open the start file in an IDE to edit content and after add executable permission
   vi ~/.vnc/xstartup
   chmod +x ~/.vnc/xstartup
   
   # Update the content of ~/.vnc/xstartup to start desktop environment
   #!/bin/sh
   xrdb $HOME/.Xresources
   startlxde &
   ```
   We can restart our vnc server and reconnect to it on vnc viewer now. We should now see a desktop displayed
   ```
   vncserver -kill :1
   vncserver :1
   # Confirm server is running and the display
   ps aux | grep vnc
10. At this point although we can see a desktop display our chroot has no access to the X server. Below command allows any user include users in chroot to be able to connect to the X server
    ```
    export DISPLAY=:1
    xhost +local:
    ```
    Now from within our chroot jail we `export DISPLAY=:1` to connect to our X server. We should also install x11-apps and test our GUI shows up in vnc viewer. For this, we would be running xclock, which should render a GUI in our vnc viewer if the setup is correct 
    ```
    # sometimes wine apps need access to /tmp/.X11-unix and .Xauthority.
    sudo mount --bind /tmp/.X11-unix /opt/amd64-bookworm/tmp/.X11-unix
    sudo cp /home/youruser/.Xauthority /opt/amd64-bookworm/root/
    export XAUTHORITY=/root/.Xauthority

    # test installation
    apt install x11-apps
    xclock
    ```
11. Now we can successfully run the setup of mt5


   
## Alternative solution
* A more stable alternative solution for setting up mt5 for Raspberry Pi is to create a Windows virtual machine on the pi, and then install mt5 on this virtual machine. Then we have to trigger actions on mt5 in the present setup from a task running on the Pi to the task running on the Windows virtual machine by creating an application programming interface(API). Our Windows virtual machine is treated as a separate kernel, and therefore, we do not have any direct way to trigger a task from the physical machine kernel to the virtual machine kernel. To do this, we would need an operating system emulator like qemu, a VM manager like virt-manager etc. We would also need to download a Windows ISO, which we would install into a setup and booted Windows VM created by qemu in this case. Although more stable this approach required more effort than the present setup, since mt5 usage is light, we don't expect to run into bottlenecks with the picked setup.
