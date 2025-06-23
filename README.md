## mt5-pi-setup
MetaTrader5 is designed to run only on Windows 64-bit machines. In order for it to run on a Raspberry Pi some steps were taken as a hack of this process.   

1. The first step is to install the required tools for setting up chroot. `Debootstrap` is a tool used to bootstrap a basic Debian or Ubuntu system into a directory, without needing a Debian installation medium. `qemu-user-static` allows running executables compiled for different CPU architectures on a host system, even if the host and target architectures are different. It does this by translating set of a target architecture to the host architecture providing usermode emulation. `binfmt-support` provides tools for managing and extending the kernel's ability to handle different binary file formats, allowing the system to recognize and execute files based on their format
   ```
   sudo apt update
   sudo apt install debootstrap qemu-user-static binfmt-support
2. Next, we create a root for amd64, which is the only acceptable architecture for mt5. For this, I used the command below to install a minimalist bootstrap of Debian Bookworm operating system for amd64, setting /opt/amd64-bookworm as the directory.
   ```
   sudo mkdir -p /opt/amd64-bookworm
   sudo debootstrap --arch=amd64 --foreign bookworm /opt/amd64-bookworm https://ftp.debian.org/debian/
3. Copy QEMU static binary into the refroot, we only copy the x86_64 binary as we only plan to emulate amd64 binary instruction set
   ```
   sudo cp /usr/bin/qemu-x86_64-static /opt/amd64-bookworm/usr/bin/
4. We can now create chroot jail and finish bootstrapping. We now have a minimal Debian Bookworm amd64 root filesystem ready.
   ```
   sudo chroot /opt/amd64-focal /debootstrap/debootstrap --second-stage
5. Next we enter our new chroot using the below command and update the package list and install any missing dependencies we would need. I take an install when needed approach here!
   ```
   sudo chroot /opt/amd64-bookworm
   apt update
   apt install <any_essential_package_missing>
