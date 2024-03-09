# How to customize the framework for building the RPI image, written by Rod.

Here is the link to the branch I worked with: 
[`rpi-img-builder-lcnc` 2.9.2 made by Rod](https://github.com/rodw-au/rpi-img-builder-lcnc/tree/linuxcnc-2.9.2).

I would like to start with a generic build. But this is just a first step in the customization of your own image. If you need in just a generic version, you can simply download it from here: https://linuxcnc.org/downloads/.

| 💡 Use balena Etcher 💡 |
| :--- |
| I didn't have success with the original Raspberry Pi Imager, but I have 100% success with [balena Etcher](https://etcher.balena.io/#download-etcher) tool. |

## 0. Long story short

I tried to split all the changes into step-by-step atomic commits to demonstrate the possible ways to reach the particular customization goal. Here they on by one:
https://github.com/golyakoff/rpi-img-builder-lcnc-ago/compare/rodw-au%3Arpi-img-builder-lcnc%3Alinuxcnc-2.9.2...linuxcnc-2.9.2

## 1. Basic customization (easy level)

https://github.com/rodw-au/rpi-img-builder-lcnc/commit/6dd00416d588c04be1fd9d645595223634f78b4d?diff=split&w=1

<details>
<summary><b>Explanation of the changes</b></summary>

- custom.txt
    - ```IMGSIZE="16384MB"```
    
        Comment from Rod:
        > This is how much memory is used when building the image. This might let you use the pi imager to burn the image. 
        
        My understanding:
        > The bigger, the better. But not too big, i.e., it should fit the size of the SD Card I am going to use for burning the image. In my case, it is 16 GB.

    - ```HOSTNAME="pi-cnc"```
        > The name of the host of the Pi machine (it can be checked after the first boot with ```hostname``` command).
- userdata.txt
    - ```NAME="AndreyCNC"``` 
        > The full name of the main non-root user for the target Pi machine.  
        ⚠️ Pay attention: the spaces are not assumed here; the script will fail in that case!
    - ```USERNAME="ago"```
        > The login name of the main non-root user for the target Pi machine (it can be checked after the first boot with ```whoami``` command).
    - ```PASSWORD="2"```
        > The password for the main non-root user for the target Pi machine.
    - ```ROOTPASSWD="1"```
        > ```1``` to set up the initial password "```toor```" for the root user. It really doesn't matter because you can change it with ```sudo passwd root``` command whenever you want in the future.
    - ```VERBOSE="1"```
        > I like to see more messages; it is very helpful for the experiments.
    - ```KBUSER="ago"```
        > The login name of the user whose session will be used for the building process. In my case, it is the same (```ago```) as the name for the target machine, so don't be confused with that. I suggest you to check your username with ```whoami``` command.
    - ```KBHOST="pc-cnc"```
        > The host name, which will be used for the building process. In my case, it is very similar to the host name of the target Pi machine, so don't be confused with that. I suggest you to check your host name with ```hostname``` command.
    - ```TIMEZONE="Europe/Moscow"```
        > I updated the time zone according to the list here: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones.
- profile.txt
    - According to the code of `lib/dialog/config` script, this file should be generated from the values selected in the console GUI after calling ```make config```.  
    I didn't use it. I filled in all the values manually instead, mostly copying them from two previous configuration files.
    If you take a look at ```Makefile```, you will find there that calling the GUI is not the only action for ```make config``` command. It also contains the access permissions setup lines:
        ``` bash
        chmod go=rx files/inits/*
        chmod go=rx files/scripts/*
        chmod go=r files/misc/*
        chmod go=r files/users/*
        ```
This is it.
</details>

__To apply the changes, follow the steps__ (these steps are relevant to any changes):
1. Run 
    ```chmod go=rx files/inits/*```  
    ```chmod go=rx files/scripts/*```  
    ```chmod go=r files/misc/*```  
    ```chmod go=r files/users/*```    
2. Run ```sudo make menu```
3. Select the option with your board > `OK`
4. Select  `Make flashable image` > `OK`
5. Wait...
6. After a while, you will be prompted to confirm the kernel configuration. Rod's default configuration is pretty good, so click `Exit`
7. Wait for the result (my time is ~30 minutes on a 16-core virtual machine)!

## 2. Updated kernel & RT-patch, default config, cmdline

https://github.com/rodw-au/rpi-img-builder-lcnc/commit/23f53fd6f9072c1be66fc40f66a6cdefa6a9d635

<details>
<summary><b>Explanation</b></summary>

By default, the commit fixed in the `userdata.txt` is [342c7ee49e862edc30c893f141f55b9211b7a43b](https://github.com/raspberrypi/linux/commit/342c7ee49e862edc30c893f141f55b9211b7a43b).

If you open this link, you will find that it is related to `rpi-6.1.y` kernel version and its date is `Dec 21, 2023`.

Then let's check the corresponding patch in `userpatches` folder. It is `patch-6.1.69-rt21.patch`.  
We can check here https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/6.1/older/ that it was released on `29-Dec-2023 05:46`.

So, my conclusion is:  
The RT patch should be the same kernel version (`6.1`) and it has to be released as closed as possible to the RPI image commit.+  
⚠️ Not very strict, honestly, but I didn't understand how to define it more carefully! ⚠️

Let's update the RT patch and change the value of the RPI commit for the corresponding one:
1. Download the latest RT patch here https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/6.1/older/ and check its release date:   
In my case, it is `patchs-6.1.77-rt24.patch.gz` from `08-Feb-2024`.
2. Gunzip it (```gunzip patchs-6.1.77-rt24.patch.gz```) and put it to `userpatches` folder INSTEAD of the previous RT patch file (`patch-6.1.69-rt21.patch` should be removed).
3. Open the RPI Linux kernel sources of the 6.1.y branch and find commits nearby `08-Feb-2024`: https://github.com/raspberrypi/linux/commits/rpi-6.1.y/?since=2024-01-20&until=2024-02-20.  
I found the key commit of `6.1.77` on `Feb 5`, and decided to get also several further commits (the fixes provided look good to take from my perspective).  
So my choose was:
    - Commit from Feb 17, 2024
    - SHA eb06d31da3e2025a2e578d8de9843e24b68137a6
4. Update the fixed commit value in `userdata.txt` with the new value "`eb06d31da3e2025a2e578d8de9843e24b68137a6`".

__Additional, but not necessary:__

Due to having more than one experiment run, I decided to save time, saved the predefined kernel configuration from the kernel confgiguration menu, and customized the build process to use it as a default:
1. Save default the kernel configuration from the GUI;
2. Find the file somewhere within the working folders of RPI Image Builder
    (something like ```find . -name "your.def.config.name"```);
3. Move it to the `defconfig` folder of RPI Image Builder with the name suffix `_defconfig`. In my case, it is `kernel-v8-ago_defconfig`;
4. Update `userdata.txt` to use this config:
    ```bash
    CUSTOM_DEFCONFIG="1"
    MYCONFIG="kernel-v8-ago_defconfig"
    ```

By the way, I updated the `cmdline` via changes in `files/userscripts/uscripts`: added `nohz_full=2,3 rcu_nocbs=2,3` to the end of `cmdline` string.

</details>

## 3. Overclocking and tuning configuration in config.txt

https://github.com/rodw-au/rpi-img-builder-lcnc/commit/f945f644741a3b5129ed22a59443e6b151cb64e6

Here may be any changes to target `/boot/broadcom/config.txt`.

## 4. Extend the bash prompt (and update the root password)

https://github.com/rodw-au/rpi-img-builder-lcnc/commit/333a346b523b1c5960df4f6e52616baa33736bc8

- Add colored `(__git_ps1)` to the prompt if we are in the git repository folder;
- Add some aliases:
    - `alias l='ls -FlatG'`  
    - `alias d='diff --suppress-common-lines --color=always'`
    - `alias grep='grep --color=auto'`
- Preserve bash history in multiple terminal windows.

The previous (default) `~/.bashrc` file was saved and can be found here `~/.bashrc~`.

## 5. Add Sublime and VS Code

https://github.com/rodw-au/rpi-img-builder-lcnc/commit/8462bba863d8a2792bd3c550c34edb4543a48405

They are separated as the installation requires custom GPG keys and additional source lists.

## 6. Add proftpd, x11vnc as a service

https://github.com/rodw-au/rpi-img-builder-lcnc/commit/dc3503ca303f50570cc7cefac5cb99b1b9421bbe

VNC server x11vnc can be managed with RealVNC clients for desktop / Android.  
⚠️ x11vnc requires a physically connected display (or stub).

## 7. Setup predefined Network Manager connections

https://github.com/rodw-au/rpi-img-builder-lcnc/commit/1a1e52431e995d33d2fcd60c69927df0a018af6d

Creating default connections for:
- Mesa on 10.10.10.10
- Home Wi-Fi (autoconnect with high priority)
- Rescue Mobile Wi-Fi (autoconnect with low priority)