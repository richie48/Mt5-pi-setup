## mt5-pi-setup
MetaTrader5 is designed to run only on Windows 64-bit machines. In order for it to run on a Raspberry Pi, some steps were taken as a hack of this process.   

1. The first step is to install the required tools for setting up chroot. `Debootstrap` is a tool used to bootstrap a basic Debian or Ubuntu system into a directory, without needing a Debian installation medium. `qemu-user-static` allows running executables compiled for different CPU architectures on a host system, even if the host and target architectures are different. It does this by translating the instruction set of a target architecture to the host architecture, providing user-mode emulation. `binfmt-support` provides tools for managing and extending the kernel's ability to handle different binary file formats, allowing the system to recognise and execute files based on their format
   ```
   sudo apt update
   sudo apt install debootstrap qemu-user-static binfmt-support
2. Next, we create a root for amd64, which is a "workable" architecture for MT5 on Linux. For this, I used the command below to install a minimalist bootstrap of Debian Bookworm operating system for amd64, setting /opt/amd64-bookworm as the directory.
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
   # For example, if we want to install version 2.6.1 of apt we can find a link to it in the provided Debian package registry
   wget http://deb.debian.org/debian/pool/main/a/apt/apt_2.6.1_amd64.deb
   dpkg -i apt_2.6.1_amd64.deb
7. We now have all the initial setup done, now we can install wine so we can be able to run Windows applications in AMD64. For this, we follow the step outlines on [wine-hq](https://gitlab.winehq.org/wine/wine/-/wikis/Debian-Ubuntu). Once this is done, we should confirm we have both Wine 32-bit and Wine 64-bit so we can run both 32-bit and 64-bit applications. If wine64 does not get installed, we would have to install it as a separate step with `apt install wine64`. Ideally, we should install Wine version 9.0.0, this is a more recent stable version of ,Wine and it patches a missing [library](https://github.com/ptitSeb/box64/issues/1555). If the version installed from winhq.org is older than this version we should look to upgrade to version 9.0.0. The first step is to remove all wine package so we do not have conflicting dependencies
   ```
   apt remove --purge 'wine*' 'libwine*'
   apt autoremove --purge
   apt clean
   ```
   now we are ready to install our new wine version. Once done verify installation.
   ```
   apt update
   apt install --install-recommends winehq-stable=9.0.0.0~bookworm-1 wine-stable=9.0.0.0~bookworm-1 wine-stable-amd64=9.0.0.0~bookworm-1 wine-stable-i386=9.0.0.0~bookworm-1
   wine --version
   wine64 --version
8. Now that we have Wine installed, we can run Windows 64-bit applications on our Pi. To install MT5, we would need to have some form of display, as we would have to click some buttons when trying to install. It was not designed to run headless. To do this, we would be trying to remotely access graphic feeds over SSH from a machine with GUI support. We make use of VNC for this.
   ```
   sudo apt install tightvncserver
   # Start the VNC server and set a password when requested
   vncserver :1
   ```
   we have to install [VNC-viewer](https://www.realvnc.com/en/connect/download/viewer/) on the machine we are SSH-ing from and connect it to our Pi. Now to be able to access the pi via a GUI by entering the IP address or hostname of our Raspberry Pi and the display in VNC Viewer "server address search". We would enter the password we had set earlier when creating the VNC server. Now we can see the Pi's desktop in a window on my PC (We can chose to skip the entire VNC installation if we can display content from the PI on a monitor with an HDMI, this did not work for me, hence why I went the remote display route)
   ```
   <raspberry_pi_ip_or_hostname>:1
   ```
   We should see a plain grey screen on our VNC viewer, this is the default screen when using VNC viewer before configuration.
9. The next step is to edit the VNC startup file to launch a desktop environment. We would need to download a lightweight desktop like LXDE. 
    ```
    sudo apt update
    sudo apt install lxde-core
    ```
    Then we replace the content in the VNC startup file to the content provided below.
   ```
   # Open the start file in an IDE to edit content and after add executable permission
   vi ~/.vnc/xstartup
   chmod +x ~/.vnc/xstartup
   
   # Update the content of ~/.vnc/xstartup to start the desktop environment
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
11. At this point although we can see a desktop display our chroot has no access to the X server. Below command allows any user include users in chroot to be able to connect to the X server
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
12. Now we can successfully run the setup of MT5. We first try to get the [mt5 installation for linux](https://www.metatrader5.com/en/download).
    ```
    wget https://download.mql5.com/cdn/web/metaquotes.software.corp/mt5/mt5ubuntu.sh
    ```
    We do not run the shell script as it may not always be compatible when working with a chroot jail. Refroot tends to struggle with multi-architecture, so we want to surgically run the steps we think are useful and modify others to suit our use case. The shell script should contain a step to download `mt5setup.exe` from a URL provided, we should explicitly run this step. Now that we have `mt5setup.exe` downloaded, we can begin the installation. But before we can install, we must first set a prefix for wine (64-bit prefix since we are trying to run 64-bit applications)
    ```
    WINEDLLPATH=/opt/wine-stable/lib64/wine/x86_64-unix WINEPREFIX=~/.mt5 wine64 winecfg
    WINEDLLPATH=/opt/wine-stable/lib64/wine/x86_64-unix WINEPREFIX=~/.mt5 wine64 mt5setup.exe
    ```
    we would need to do some button clicking on the GUI now displayed in VNC viewer to complete the installation. Once done, we should find our MT5 application installed at `/opt/amd64-bookworm/.mt5/drive_c/Program\ Files/`
13. We no longer need to interact with our GUI and therefore can resolve to using a virtual display when running MT5 forward. We can install a lightweight display server in our chroot jail which we would be runnning before launching MT5.
    ```
    sudo apt install xvfb
    Xvfb :1 -screen 0 1024x768x24 &
    export DISPLAY=:1
14. Now we have MT5 installed and running, we want to be able to interact with MT5 from within a python script. For this we would need [python_metatrader5](https://www.mql5.com/en/docs/python_metatrader5), which serves as a remote control for mt5. This does not run outside of Windows. To work around this, we would be installing Python for Windows within `Wine Shell`, then we would be able to install python_metatrader5 in the shell. I noticed that all the python3.13 installers were 32-bit, which made it difficult to install my 64-bit Python executable. The alternative was to install [Windows embeddable package (64-bit)](https://www.python.org/ftp/python/3.13.5/python-3.13.5-embed-amd64.zip).
    ```
    mkdir -p ~/.mt5/drive_c/python313-embed
    unzip python-3.13.5-embed-amd64.zip -d ~/.mt5/drive_c/python313-embed
    ```
    we can confirm python is now working in our wine shell
    ```
    WINEDLLPATH=/opt/wine-stable/lib64/wine/x86_64-unix WINEPREFIX=~/.mt5 wine64 cmd
    C:\python313-embed\python.exe --version
    # We should see "Python 3.13.0"
15. Because this is a lightweight version of Python we do not get pip as part of the embedded Python package. We would have to install pip in another step. We need this so we can install all the packages needed to remotely control mt5. 
   
## Alternative solution
* A more stable alternative solution for setting up MT5 for Raspberry Pi is to create a Windows virtual machine on the pi, and then install mt5 on this virtual machine. Then we have to trigger actions on mt5 in the present setup from a task running on the Pi to the task running on the Windows virtual machine by creating an application programming interface(API). Our Windows virtual machine is treated as a separate kernel, and therefore, we do not have any direct way to trigger a task from the physical machine kernel to the virtual machine kernel. To do this, we would need an operating system emulator like qemu, a VM manager like virt-manager etc. We would also need to download a Windows ISO, which we would install into a setup and booted Windows VM created by qemu in this case. Although more stable, this approach required more effort than the present setup, since mt5 usage is light, we don't expect to run into bottlenecks with the picked setup.
