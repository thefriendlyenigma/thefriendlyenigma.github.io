+++
title = "Connecting Kobo to Wireguard VPN"
date = 2025-04-23
draft = false
showTableOfContents = true
+++

## Summary {#summary}

This is a guide to set up userspace Wireguard on the Kobo Clara Colour eReader, allowing you to connect the Kobo to a VPN that uses Wireguard configuration files.
The use-case for this, is using a VPN that to connects to a **personal** network, allowing the user to access eBook servers hosted on their personal devices.


## Skills Demonstrated {#skills-demonstrated}

-   Working with embedded Linux terminals
-   Cross-compiling
-   Networking/VPN configuration
-   SSH hardening


## IMPORTANT INFORMATION {#important-information}


### Persistence Across Firmware Updates {#persistence-across-firmware-updates}

With my current experience involving Kobo firmware updates: after an update, the installed files and configurations remain, but in order to _access_ the terminal that can start the VPN again, you need connect the Kobo to a computer through USB and run the script to install KFMon. This can be retrieved from [the same page as the KOReader install package below](https://www.mobileread.com/forums/showthread.php?t=314220).


### VPN Has To Be Activated On Each Reboot {#vpn-has-to-be-activated-on-each-reboot}

It is _technically_ possible to automatically run a script that starts the VPN on each device reboot.

However, there is the risk of losing network access while configuring the scripts, or if the Wireguard server were to malfunction while tunnelling all traffic through it.

The only way to restore network access in that situation would be through the on-device terminal, which is awkward to use for extended periods, and as stated above, is temporarily lost on firmware updates.

I feel that it would be best to keep the device in a state where if a networking mishap occurred, you could restore normal functionality with a simple device reboot.


## Installing KOReader {#installing-koreader}

Install KOReader through [NiLuJe's One-Click Install package](https://www.mobileread.com/forums/showthread.php?t=314220), following the instructions for your respective Operating System. In addition to providing an excellent reading environment, KOReader functions as a terminal through which you can access your device's root filesystem and run Linux commands. The terminal functionality is a key part of enabling the VPN connection.

{{< details "**How to Access Terminal in KOReader**" >}}
1.  Tap the top bar to open the KOReader menu.

    {{< figure src="/images/kobo-wireguard/1.png" alt="Tap the top bar to open the KOReader menu." >}}
2.  Select the Settings icon.

    {{< figure src="/images/kobo-wireguard/2.png" alt="Select the Settings icon." >}}
3.  Go to Page 2 of the Settings section.

    {{< figure src="/images/kobo-wireguard/3.png" alt="Go to Page 2 of the Settings section." >}}
4.  Select the More Tools button.

    {{< figure src="/images/kobo-wireguard/4.png" alt="Select the More Tools button." >}}
5.  Select the Terminal Emulator button.

    {{< figure src="/images/kobo-wireguard/5.png" alt="Select the Terminal Emulator button." >}}
6.  Select the Open terminal session (not running) option.

    {{< figure src="/images/kobo-wireguard/6.png" alt="Select the Open terminal session (not running) option." >}}
{{< /details >}}


## Setting up SSH access {#setting-up-ssh-access}

KOReader's SSH package is unfortunately outdated and insecure, which causes it to have connectivity issues with modern computers.
Luckily, [NiLuJe's _Yet another telnet/sshd &amp; misc tools package_](https://www.mobileread.com/forums/showthread.php?t=254214) provided on the MobileRead forums has a properly functioning SSH service through the program Dropbear.

Follow the instructions as described in the forum to install the package.

{{< alert >}}
**PLEASE DO NOT DO THIS STEP ON PUBLIC WIFI.**
By default, SSH access allows anyone to connect without the need for a password or key.
{{< /alert >}}

Securing SSH access to use public key auth only:

1.  If you don't have an SSH key registered on your main device yet, generate one.
    This is done with the command:
    ```shell
    ssh-keygen
    ```
2.  Next, open the public key file, which as of OpenSSH 9.x's defaults, should be `id_ed25519.pub` in a text editor, and copy its **contents** to the clipboard.
3.  SSH to your Kobo device. The IP address of the device can be found in the KOReader menu through the Settings/Gear Icon -&gt; Network -&gt; Network Info. The user is `root`, meaning that if the Kobo's IP address was `192.168.1.128`, the command to connect would be:
    ```shell
    ssh root@192.168.1.128
    ```
    Make sure that the Kobo does NOT fall asleep while connected.
4.  Once connected to the Kobo through SSH, execute the command to create and edit the file containing the Kobo's authorised public keys.
    ```shell
    vi /usr/local/niluje/usbnet/etc/authorized_keys
    ```
5.  Press `i` to enter `Insert Mode`, then press `Ctrl+Shift+v` to paste the public key into the file. Press `:wq` to save the file and quit.
6.  Execute the command to edit the options for the Kobo's SSH service:
    ```shell
    vi /usr/local/stuff/bin/stuff-daemons.sh
    ```
    Uncomment Line 22.
    The line should be, `#SSHD_OPTS="${SSHD_OPTS} -s"`, and the line above should have the comment, `NOTE: Uncomment if you want to limit dropbear to shared-key auth only`.
    You need to remove the `#`.

    Line 22 Before:
    ```shell
    ## NOTE: Uncomment if you want to limit dropbear to shared-key auth only
    #SSHD_OPTS="${SSHD_OPTS} -s"
    ```
    Line 22 After:
    ```shell
    ## NOTE: Uncomment if you want to limit dropbear to shared-key auth only
    SSHD_OPTS="${SSHD_OPTS} -s"
    ```


## Compiling/Retrieving Wireguard Binaries {#compiling-retrieving-wireguard-binaries}


### Userspace Wireguard {#userspace-wireguard}

I have decided to use the userspace program `wireguard-go`, alongside `wireguard-tools` to manage the interfaces, rather than Wireguard's kernel modules.

The Kobo _does_ support Wireguard's kernel modules, as demonstrated by the terminal when running `wireguard-go`:

```text
┌──────────────────────────────────────────────────────┐
│                                                      │
│   Running wireguard-go is not required because this  │
│   kernel has first class support for WireGuard. For  │
│   information on installing the kernel module,       │
│   please visit:                                      │
│         https://www.wireguard.com/install/           │
│                                                      │
└──────────────────────────────────────────────────────┘
```

However, as the [documentation on the Kobo kernel](https://github.com/kobolabs/Kobo-Reader/blob/master/documentation/README.kernel) states, if the kernel were to malfunction, then it would require physically opening up the device and physically accessing the storage to fix.

Secondly, with newer devices such as the Kobo Clara Colour, the model has eMMC storage rather than an internal SD card, meaning that if the device were rendered unbootable, specialised equipment would be required to access the storage.

Lastly, `wireguard-go` is the same version of Wireguard that the proprietary service Tailscale uses. [A version of Tailscale for the Kobo](https://github.com/videah/kobo-tailscale) exists and is proven to work.


### Obtaining the `wireguard-go` binary {#obtaining-the-wireguard-go-binary}

`wireguard-go` is already available in the form of various pre-compiled binaries from [this repository](https://github.com/P3TERX/wireguard-go-builder).
The Kobo has a 32-bit ARM processor, so the version to download, as of writing, is [wireguard-go-linux-arm.tar.gz](https://github.com/P3TERX/wireguard-go-builder/releases/download/0.0.20230223/wireguard-go-linux-arm.tar.gz).


### Cross-compiling `wireguard-tools` {#cross-compiling-wireguard-tools}

The Kobo Clara uses a 32-bit ARM processor, meaning that the `wg` binary from [wireguard-tools](https://github.com/WireGuard/wireguard-tools) needs to be specifically cross-compiled for it.
I have already cross-compiled the binary for my Kobo, which is downloadable [here](https://github.com/thefriendlyenigma/wireguard-tools/releases/tag/v1.0.20250423).

If you would like to compile it yourself, then you will need to be running Linux. Download [koxtoolchain](https://github.com/koreader/koxtoolchain) scripts: both the scripts from the repository itself, and the prebuilt `kobov4` toolchain.

1.  Download the [kobov4 toolchain](https://github.com/koreader/koxtoolchain/releases/download/2025.01/kobov4.tar.gz) to your home directory
    -   As of writing, the Kobo Clara runs firmware of version 4.x.
2.  Download [the koxtoolchain repository itself as an archive](https://github.com/koreader/koxtoolchain/archive/refs/tags/2025.01.tar.gz) to your home directory
3.  Download [the wireguard-tools repository](https://github.com/WireGuard/wireguard-tools) to your home directory
4.  Extract all three downloaded archives. Your home directory should have three new folders:
    -   koxtoolchain-2025.01
    -   x-tools
    -   wireguard-tools
5.  Add `x-tools/arm-kobov4-linux-gnueabihf/bin` to your PATH. This can be done by adding this line to the bottom of the `.bashrc` file in the home directory:
    ```shell
    PATH=$PATH:/full/path/to/x-tools/arm-kobov4-linux-gnueabihf/bin/
    ```
    If the Terminal is open, close it and reopen it for the change to take effect.
6.  Install the `build-essential` package, which provides dependencies for compiling:
    ```shell
    sudo apt install build-essential
    ```
7.  Open a Terminal in the home directory, then run the command to activate the compiling environment:
    ```shell
    source koxtoolchain-2025.01/refs/x-compile.sh kobov4 env bare
    ```
8.  Run this command to cross-compile the binary:
    ```shell
    make CC=$HOME/x-tools/arm-kobov4-linux-gnueabihf/bin/arm-kobov4-linux-gnueabihf-gcc-14.2.0 -C wireguard-tools/src
    ```
9.  The resultant binary will be located in the `wireguard-tools/src` folder, with the name `wg`. This is the file that needs to be copied to your Kobo.


### Transferring Binaries to the Kobo {#transferring-binaries-to-the-kobo}

So as not to interfere with the Kobo's default system files (which are likely to be overridden upon a system update), the `wg` binary and the `wireguard-go` binary are to be placed in the directory `/usr/local/bin`.


## Editing Wireguard Conf File {#editing-wireguard-conf-file}

For this step, an example Wireguard configuration file from [PiVPN](https://www.pivpn.io/) was used, and I only used IPv4 addresses.

The tool `wg-quick`, which is used in most "full" systems to create Wireguard connections, does not work here because the programs it depends on such as `setopt`, `systemd`, and `resolvconf` do not exist on the Kobo OS.

Therefore, you need to manually change the Wireguard configuration file to allow the `wg` binary to process it:

1.  **Record** the two lines in the `[Interface]` section beginning with `Address =` and `DNS =` in  another document, then remove them from the configuration file.
2.  Add this line at the **bottom** of the document: `PersistentKeepAlive = 60` . Without this line, the IP address of the Kobo will needed to be pinged from the outside in order for the network connection to start.
3.  Depending on your personal preferences, the line `AllowedIPs = 0.0.0.0/0, ::0/0` to the range of IP addresses you would like to use the VPN for, as well as the IP address range of the VPN's internal network.

    For example, if you only want to use the VPN for IP addresses in the `192.168.1.x` range, and the IP address range of the VPN's internal network is `10.10.0.x` you would replace the line with: `AllowedIPs = 192.168.1.0/24, 10.10.0.0/24`

Generated Wireguard conf file example:

```cfg
[Interface]
PrivateKey = PRIVATE_KEY
Address = KOBO_IPV4_ADDRESS, KOBO_IPV6_ADDRESS
DNS = WIREGUARD_DNS_ADDRESS

[Peer]
PublicKey = PUBLIC_KEY
PresharedKey = PRESHARED_KEY
Endpoint = ENDPOINT_IPV4_ADDRESS
AllowedIPs = 0.0.0.0/0, ::0/0
```

The same file, edited for the Kobo:

```cfg
[Interface]
PrivateKey = PRIVATE_KEY

[Peer]
PublicKey = PUBLIC_KEY
PresharedKey = PRESHARED_KEY
Endpoint = ENDPOINT_IPV4_ADDRESS
AllowedIPs = 192.168.1.0/24, 10.10.0.0/24
PersistentKeepAlive = 60
```

Remember:

-   This presumes you would like to **split-tunnel** network access, using the examples of `192.168.1.0/24` for the devices on the destination network and `10.10.0.0/24` for the Wireguard network itself.
-   The `KOBO_IPV4_ADDRESS` and `WIREGUARD_DNS_ADDRESS` values must be recorded in a separate document

Transfer the final conf file to an easy-to-access place on the Kobo device. In my case, I have saved the script inside `/mnt/onboard`, which is the same place where user files such as books are saved.

This location was chosen because it won't be overridden upon firmware updates, and it is easily accessible over USB. I recommend the filename being a simple name, without spaces, as it will be required for the launching script.


## Final Steps on Kobo {#final-steps-on-kobo}


### Configuring KOReader to properly load the PATH {#configuring-koreader-to-properly-load-the-path}

First of all, KOReader's Terminal needs to be configured to acknowledge that the directory `/usr/local/bin` contains executable files.
This can be done by SSHing into the Kobo, creating the file `/mnt/onboard/.adds/koreader/scripts/profile.user`, and then entering in contents as:

```shell
PATH=$PATH:/usr/local/bin
```

Now whenever the **KOReader terminal** (NOT the SSH terminal) is started, it can run programs and scripts stored inside `/usr/local/bin`.


### Configuring VPN DNS {#configuring-vpn-dns}

Because `wg-quick` cannot be used and DNS isn't set by PiVPN automatically, you need to add the DNS servers in `/etc/resolv.conf` in order to ensure proper internet connectivity.
**This is where you paste in the addresses of the DNS servers from the original Wireguard configuration file.**

For example, if you wanted to use the `1.1.1.1` DNS server and its fallback `1.0.0.1`, you would enter inside the file, two lines:

-   `nameserver 1.1.1.1`
-   `nameserver 1.0.0.1`

The `/etc/resolv.conf` file would look like this at the end:

```cfg
# Generated by dhcpcd from wlan0
# /etc/resolv.conf.head can replace this line
domain local
nameserver 1.1.1.1
nameserver 1.0.0.1
# /etc/resolv.conf.tail can replace this line
```


### Creating VPN-launching Script {#creating-vpn-launching-script}

Finally, you write the script that you will run to launch the VPN from the KOReader terminal.

A template for split-tunnelling is as follows:

```shell
#!/bin/sh
wireguard-go wg0
wg setconf wg0 $PATH_TO_WIREGUARD_CONFIG
ip link set wg0 up
ip addr add $WIREGUARD_KOBO_IP dev wg0
ip route add $WIREGUARD_NETWORK_ADDRESS dev wg0
ip route add $DESTINATION_NETWORK_ADDRESS dev wg0
```

What this does is:

-   Create a Wireguard interface
-   Configure the Wireguard interface according to the provided conf file
-   Activate the Wireguard interface
-   Give the Wireguard interface the IP address that the Kobo has been assigned in the Wireguard network
    -   **This is where you would paste the IPv4 address that you copied while editing the conf file.**
-   Tell the Kobo to use the Wireguard network interface when contacting IP addresses in the Wireguard network
-   Tell the Kobo to use the Wireguard network interface when contacting IP addresses in the local destination network

Using the variables:

-   `/mnt/onboard/wireguard-kobo.conf` is the location of the configuration file
-   The Kobo's IP address in the Wireguard network is `10.10.0.2`
-   The Wireguard network address is `10.10.0.0/24`
-   The destination network, which has local devices that you would like the Kobo to access, has the address `192.168.1.0/24`

<!--listend-->

```shell
#!/bin/sh
wireguard-go wg0
wg setconf wg0 /mnt/onboard/wireguard-kobo.conf
ip link set wg0 up
ip addr add 10.10.0.2/24 dev wg0
ip route add 10.10.0.0/24 dev wg0
ip route add 192.168.1.0/24 dev wg0
```

Save the script as `/usr/local/bin/vpn` and use `chmod +x` to make it executable.
Now to activate the Wireguard VPN, you need to go to KOReader Terminal and type `vpn`, then press enter. This will remain active until the device powers down (the VPN still remains active in sleep mode).

To deactivate the VPN, you can just restart the device, and it will have resumed regular network connectivity upon reboot.
To reactivate the VPN upon reboot, you simply need to enter the KOReader Terminal and type `vpn`, then press enter.

Congratulations! Now with the power of self-hosted book servers you can run, the options for your Kobo have dramatically expanded! 🥳
