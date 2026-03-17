+++
title = "Pipewire Setup for Single GPU Passthrough"
date = 2025-08-25
draft = false
showTableOfContents = true
+++

## Summary {#summary}

This is a guide that allows Virtual Machine audio input and output to run through the logged in user's Pipewire sound service when using Single GPU Passthrough in Linux.

Ordinarily, the user would have to either:

1.  Pass through the host sound card directly - which can lead to the sound card ceasing to function after terminating the VM.
2.  Purchase a USB soundcard, which can have degradation in sound quality compared to the computer's main sound card.

By having the VM run its sound through Pipewire, the user retains the quality and convenience of utilising the computer's sound card directly, while allowing the sound card to remain usable upon returning to the host OS.


## Prerequisites {#prerequisites}

-   Having a Virtual Machine with Single GPU Passthrough configured, as per [RisingPrism's guide.](https://gitlab.com/risingprismtv/single-gpu-passthrough/-/wikis/home)
-   Running a Linux host that uses **apparmor** as its security solution, it is unknown whether this method works with SELinux-based systems.


## Step 1: Adding QEMU user to audio group {#step-1-adding-qemu-user-to-audio-group}

Add the user you're running qemu as - the same user you run your Single GPU Passthrough VM with, if you followed Risingprism's guide - to the `audio` group, then **REBOOT** your PC. The PC must be restarted for the permission to fully set.

```shell
sudo usermod -aG audio $USER
```


## Step 2: Configure AppArmor for Libvirt {#step-2-configure-apparmor-for-libvirt}

By default, the AppArmor security service stops libvirt from accessing the Pipewire sound service directly. This step removes the restriction.

1.  Create folder if doesn't already exist: `/etc/apparmor.d/abstractions/libvirt-qemu.d`
2.  Create the text file file `/etc/apparmor.d/abstractions/libvirt-qemu.d/local-audio`
3.  Inside the file, write the following, replacing 1000 with UID of the user qemu is running with. [(Source)](https://superuser.com/a/1895915)
    ```nil
    # Allow libvirt QEMU VMs access to audio stuff (i.e. pipewire config files and
    # pipes)
    #include <abstractions/audio>
    "/run/user/1000/pipewire-0" rw,
    ```


## Step 3: Configure the VM XML {#step-3-configure-the-vm-xml}

Virutal Machine Manager needs to have XML editing enabled for this step.

1.  First, you need to find the names for the Pipewire input (your microphone) and output (your spekaers). You have to do this in the Terminal.
    1.  For input, run `pactl list short sources`. The resultant text should be an entry similar to `alsa_input.pci-0000_00_1f.3.analog-stereo`. Copy this into a text file.
    2.  For output, run `pactl list short sinks`. The resultant text should be an entry similar to `alsa_output.pci-0000_00_1f.3.analog-stereo`. Copy this into the same text file as the above.
2.  Open up the configuration for your VM in Virtual Machine Manager, then in the Overview tab, enter the XML section.
3.  Remove all existing `<audio>` and `<sound>` sections.
4.  In the place where the `<sound>` section used to be, add:
    ```xml
    <sound model="ich9">
      <codec type="micro"/>
      <audio id="1"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x1b" function="0x0"/>
    </sound>
    ```
5.  Now it's time to enter in the input and outputs you found in Step 1. Using the values from the example above, enter in the new audio section the below format [(source)](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Passing_audio_from_virtual_machine_to_host_via_PipeWire_directly).
    ```xml
    <audio id='1' type='pipewire' runtimeDir='/run/user/1000'>
      <input name='alsa_input.pci-0000_00_1f.3.analog-stereo' streamName='win11input'/>
      <output name='alsa_output.pci-0000_00_1f.3.analog-stereo' streamName='win11output'/>
    </audio>
    ```

    1.  The `1000` in `/run/user/1000` should be replaced with the user ID of the user you're running qemu as. If you followed Risingprism's guide, this should be _your_ user ID, which can be found by running `id -u` in the terminal.
    2.  The input and output in the `name` part of their respective sections.
    3.  The `streamName` label is optional, but is useful to keep track of what the configuration is.
6.  The full `sound` and `audio` section should now look similar to this:
    ```xml
    <sound model='ich9'>
      <codec type='micro'/>
      <audio id='1'/>
      <address type='pci' domain='0x0000' bus='0x0b' slot='0x00' function='0x0'/>
    </sound>
    <audio id='1' type='pipewire' runtimeDir='/run/user/1000'>
      <input name='alsa_input.pci-0000_00_1f.3.analog-stereo' streamName='win11input'/>
      <output name='alsa_output.pci-0000_00_1f.3.analog-stereo' streamName='win11output'/>
    </audio>
    ```


## Step 4: Keeping Pipewire Running Upon Logout {#step-4-keeping-pipewire-running-upon-logout}

By default, Pipewire is run as a user service, meaning it automatically stops if the user logs out.
Because starting a Single GPU Passthrough VM is acknowledged by the system as logging out, you need to modify the scripts that run upon your VM starting to keep user services running.

1.  Access the file `/etc/libvirt/hooks/qemu`. The default file installed by Risingprism's guide looks like:
    ```bash
    #!/bin/bash

    OBJECT="$1"
    OPERATION="$2"

    if [[ $OBJECT == "win10" ]]; then
            case "$OPERATION" in
                    "prepare")
                    systemctl start libvirt-nosleep@"$OBJECT"  2>&1 | tee -a /var/log/libvirt/custom_hooks.log
                    /usr/local/bin/vfio-startup 2>&1 | tee -a /var/log/libvirt/custom_hooks.log
                    ;;

                    "release")
                    systemctl stop libvirt-nosleep@"$OBJECT"  2>&1 | tee -a /var/log/libvirt/custom_hooks.log
                    /usr/local/bin/vfio-teardown 2>&1 | tee -a /var/log/libvirt/custom_hooks.log
                    ;;
            esac
    fi
    ```
2.  In the `"prepare")` section, above the `systemctl start libvirt-nosleep` line, add `loginctl enable-linger user` (replacing `user` with your actual username).
3.  In the `"release")` section, above the `systemctl stop libvirt-nosleep` line, add `loginctl disable-linger user` (replacing `user` with your actual username).
4.  The result should look like:
    ```bash
    #!/bin/bash

    OBJECT="$1"
    OPERATION="$2"

    if [[ $OBJECT == "win10" ]]; then
            case "$OPERATION" in
                "prepare")
                    loginctl enable-linger user
                    systemctl start libvirt-nosleep@"$OBJECT"  2>&1 | tee -a /var/log/libvirt/custom_hooks.log
                    /usr/local/bin/vfio-startup 2>&1 | tee -a /var/log/libvirt/custom_hooks.log
                    ;;

                "release")
                    loginctl disable-linger user
                    systemctl stop libvirt-nosleep@"$OBJECT"  2>&1 | tee -a /var/log/libvirt/custom_hooks.log
                    /usr/local/bin/vfio-teardown 2>&1 | tee -a /var/log/libvirt/custom_hooks.log
                    ;;
            esac
    fi
    ```

{{< alert >}}
**IMPORTANT NOTE!**

If you have user services that depend on the GPU, such as Sunshine, you need to also add a command to **stop AND disable** those services before the VM starts. Otherwise, your system could crash.
{{< /alert >}}


## Conclusion {#conclusion}

You should now have fully functioning sound on your Single GPU Passthrough Virtual Machine, without the need to directly pass a soundcard or USB peripheral through!
