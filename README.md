# Fedora KDE Yubikey 2FA U2F Setup Guide
A guide to setup a Yubikey physical security key as a second factor authentication (2FA) using U2F within Fedora KDE and SDDM. Fedora Linux using the KDE Desktop Environment (DE) can use a Yubikey as 2FA for the following login types:

* login screen (both initial and lock screen)
* sudo
* su

Using these directions will require a manually entered password AND the Yubikey to not only be plugged in to a USB port, but the button pressed. This means that even if someone gets your password they would be unable to login without your physical Yubikey.

### What This Process Will NOT Do
These steps will NOT require the Yubikey for SSH or any other form of remote access. It only applies 2FA to the above listed logins. It also does not replace using a manually entered password to login. It can potentially be modified to do so, but is beyond the scope of this guide.

### Warnings!!
* Use these steps at your own risk! I, nor anyone else, are responsible if you execute any of the steps in this guide and loose access to your system.
* Test this before using seriously. Setup a VM or a spare machine and ensure you have the steps down and confirm it works completely, including after a reboot.
* I highly recommend having at least 1 spare key if not more (IE: 2 or more total keys) in the event you lose or break a key.
* Do NOT follow this guide unless you understand the implications of what it does and that it can prevent access to a system if used incorrectly

Be aware that if you do not also encrypt your drive that it may be possible to circumvent the need for the Yubikey/2FA. I would also recommend password protecting your UEFI/BIOS along with encrypting your drive, but that is up to you.

## Outline
The following are the overall steps we will take to setup the Yubikey for 2FA using U2F.

1. Required Packages
2. Setup Keys/Mapping
3. Test Key/Mapping
4. Implement 2FA
   1. Sudo 2FA
   2. SU 2FA
   3. KDE Initial Login 2FA
   4. KDE Lock Screen 2FA

Of the "Implementation" options, you may use all of them or only the ones you want. IE: you can setup Yubikey for `KDE initial login` and nothing else or you could setup `sudo`, `su` and `initial login` but not the `lock screen`.

Be aware that if someone gets your password (assuming you are a sudo user) or the root password and 2FA is not enabled for `sudo` and/or `su`, that it will be potentially possible to bypass the need for 2FA elsewhere.

## Requirements
To follow this guide you will need the following:

1. A Yubikey (I have only tested with a Yubikey 5 NFC but the directions should apply to all recent Yubikeys)
2. A free USB port matching the type of your Yubikey
3. Fedora with the KDE DE installed and setup. (I have only tested on Fedora 30, may work with previous versions)
4. Time to read, understand, test and implement the steps in this guide.

### Required Packages
It is assumed you have Fedora installed with the KDE DE setup and ready. Only 2 base packages need to be installed manually for the Yubikey with U2F to work:

1. pam-u2f
2. pamu2fcfg

To install them use the following command:
```bash
sudo dnf install pam-u2f pamu2fcfg -y
```

This will install the required packages and their dependencies.

(The `-y` parameter accepts the install without human response, omit if you want to review packages to be installed before installing them.)

While not required, I would recommend a reboot just to be sure.

## Setup Keys/Mapping
At this stage we will essentially create a file(s) that the Yubikey authenticates against. We have to make a choice that will impact all of our configuration going forward.

The major consideration will be how many people are using the machine. If:

1. It is yourself and root only, then we can take the approach of creating 2 files, one for us and one for root.
2. If there are many users it's better to create a single file that maps to each user. You could use this for a single user environment as well.

I will refer to 1 as `single user` and 2 as `multi-user` going forward. From now on, stick to either `single user` or `multi-user` directions, do NOT do both.

### Single User
We will need to create 2 files, one in the user directory and one in the root user directory.

Your User:
1. Be in your users home folder, if need be use `cd /home/user_name` and replace `user_name` with your actual user name.
2. Execute `mkdir ~/.config/Yubico` to create the directory
3. Run `pamu2fcfg > ~/.config/Yubico/u2f_keys`
  * For each additional physical Yubikey you want to have as backups run the following `pamu2fcfg -n >> ~/.config/Yubico/u2f_keys`

For Root:
1. Switch to the root user using `su`
2. Change to the root directory via `cd /root`
3. Repeat steps 2-3 from the "You User" section.

### Multi-User
We are going to create a single file and store all user Yubikey auth info in the file.

Run the following commands:
1. `pamu2fcfg -u user_name > /etc/u2f_mappings` (replace user_name with the actual user name)
  * For each following user run `pamu2fcfg -u user_name >> /etc/u2f_mappings` (replace user_name with the actual user name)
2. `pamu2fcfg -u root >> /etc/u2f_mappings`

## Testing Keys/Mapping
To ensure the steps taken for key setup/mapping has worked, enabling 2FA on sudo is likely the fastest and safest method for testing.

To test, follow the steps given in the Sudo 2FA section. Make sure to keep the terminal window you make the change in open, if you close this and are unable to login as sudo you may en up being locked out of sudo for good.

Open a second terminal window and execute `sudo echo test`. You should be prompted to touch your Yubikey button and then enter your password before seeing "test" in stdout. If not recheck the steps from Sudo 2FA in the first terminal window and try again.

If it works and you do not want to enforce 2FA on sudo long term, simply remove the line added in the directions to `/etc/pam.d/sudo`.

## Implement 2FA
This is where we will enable using the Yubikey as 2FA via U2F for each login type we desire.

### Sudo 2FA
To enforce 2FA using U2F with your Yubikey for `sudo`, do the following:

1. `sudo vi /etc/pam.d/sudo`
2. Above the line `auth include system-auth` insert the following:
  * **Single User** `auth required pam_u2f.so cue`
  * **Multi-User** `auth required pam_u2f.so authfile=/etc/u2f_mappings cue`
3. To save and exit `:wq!`

Note that `cue` on the end of the added line displays a prompt in the terminal when it's time to press the button on your Yubikey. The message reads "Please touch the device".

When attempting to use `sudo` in the terminal, you will now be prompted to touch the button on the Yubikey and then be prompted for your password. If the key is not connected or you do not press the button in a reasonable time frame authentication will fail and not allow executing the command as sudo.

### SU 2FA
To enforce 2FA using U2F with your Yubikey for `su`, do the following:

1. `sudo vi /etc/pam.d/su`
2. Below the line `auth sufficient pam_rootok.so` insert the following:
  * **Single User** `auth required pam_u2f.so cue`
  * **Multi-User** `auth required pam_u2f.so authfile=/etc/u2f_mappings cue`
3. To save and exit `:wq!`

Note that `cue` on the end of the added line displays a prompt in the terminal when it's time to press the button on your Yubikey. The message reads "Please touch the device".

For `su` to work with the Yubikey, whichever user you attempt to su into needs to have a key assigned to it. If going by the single user directions, this means that the user needs to have a `u2f_keys` file. If using the multi-user directions, then you must have mapped that user into `u2f_mappings` with the desired key(s).

When attempting to use `su` in the terminal, you will now be prompted to touch the button on the Yubikey and then be prompted for your password. If the key is not connected or you do not press the button in a reasonable time frame authentication will fail and not allow executing the su command.

### KDE Initial Login 2FA
To enforce 2FA using U2F with your Yubikey for the initial login to KDE via SDDM. This will not require the Yubikey to login when the screen has been locked.

1. `sudo vi /etc/pam.d/sddm`
2. Below the line `auth substack password-auth` insert the following:
  * **Single User** `auth required pam_u2f.so`
  * **Multi-User** `auth required pam_u2f.so authfile=/etc/u2f_mappings`
3. To save and exit `:wq!`

Note that `cue` does not work for the login screen like it does for sudo/su.

When initially logging into the GUI, you will be prompted for your password. There will be no prompt to press the button on the Yubikey, but the button will start blinking after entering the password. Once the password is entered/submitted and the button blinks, press it to finish logging in. If the key is not connected or you do not press the button in a reasonable time frame authentication will fail and not login.

To be clear, the only indication of when and that the Yubikey is required and needs the button pressed is after entering and submitting the password. It is only in the form of the button on the Yubikey blinking at that time. If you fail to press it or do not have the key inserted there will be no message about failing to login.

### KDE Lock Screen 2FA
To enforce 2FA using U2F with your Yubikey for the lock screen of KDE/SDDM. This will require the Yubikey to login any time the screen has been locked.

1. `sudo vi /etc/pam.d/kde`
2. Below the line `auth substack password-auth` insert the following:
  * **Single User** `auth required pam_u2f.so`
  * **Multi-User** `auth required pam_u2f.so authfile=/etc/u2f_mappings`
3. To save and exit `:wq!`

Note that `cue` does not work for the login screen like it does for sudo/su.

When logging into the lock screen, you will be prompted for your password. There will be no prompt to press the button on the Yubikey, but the button will start blinking after entering the password. Once the password is entered/submitted and the button blinks, press it to finish logging in. If the key is not connected or you do not press the button in a reasonable time frame authentication will fail and not login.

To be clear, the only indication of when and that the Yubikey is required and needs the button pressed is after entering and submitting the password. It is only in the form of the button on the Yubikey blinking at that time. If you fail to press it or do not have the key inserted there will be no message about failing to login.

## Resources
Here are some of the resources that helped me piece this together. Many are not specific to Fedora or KDE but gave me a point of reference. Some are also using different methods/protocols while others are dated or incomplete. The Fedora specific Yubico support article is pretty bad.

I created this guide as a resource for myself and others as I could not find clear, up-to-date directions specific to Fedora and KDE/SDDM to setup my Yubikeys for 2FA using U2F for logins.

* My reddit post in which the user abitstick was very helpful in working this out. Thanks! [How Do I Setup Yubikey with Fedora 30 KDE/SDDM (and maybe LUKS v2)?](https://www.reddit.com/r/linuxquestions/comments/bwcn78/how_do_i_setup_yubikey_with_fedora_30_kdesddm_and/?utm_source=share&utm_medium=web2x)
* The Yubico support article for RHEL using challenge/response vs U2F. [Red Hat Linux Login Guide - Challenge Response](https://support.yubico.com/support/solutions/articles/15000011445-red-hat-linux-login-guide-challenge-response)
* The Yubico support article for Ubuntu using U2F. [Ubuntu Linux Login Guide - U2F](https://support.yubico.com/support/solutions/articles/15000011356-ubuntu-linux-login-guide-u2f)
* A blog post I found specific to Mint and U2F. Helped with mapping and has clear examples. [TREZOR/U2F Login into Your Linux Mint](https://blog.trezor.io/trezor-u2f-login-into-your-linux-mint-bd3684d4a8ba)

For those interested, I have another guide [Fedora KDE Minimal Install](https://github.com/Zer0CoolX/Fedora-KDE-Minimal-Install-Guide) to install Fedora Linux with the KDE Plasma Desktop Environment (DE) from a minimal Fedora installation. I do this to limit the bloat and have more control over what gets installed on my system versus using the official KDE spin.

## Final Thoughts
This is a work in progress. I intend to update it as I gain a better understanding of how it all works. Should you find a problem or have a suggestion for how to make the guide better please submit an issue, providing as much detail as you can.

There are a few things I would like to find out if they are possible and if so how to do (so any help would be appreciated):
1. Having sudo/su prompt for the password before prompting for the Yubikey. Seems odd to me to request the key/button press before entering a password.
2. Having some sort of prompt or message on the login/lock screen for when the Yubikey input is needed. Static is better than nothing, but a dynamic prompt/message would be neat.
3. From a security standpoint, is the `single user` or `multiuser` method better? Is neither good or are either ok?
4. Potentially adding directions for using the Yubikey for remote SSH as well. This could be a very different process.

It is my hope that this guide will help others accomplish the same goals I have set out to accomplish. I wanted to leverage my Yubikey using U2F 2FA for my logins on Fedora 30 KDE and was unable to find a concise source for directions to accomplish this.

Thanks!
