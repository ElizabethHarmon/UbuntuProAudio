# A Pro Audio Tuning Guide for Manjaro (and other Arch-based distros)

Following this guide will hopefully allow you to get the best possible performance on Linux for professional audio needs. Even though these steps are well-tested, it is wise to research what each step accomplishes and why (the search engine is your friend :P ).

## Fundamentals

To get started after installing Manjaro, you could try just steps 3 and 5 below. If you need to use windows plugins on Linux also follow step 11 (easy: wine-staging, more advanced but potentially more performance: wine-tkg). Based on your individual pro audio needs, workflows, hardware specifications and more, your mileage may vary. If you are still having audio performance issues, try following the full guide...

### Pipewire?

Manjaro includes an extremely convenient way of switching to Pipewire (see https://pipewire.org/ for more details). You may choose to wait until it ships as default in future releases although it is just as easy to roll things back. To switch to Pipewire run:
```shell
yay -S manjaro-pipewire pipewire-jack
```
Be sure to say 'yes' to removing conflicting packages. Reboot! It would also be wise to install a graph manager like qpwgraph to be able to make connections between apps and devices:
```shell
yay -S qpwgraph
```
That should give you everything you need to get up and running. I consider Pipewire ready for primetime at this point. 

#### Pipewire configuration

If you want to change the default samplerate, buffer size etc, you need to copy `/usr/share/pipewire/pipewire.conf` over to `/etc/pipewire/` and uncomment a few lines:

![2022-04-19_09-19](https://user-images.githubusercontent.com/90937680/163958025-f25a0f05-bc53-4fca-8f7f-28a94308407b.png)

To temporarily change samplerate/buffer size do not use PIPEWIRE_LATENCY environment variable. Instead, use:
```shell
pw-metadata -n settings 0 clock.force-rate <samplerate>
```
and
```shell
pw-metadata -n settings 0 clock.force-quantum <buffer-size>
```
To return to default values:
```shell
pw-metadata -n settings 0 clock.force-rate 0
```
and
```shell
pw-metadata -n settings 0 clock.force-quantum 0
```

See https://gitlab.freedesktop.org/pipewire/pipewire/-/wikis/Config-PipeWire#setting-buffer-size for more details.

#### Back to ALSA/Pulse?

In the unlikely event you need to switch back:
```shell
yay -S manjaro-pulse
```
Again, say 'yes' to removing conflicts (and then reboot).

## Full In-depth Guide

1) Install Manjaro KDE (or other favorite Arch-based distro)

    _Optional:_
    Install `yay`:
    ```shell
    pacman -S yay
    ```
    Or, build from source:
    ```shell
    pacman -S --needed git base-devel
    git clone https://aur.archlinux.org/yay.git
    cd yay
    makepkg -si
    ```

2) rtcqs (formerly known as realtimeconfigquickscan)
    ```shell
    git clone https://codeberg.org/rtcqs/rtcqs.git
    cd rtcqs
    ./src/rtcqs/rtcqs.py
    ```

3) install realtime-privileges
    ```shell
    yay -S realtime-privileges
    ```
    Add user to "realtime" group and "audio" group (to satisfy `rtcqs`)
    ```shell
    sudo usermod -a -G realtime,audio $USER
    ```
    Log out/in or reboot...

4) add "threadirqs" as kernel parameter
    ```shell
    sudo nano /etc/default/grub
    ```
    change 
    `GRUB_CMDLINE_LINUX=""` to `GRUB_CMDLINE_LINUX="threadirqs"`
    
    ```shell
    sudo update-grub
    ```

5) Set governor to "performance"

    i. Temporary:
    ```shell
    sudo cpupower frequency-set -g performance
    ```
    ii. Permanent:
    
    Add `cpufreq.default_governor=performance` as a kernel parameter:
   
    ```shell
    sudo nano /etc/default/grub
    ```
    LIne should now read: 
     `GRUB_CMDLINE_LINUX="cpufreq.default_governor=performance threadirqs"`
     ```shell
    sudo update-grub
    ```
    or, for kernels < 5.9:
    ```shell
    sudo nano /etc/default/cpupower (uncomment governor and change to performance)
    systemctl enable --now cpupower.service
    systemctl start cpupower.service
    ```


6) swappiness
    ```shell
    sudo nano /etc/sysctl.d/99-sysctl.conf
    ```
    add "vm.swappiness=10"

7) base-devel (as necessary)
    ```shell
    sudo pacman -S --needed base-devel
    ```

8) install udev-rtirq
    ```shell
    git clone https://github.com/jhernberg/udev-rtirq.git
    cd udev-rtirq
    sudo make install
    reboot
    ```

9) Jack2 + Jack D-Bus (__skip this step if you switched to Pipewire__)
    ```shell
    yay -S qjackctl jack2-dbus
    ```
    Enable Jack D-Bus interface:  
    ![image](https://user-images.githubusercontent.com/79659262/124497122-51218300-ddb2-11eb-8cb8-4bf873e026cd.png)


10) DAW & Plugins

    REAPER: 
    http://reaper.fm/download.php or,  
    ```shell
    yay -S reaper-bin
    ```
    change RT priority to 40 on audio device page?  
    
    
    Also be sure to check out Bitwig Studio, Tracktion Waveform, Ardour, Mixbus, Qtractor, LMMS, Rosegarden, Zrythm etc...
    https://en.wikipedia.org/wiki/List_of_Linux_audio_software#Digital_audio_workstations_(DAWs)

* Native plugins (`yay -S [pkgname]`)
  * x42-plugins (https://x42-plugins.com/x42/)
  * airwindows-git (http://www.airwindows.com/)  
  * lsp-plugins  (https://lsp-plug.in/)
  * zam-plugins  (http://www.zamaudio.com/?p=976)
  * distrho-ports (https://distrho.sourceforge.io/ports.php)
  * dpf-plugins (https://distrho.sourceforge.io/plugins.php)
  * dragonfly-reverb (https://michaelwillis.github.io/dragonfly-reverb/)
  * Bertom Denoiser (https://www.bertomaudio.com/denoiser.html)
  * sfizz / sfizz-git (https://sfz.tools/sfizz/)

11) Wine-staging or Wine-tkg

    Perhaps start with vanilla wine-staging and see how you fare in terms of performance. If your workflows rely heavily on VSTi like Kontakt, you may find better performance with wine-tkg (fsync enabled). 

    #### Wine-staging

    ```shell
    yay -S wine-staging
    ```

    Or, install a particular version that you know is compatible:
   
    ```shell
    yay -S downgrade
    sudo DOWNGRADE_FROM_ALA=1 downgrade wine-staging
    ```
    
    Check https://github.com/robbert-vdh/yabridge#tested-with for up-to-date info.
    
    OR...for the more adventurous:
   
    #### Wine-tkg
   
    Either download wine-tkg from https://github.com/Frogging-Family/wine-tkg-git/releases or follow the instructions to git clone and install latest version: https://github.com/Frogging-Family/wine-tkg-git/tree/master/wine-tkg-git#quick-how-to-
   
    NOTE: In order to take advantage of fsync, add `export WINEFSYNC=1` to your shell's enviroment profile. See https://github.com/robbert-vdh/yabridge#environment-configuration for more information. 
       
12) Install yabridge

    ```shell
    yay -S yabridge-bin
    ```
    or for latest git:
    ```shell
    yay -S yabridge-git yabridgectl-git
    ```
    
    iii. Configure yabridge according to https://github.com/robbert-vdh/yabridge#readme  
    iv. if using wine-tkg, append "export WINEFSYNC=1" to ~/.bash_profile  
    ```shell
    nano ~/.bash_profile
    . ~/.bash_profile
    ```
    
    Or, if your shell is not bash, use the guide here: https://github.com/robbert-vdh/yabridge#environment-configuration
    
    v. Install Windows VST2 and VST3 (preferred) plugins
    
    
   ### Thanks
   Robbert van der Helm (of yabridge fame) for pointing out typos/omissions and suggesting alternative methods. JamesPeters (REAPER forums) for suggesting expansion of steps 3, 4 & 5. graves501 for rtcqs info update, audiojunkie for the latest title, Soli Deo Gloria for wine-tkg suggestions, Damien Zammit (of zam-plugins fame) and MikeLupe for discussion of other Linux DAWs including Ardour and Zrythm.




