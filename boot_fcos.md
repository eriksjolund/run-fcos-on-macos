## Boot up a Fedora CoreOS VM on macOS (MacBook Pro M1)

### Summary

1. Install requirements
2. Create SSH key pair
3. Create Butane configuration file
4. Convert from Butane to Ignition format
5. Download Fedora CoreOS qcow2 image
6. Boot the qcow2 image with the Ignition file passed as an argument

### Details

1. Install the package manager __brew__ from https://brew.sh

2. Install __qemu__, __xquartz__ and __butane__ by running the commands
   ```
   brew update
   brew install qemu xquartz butane
   ```
3. If the directory _~/.ssh_ does not yet exist, create the directory and limit access to it
   ```
   mkdir ~/.ssh
   chmod 700 ~/.ssh
   ```
4. Generate the SSH key pair. Run the command
   ```
   ssh-keygen -t ed25519 -f ~/.ssh/fcos
   ```
   to generate the two files
   * _~/.ssh/fcos_
   * _~/.ssh/fcos.pub_
5. Create the directory _~/fcos_ and cd into it
   ```
   mkdir ~/fcos
   cd ~/fcos
   ```
6. Generate a butane file
   ```
   cat << EoF > ./file.butane
   variant: fcos
   version: 1.4.0
   passwd:
     users:
       - name: core
         ssh_authorized_keys:
   EoF
   echo -n "        - " >> ./file.butane
   cat ~/.ssh/fcos.pub >> ./file.butane 
   ```
   The resulting file should look something like
   ```
   variant: fcos
   version: 1.4.0
   passwd:
     users:
       - name: core
         ssh_authorized_keys:
           - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICREcM6l+Rqjy4X5xOw9/KC2ZXnGi2GnD7Zf9wCZf/Zx myuser@MYCOMPUTER
   ```
   but with a different public SSH key.
7. Convert the file _file.butane_ to _file.ignition_.
   ```
   butane \
     --strict \
     --output file.ignition \
     file.butane
   ```
8. Go to https://getfedora.org/en/coreos/download?tab=cloud_launchable&stream=stable&arch=aarch64
9. Decide which release stream you would like to download (_Stable_, _Testing_ or _Next_) and click
   the __Show Downloads__ for that release stream.
10. In the __Architecture__ dropdown menu select: _aarch64_
11. Click the tab __Bare Metal & Virtualized__
12. Click __Download__ in the section __QEMU (qcow2.xz)__

13. Move the downloaded file to the current directory and decompress it
    ```
    mv ~/Downloads/fedora-coreos-*-qemu.aarch64.qcow2.xz .
    xz -d fedora-coreos-*-qemu.aarch64.qcow2.xz
    ```
14. Create a copy of the qcow2 file that will be used by qemu
    ```
    cp fedora-coreos-*-qemu.aarch64.qcow2 fcos.qcow2
    ```
    (The file _fcos.qcow2_ will be modified when the VM is booted and configured)
15. Append the following text to the file _~/.ssh/config_
    ```
    Host fcos
      HostName 127.0.0.1
      User core
      Port 10022
      ServerAliveInterval 300
      IdentityFile ~/.ssh/fcos
    ```
    (Create the file _~/.ssh/config_ if it is missing)
16. Create the file _pflash.img_ by running the command
    ```
    /bin/dd if=/dev/zero conv=sync bs=1m count=64 of=pflash.img
    ```
17. Create the file _start.sh_ with the following file contents
    ```
    #!/bin/bash
    qemu-system-aarch64 \
      -accel hvf \
      -accel tcg \
      -cpu host \
      -device virtio-net,netdev=hostnet0,mac=16:24:c4:07:e4:87 \
      -device virtio-serial \
      -drive file=/opt/homebrew/Cellar/qemu/7.1.0/share/qemu/edk2-aarch64-code.fd,if=pflash,format=raw,readonly=on \
      -drive file=pflash.img,if=pflash,format=raw \
      -drive if=virtio,file=fcos.qcow2 \
      -fw_cfg name=opt/com.coreos/config,file=file.ignition \
      -m 2048 \
      -machine virt,highmem=on \
      -monitor stdio \
      -netdev user,id=hostnet0,hostfwd=tcp:127.0.0.1:10022-0.0.0.0:22 \
      -smp 1
    ```
18. Start the VM
    ```
    bash start.sh
    ```
19. Wait for the VM to boot (about 17 seconds) and then log in with SSH
    ```
    ssh fcos
    ```
