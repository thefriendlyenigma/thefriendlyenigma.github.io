+++
title = "Nvidia GPU Passthrough to VM on Gaming Laptop"
date = 2025-11-30
draft = false
showTableOfContents = true
+++

## Summary {#summary}

This is how to setup GPU passthrough on a laptop - allocating the dedicated Nvidia GPU for use by Virtual Machines, while leaving the Integrated GPU with the host for basic display capabilities.

My specific use-case for this was to repurpose a laptop as both a secondary display for my desktop PC (through Moonlight), and a dedicated Type 1 Hypervisor with GPU acceleration.

I found that although the process is simpler than a Single GPU passthrough, there are a few 'pitfalls' that if fallen into, can turn the process into a timesink. Therefore, I am publishing this guide on how to most efficiently complete the process.


## Prerequisites {#prerequisites}

-   The host laptop must:
    -   Be capable of virtualisation (compatible CPU, enabled in BIOS settings, enough RAM for the guest OS, etc.).
    -   Have Linux Mint 22.x installed, with Nvidia proprietary drivers installed through Mint's Driver Manager.
        -   In theory, other Linux distributions could work, I only use Linux Mint as an example because Nvidia's proprietary drivers also install the Nvidia PRIME Applet.
        -   In my case, I had Linux Mint 22.3 installed.
    -   Have both a dedicated Nvidia GPU and an iGPU.
        -   In my case, I used a Lenovo LOQ 15IRX11, which has a Nvidia RTX 5050 and Intel UHD Graphics for 13th Gen Intel Processors.
-   A hex editor such as [Okteta](https://flathub.org/en/apps/org.kde.okteta) must be installed on the host machine.
-   [Moonlight](https://flathub.org/en/apps/com.moonlight_stream.Moonlight) must be installed on the host machine.
-   An official Windows 11 ISO, which [can be downloaded from Microsoft directly.](https://www.microsoft.com/en-us/software-download/windows11)
-   The Virtio Drivers ISO, which [can be downloaded from Red Hat.](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/?C=M;O=D)


## Step 1: Preparing the GPU {#step-1-preparing-the-gpu}

1.  First, confirm that through the Nvidia PRIME applet, your laptop GPU settings are at "NVIDIA On-Demand".
    -   This should be the default setting, but if it isn't, then change it to "NVIDIA On-Demand", then log out and back in.
2.  Follow [Step 6 of risingprism's guide](https://gitlab.com/risingprismtv/single-gpu-passthrough/-/wikis/6%29-Preparation-and-placing-of-the-ROM-file), the abbreviated steps as follows:
    -   Run the **x86 version** of `nvflash` to extract the vbios from your Nvidia GPU as the file `vbios.rom`.
        ```shell
        sudo chmod +x nvflash
        sudo ./nvflash --save vbios.rom
        ```

        -   Even though the guide recommends the x64 version, I found that the x64 version didn't work whereas the x86 version did.
    -   Edit the vbios using Okteta - remove everything from BEFORE the line containing VIDEO, then save the resultant file as `patched.rom`.
    -   With `sudo` permissions, create the folder `/usr/share/vgabios`
        ```shell
        sudo mkdir -p /usr/share/vgabios
        ```
    -   Open the Terminal in the location `patched.rom` is, move the file to inside folder `/usr/share/vgabios`.
        ```shell
        sudo mv patched.rom /usr/share/vgabios/
        ```
    -   Change permissions to 644, and set your username as the owner and group
        ```shell
        sudo chmod 644 /usr/share/vgabios/patched.rom
        sudo chown -R $USER:$USER /usr/share/vgabios/patched.rom
        ```

        -   **This presumes that the name of your primary group is the same as your username.**
3.  Change Nvidia PRIME settings to Power Saving Mode (Intel only), then log out and in. This will limit your host to only using the iGPU, so the Nvidia GPU is now free to be attached to Virtual Machines.


## Step 2: Installing and Configuring Virtualisation Software {#step-2-installing-and-configuring-virtualisation-software}

1.  Install `virt-manager` (which will in turn install its associated programs as dependencies) and `bridgeutils`.
    ```shell
    sudo apt install virt-manager bridgeutils
    ```
2.  Add your user to the `libvirt` and `kvm` groups, which grants you permission to use the .
    ```shell
    sudo usermod -a -G kvm,libvirt $(whoami)
    ```
3.  Start the `libvirt` daemon, and also set it to start on every system boot.
    ```shell
    sudo systemctl enable --now libvirtd
    ```
4.  Enable the virtual machine default network, .
    ```shell
    sudo virsh net-autostart default
    ```


## Step 3: Configuring the Windows 11 Virtual Machine {#step-3-configuring-the-windows-11-virtual-machine}

This uses [Risingprism's configurations in Step 5 of their guide](https://gitlab.com/risingprismtv/single-gpu-passthrough/-/wikis/5%29-Configuring-Virtual-Machine-Manager) as a reference.

1.  When prompted to make storage, it is optimal for performance to create a RAW file and have the disk pre-filled.
2.  **Overview Tab:** Set the Firmware as `/usr/share/OVMF/OVMF_CODE_4M.secboot.fd`
3.  **CPUs Tab:** Check the "Copy host CPU configuration (host-passthrough)", and match the number of Cores and Threads to the host.
4.  **Disk Tab:** Set the Bus type as VirtIO. In 'Advanced Settings', set Cache mode to writeback.
5.  **NIC Tab:** Set the device model as virtio.
    -   Note this means the VM will not have network access until Virtio Drivers are installed.
6.  **Add Hardware:** The Virtio Driver ISO should be added as a CDROM Device with a SATA bus type.


## Step 4: Installing the Windows 11 Virtual Machine {#step-4-installing-the-windows-11-virtual-machine}

1.  Follow the Windows installation as you normally would. Once you reach a page that asks you to select the drive to install to, you need to click on "Load Driver", select the Virtio CD. Expand the 'amd64' folder dropdown, then select the 'w11' folder that appears inside it.
2.  Once Windows 11 has installed to the virtual drive and rebooted to the welcome / account setup screen, press Shift+F10, then enter into the command prompt `ms-cxh:localonly`. Set your username and password, then toggle privacy settings as you wish.
3.  Once the installation completes and the desktop appears, go to Explorer and access the Virtio CD again. Double-click on `virtio-win-guest-tools.exe` to install the full Virtio driver package. This will grant network access, and also ensure proper integration between the VM and the host system.


## Step 5: Installing Sunshine and Confirming Remote Access {#step-5-installing-sunshine-and-confirming-remote-access}

These steps are **VERY** important - Sunshine and Moonlight are how you will be able to utilise GPU acceleration on the VM. Additionally, once the step after this is completed, the Windows 11 cursor will no longer properly sync to the display in virt-manager's built-in GUI, so configuring from within the VM will become very difficult.

1.  Install Sunshine in the VM, and configure your username and password.
2.  On your host, go to your VM's settings in virt-manager, select the network card, and find the IP address of your VM. By default, it should begin with 192.168.122.
3.  Enter the IP address into your web browser on your host, adding an `https://` at the front and `:47990` at the end. This should load the Sunshine Web UI of your VM.
4.  Keeping the Web UI open in the background, start Moonlight on your host PC, create a new connection, then enter the VM's IP address. You should be prompted for to enter a PIN in on the Sunshine device. Enter the PIN from Moonlight in the Sunshine Web UI to register your host's Moonlight installation with the VM's Sunshine installation.
5.  Try to access the VM's desktop through Moonlight. It should properly load. If not, then confirm your network settings aren't blocking anything, on both your VM and host.


## Step 6: Installing a Virtual Display {#step-6-installing-a-virtual-display}

1.  In the VM, install the [Virtual Display Driver from MikeTheTech](https://github.com/VirtualDrivers/Virtual-Display-Driver).
2.  Enter Sunshine through the Web UI on your host.
3.  Go to the Troubleshooting section, then click the 'Restart Sunshine' button. When Sunshine restarts, it should be able to detect the Virtual Display Driver.
4.  Search the Logs to find the unique ID of the Virtual Display Driver. The `manufacturer_id` value should be "MTT" and the `product_code` value should be "1337". The unique ID should be located a couple of lines above the manufacturer_id value, under the heading `device_id`.
5.  Enter the Configuration section, then go to the Audio/Video tab.
    -   Enter the unique ID of the Virtual Display Driver into the `config.output_name_windows` section
    -   In the Advanced display device options, click the "Device configuration" dropdown menu, and change the selected item from `Disabled (default)` to `Deactivate other displays and activate only the specified display`.
6.  Click 'Save', then 'Apply'.
7.  Access your VM through Moonlight on the host again, which should properly load the desktop, then shut down the VM.


## Step 7: Adding the GPU {#step-7-adding-the-gpu}

1.  In virt-manager, go to your VM's configuration settings, select "Add Hardware", select "PCI Host Device", then add both Nvidia PCI devices - the GPU and the GPU Audio - to your Windows 11 VM.
    -   They may not explicitly specify that they are GPUs, they may only say `0000:01:00:0 NVIDIA Corporation`  and `0000:01:00:1 NVIDIA Corporation`. In that case, the device that ends with `:0` is the GPU, and the device that ends with `:1` is the GPU audio.
    -   DO **NOT** DELETE THE SPICE DISPLAY - it is the only way to enter system shortcuts such as `Ctrl+Alt+Del` or `Win+L` into the VM, Moonlight doesn't send those inputs through.
2.  Access the GPU XML, and add the path to your ROM file beneath the `</source>` line.
    -   By default, the XML would look like:
        ```xml
        <hostdev mode="subsystem" type="pci" managed="yes">
          <source>
            <address domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
          </source>
          <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x0"/>
        </hostdev>
        ```
    -   After adding the rom file, it would look similar to:
        ```xml
        <hostdev mode="subsystem" type="pci" managed="yes">
          <source>
            <address domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
          </source>
          <rom file="/usr/share/vgabios/patched.rom"/>
          <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x0"/>
        </hostdev>
        ```
3.  Boot up the Windows 11 VM, connect to it through Moonlight, and run a Windows Update. This should automatically download the Nvidia drivers. This process can take a while, in my experience from 30 to 60 minutes.
    -   If Windows Update does not download the Nvidia driver, or you would prefer to use the latest version of Nvidia drivers, [then download and run Nvcleanstall from TechPowerUp instead.](https://www.techpowerup.com/download/techpowerup-nvcleanstall/)
4.  Finally, once Nvidia drivers are installed, restart Sunshine through the Web UI, then connect to the VM through Moonlight again. Checking the Task Manager, you should be able to see the presence of the Nvidia GPU, as well as a Nvidia icon in the taskbar confirming you now have a functioning GPU.
5.  Congratulations, you now have a fully functional VM with GPU acceleration, _without_ taking control away from the host OS!


## Additional Configuration {#additional-configuration}

-   If your laptop has an ethernet adapter, and you would like for the VM to be accessible by **remote** hosts on the physical network, use virt-manager to add the interface as an additional Network card with the `macvtap` type.
-   If your laptop can only connect to the network through WiFi, then currently, `macvtap` cannot be utilised. Alternative options for the remote host to access the VM are installing ZeroTier or Tailscale on both of them. It is also possible to configure a WireGuard server such as [PiVPN](https://www.pivpn.io/) on the local network, then connecting both the remote host and the VM to it.
-   For lower latency and an even more accurate picture when connecting the **host** device to the VM, it is possible to install and configure [Looking Glass](https://looking-glass.io/). Setup for that program is beyond the scope of this guide, so please refer to the website for more information.
