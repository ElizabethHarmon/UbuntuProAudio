# A Pro Audio Tuning Guide for Ubuntu (and other Ubuntu/Debian-based distros)
[![Discord](https://img.shields.io/discord/985310454266081280.svg?label=Discord&logo=discord&logoColor=ffffff&color=7389D8&labelColor=6A7EC2)](https://discord.gg/GYeBgF7Em9)

Following this guide will hopefully allow you to get the best possible performance on Linux for professional audio needs. Even though these steps are well-tested, it is wise to research what each step accomplishes and why (the search engine is your friend :P ).

## Fundamentals

To get started after installing Ubuntu, you could try just steps 3 and 5 below. If you need to use windows plugins on Linux also follow step 11 (easy: wine-staging, more advanced but potentially more performance: wine-tkg). Based on your individual pro audio needs, workflows, hardware specifications and more, your mileage may vary. If you are still having audio performance issues, try following the full guide...

### Pipewire?

Ubuntu includes a way of switching to Pipewire (see https://ubuntuhandbook.org/index.php/2022/04/pipewire-replace-pulseaudio-ubuntu-2204/ & https://pipewire.org/ for more details). You may choose to wait until it ships as default in future releases although it is just as easy to roll things back. To switch to Pipewire run:

```shell
sudo apt install pipewire-audio-client-libraries libspa-0.2-bluetooth libspa-0.2-jack
sudo apt install wireplumber pipewire-media-session-
sudo cp /usr/share/doc/pipewire/examples/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d/
sudo cp /usr/share/doc/pipewire/examples/ld.so.conf.d/pipewire-jack-*.conf /etc/ld.so.conf.d/
sudo ldconfig
sudo apt remove pulseaudio-module-bluetooth
systemctl --user --now enable wireplumber.service
pactl info
```
Be sure to say 'yes' to removing conflicting packages. Reboot! It would also be wise to install a graph manager like qpwgraph to be able to make connections between apps and devices:

```shell
sudo apt install qpwgraph
```
That should give you everything you need to get up and running. I consider Pipewire ready for primetime at this point. For Debian-specific instructions please see For Debian in particular see https://wiki.debian.org/PipeWire.

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

See https://gitlab.freedesktop.org/pipewire/pipewire/-/wikis/Config-PipeWire#setting-buffer-size and https://gitlab.freedesktop.org/pipew...ormance-tuning for more details. 

#### Back to ALSA/Pulse?

In the unlikely event you need to switch back:

```shell
sudo apt remove pipewire-audio-client-libraries libspa-0.2-bluetooth libspa-0.2-jack
sudo apt install pipewire-media-session wireplumber-
rm -f ~/.config/systemd/user/pipewire-session-manager.service
systemctl --user --now enable pipewire-media-session
```

If the sound still isn't working:

```shell
systemctl --user --now disable pipewire-pulse.service pipewire-pulse.socket
systemctl --user --now reenable pulseaudio.service pulseaudio.socket
```

substituting --user for --global if you originally enabled globally.

## Full In-depth Guide

### Install a flavor of Ubuntu (or other favorite Ubuntu-based or Debian-based distro)

To make your life easier, install either Ubuntu Studio or AVLinux. Almost all of the following tweaks are taken care of. Otherwise, pick a regular distro such as Ubuntu, MXLinux etc.

### Install a low-latency kernel (Ubuntu-based)

```shell
sudo apt update && sudo apt upgrade -y
sudo apt install linux-lowlatency
reboot
```
Or, for even better performance:

#### Liquorix (Debian-based or Ubuntu-based)

```shell
sudo apt update && sudo apt upgrade -y
sudo apt install software-properties-common apt-transport-https wget ca-certificates gnupg2 ubuntu-keyring -y
sudo add-apt-repository ppa:damentz/liquorix -y
sudo apt update
sudo apt-get install linux-image-liquorix-amd64 linux-headers-liquorix-amd64 -y
reboot
```

### rtcqs (formerly known as realtimeconfigquickscan)
    
```shell
git clone https://codeberg.org/rtcqs/rtcqs.git
cd rtcqs
./src/rtcqs/rtcqs.py
```

### Add user to audio group and configure realtime privileges

I believe that installing jackd2 takes care of the following these days. It is always worth double-checking especially if using pipewire.

```shell
sudo nano/etc/security/limits.d/audio.conf
```
Add the following lines:

```shell
@audio   -  rtprio     95
@audio   -  memlock    unlimited
```

Then create an audio group (if it doesn't exist already) and add your user to it:

```shell
sudo groupadd audio 
sudo usermod -a -G audio $USER
```

Log out/in or reboot...

### Add "threadirqs" as kernel parameter

```shell
sudo nano /etc/default/grub
```
change 
`GRUB_CMDLINE_LINUX=""` to `GRUB_CMDLINE_LINUX="threadirqs"`
    
```shell
sudo update-grub
```

### Set governor to "performance"

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
sudo nano /etc/default/cpupower # uncomment governor and change to performance
systemctl enable --now cpupower.service
systemctl start cpupower.service
```
    
### Swappiness

```shell
sudo nano /etc/sysctl.d/99-sysctl.conf
```
add "vm.swappiness=10"
    
###  Spectre/Meltdown Mitigations

If you run `rtcqs.py` and it gives you a warning about Spectre/Meltdown Mitigations, you could add `mitigations=off` to GRUB_CMDLINE_LINUX. Warning: disabling these mitigations will make your machine less secure! https://wiki.linuxaudio.org/wiki/system_configuration#disabling_spectre_and_meltdown_mitigations

### base-devel (as necessary)

```shell
sudo pacman -S --needed base-devel
```

### Install udev-rtirq

```shell
git clone https://github.com/jhernberg/udev-rtirq.git
cd udev-rtirq
sudo make install
reboot
```

### Jack2 + Jack D-Bus (__skip this step if you switched to Pipewire__)

```shell
sudo apt install qjackctl jackd2
```
Enable Jack D-Bus interface:  
![image](https://user-images.githubusercontent.com/79659262/124497122-51218300-ddb2-11eb-8cb8-4bf873e026cd.png)


### DAW & Plugins

REAPER: 
http://reaper.fm/download.php 

change RT priority to 40 on audio device page?  


Also be sure to check out Bitwig Studio, Tracktion Waveform, Ardour, Mixbus, Qtractor, LMMS, Rosegarden, Zrythm etc...
https://en.wikipedia.org/wiki/List_of_Linux_audio_software#Digital_audio_workstations_(DAWs)

#### Native plugins
- x42-plugins (https://x42-plugins.com/x42/)
- airwindows-git (http://www.airwindows.com/)  
- lsp-plugins  (https://lsp-plug.in/)
- zam-plugins  (http://www.zamaudio.com/?p=976)
- distrho-ports (https://distrho.sourceforge.io/ports.php)
- dpf-plugins (https://distrho.sourceforge.io/plugins.php)
- dragonfly-reverb (https://michaelwillis.github.io/dragonfly-reverb/)
- Bertom Denoiser (https://www.bertomaudio.com/denoiser.html) (not in the Arch repos or AUR)
- sfizz / sfizz-git (https://sfz.tools/sfizz/)

A brilliant resource for Debian- and Ubuntu-based distros is  https://kx.studio/. Add the repo by downloading and installing the [KXStudio repo](https://launchpad.net/~kxstudio-debian/+archive/kxstudio/+files/kxstudio-repos_11.1.0_all.deb).

### Wine-staging or Wine-tkg

Perhaps start with vanilla wine-staging and see how you fare in terms of performance. If your workflows rely heavily on VSTi like Kontakt, you may find better performance with wine-tkg (fsync enabled). 

#### Wine-staging

```shell
sudo apt install wine-staging
```

Or, install a particular version that you know is compatible (in this case v6.14):

```shell
version=6.14
variant=staging
codename=$(shopt -s nullglob; awk '/^deb https:\/\/dl\.winehq\.org/ { print $3; exit }' /etc/apt/sources.list /etc/apt/sources.list.d/*.list)
suffix=$(dpkg --compare-versions "$version" ge 6.1 && ((dpkg --compare-versions "$version" ge 6.17 && echo "-2") || echo "-1"))
sudo apt install --install-recommends {"winehq-$variant","wine-$variant","wine-$variant-amd64","wine-$variant-i386"}="$version~$codename$suffix"
```

To prevent the package being updated:

```shell
sudo apt-mark hold winehq-staging
```
    
Check https://github.com/robbert-vdh/yabridge#tested-with for up-to-date info.

OR...for the more adventurous:
   
#### Wine-tkg
   
Either download wine-tkg from https://github.com/Frogging-Family/wine-tkg-git/releases or follow the instructions to git clone and install latest version: https://github.com/Frogging-Family/wine-tkg-git/tree/master/wine-tkg-git#quick-how-to-

NOTE: In order to take advantage of fsync, add `export WINEFSYNC=1` to your shell's enviroment profile. See https://github.com/robbert-vdh/yabridge#environment-configuration for more information. 
       
### Install yabridge

i. Please follow the instructions at https://github.com/robbert-vdh/yabridge#usage

To begin, download the latest release from https://github.com/robbert-vdh/yabridge/releases and run:

```shell
tar -C ~/.local/share -xavf yabridge-x.y.z.tar.gz
```

where x.y.z is the version number such as 4.0.1. Don't forget to add yabridgectl to your shell's search path by adding `export PATH="$PATH:$HOME/.local/share/yabridge` to the end of ~/.bashrc. Close then re-open the terminal.

ii. Configure yabridge according to https://github.com/robbert-vdh/yabridge#readme  
iii. if using wine-tkg, append "export WINEFSYNC=1" to ~/.bash_profile 

```shell
nano ~/.bash_profile
. ~/.bash_profile
```

Or, if your shell is not bash, use the guide here: https://github.com/robbert-vdh/yabridge#environment-configuration

iv. Install Windows VST2 and VST3 (preferred) plugins
    
    
### Thanks
Audiojunkie for various links and clarifications including KXStudio.




