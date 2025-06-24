## mt5-pi-setup
MetaTrader5 is designed to run only on Windows 64-bit machines. In order for it to run on a Raspberry Pi some steps were taken as a hack of this process.   

1. The first step is to install the required tools for setting up chroot. `Debootstrap` is a tool used to bootstrap a basic Debian or Ubuntu system into a directory, without needing a Debian installation medium. `qemu-user-static` allows running executables compiled for different CPU architectures on a host system, even if the host and target architectures are different. It does this by translating instruction set of a target architecture to the host architecture providing usermode emulation. `binfmt-support` provides tools for managing and extending the kernel's ability to handle different binary file formats, allowing the system to recognize and execute files based on their format
   ```
   sudo apt update
   sudo apt install debootstrap qemu-user-static binfmt-support
2. Next, we create a root for amd64, which is the only acceptable architecture for mt5. For this, I used the command below to install a minimalist bootstrap of Debian Bookworm operating system for amd64, setting /opt/amd64-bookworm as the directory.
   ```
   sudo mkdir -p /opt/amd64-bookworm
   sudo debootstrap --arch=amd64 --foreign bookworm /opt/amd64-bookworm https://ftp.debian.org/debian/
3. Copy QEMU static binary into the refroot, we only copy the x86_64 static as we only plan to emulate amd64 binary instruction set
   ```
   sudo cp /usr/bin/qemu-x86_64-static /opt/amd64-bookworm/usr/bin/
4. We can now create chroot jail and finish bootstrapping. We now have a minimal Debian Bookworm amd64 root filesystem ready.
   ```
   sudo chroot /opt/amd64-bookworm /debootstrap/debootstrap --second-stage
5. Next we enter our new chroot using the below command and update the package list and install any missing dependencies we would need, point to the right source list for this distribution. I take an install when needed approach here!
   ```
   sudo chroot /opt/amd64-bookworm

   echo "deb https://deb.debian.org/debian bookworm main contrib non-free non-free-firmware" > /etc/apt/sources.list
   echo "deb https://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware" >> /etc/apt/sources.list
   echo "deb https://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware" >> /etc/apt/sources.list

   apt update
   apt install <any_essential_package_missing>
6. It is possible the minimalist bootstrap does not contain apt or wget, which makes it dificult to make any other installation. A walk around for this is to download wget and apt outside the chroot jail and then copy the dpkg files to the chroot jail which we can install with dpkg. The problem with this approach is there tends to be more packages or libraries that the installed dpkg depend on, so we end up having to download those outside of our chroot jail and copy then into our chroot to get this working (we just found out why package managers are so great :)). Once we sort the dependencies for apt and wget installation of other packages becomes easier.
   ```
   # registry to find debian packages directly to download
   https://deb.debian.org/debian/pool/main/
   # for example if we want to install version 2.6.1 of apt we can find a link to it in the provided debian package registry
   wget http://deb.debian.org/debian/pool/main/a/apt/apt_2.6.1_amd64.deb
   dpkg -i apt_2.6.1_amd64.deb
7. We now have all the initial setup done, now we can install wine so we can be able to run Windows applications in amd64. For this, we follow the step outlines on [wine-hq](https://gitlab.winehq.org/wine/wine/-/wikis/Debian-Ubuntu). Once this is done we should confirm we have both wine 32 bit and wine 64 bit so we can run both 32-bit and 64 bit applications. If wine64 does not get installed we would have to install it as a seperate step with `apt install wine64`.
