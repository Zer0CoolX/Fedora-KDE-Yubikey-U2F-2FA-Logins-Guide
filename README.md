**UNTESTED for Fedora 31** - Try at your own risk in Fedora 31 and later. I do not plan to maintain this repo further at this time.

# Fedora KDE Yubikey 2FA U2F Setup Guide
A guide to setup a Yubikey physical security key as a second factor authentication (2FA) using U2F within Fedora KDE and SDDM. Fedora Linux using the KDE Desktop Environment (DE) can use a Yubikey as 2FA for the following login types:

* login screen (both initial and lock screen)
* sudo
* su
* policykit pop-up windows (authentication when opening an app in the GUI)

Using these directions will require a manually entered password AND the Yubikey to not only be plugged in to a USB port, but the button pressed when prompted for credentials. This means that even if someone gets your password they would be unable to login without your physical Yubikey.

### What This Process Will NOT Do
These steps will NOT require the Yubikey for SSH or any other form of remote access. It only applies 2FA to the above listed logins. It also does not replace using a manually entered password to login. It can potentially be modified to do so, but is beyond the scope of this guide.

### Warnings!!
* Use these steps at your own risk! I, nor anyone else, are responsible if you execute any of the steps in this guide and loose access to your system.
* Test this before using seriously. Setup a VM or a spare machine and ensure you have the steps down and confirm it works completely, including after a reboot.
* I highly recommend having at least 1 spare key if not more (IE: 2 or more total keys) in the event you lose or break a key.
* Do NOT follow this guide unless you understand the implications of what it does and that it can prevent access to a system if used incorrectly

Be aware that if you do not also encrypt your drive that it will be possible to circumvent the need for the Yubikey/2FA. I would also recommend password protecting your UEFI/BIOS along with encrypting your drive, but that is up to you.

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
   5. PolicyKit KDE Agent 2FA

Of the "Implementation" options, you may use all of them or only the ones you want. IE: you can setup Yubikey for `KDE initial login` and nothing else or you could setup `sudo`, `su` and `initial login` but not the `lock screen`.

Be aware that if someone gets your password (assuming you are a sudo user) or the root password and 2FA is not enabled for `sudo` and/or `su`, that it will be possible to bypass the need for 2FA elsewhere.

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
At this stage we will create files, key mappings, that the Yubikey authenticates against. There are 2 approaches to how we do this. The one we will be using is to have a single file for each user so they may set and change thier own keys when needed and so that no single person has access to all the keys in a single place. It also makes configuring the PAM files later a bit easier. The second method, which I will not cover, uses a single mapping file `/etc/u2f_mappings`.

### Creating Key Mapping Files
We will need to create 1 file for each user and a file for the root user.

**Each User** (signed in as that user):
1. Be in the users home folder, if need be use `cd /home/user_name` and replace `user_name` with your actual user's name.
2. Execute `mkdir ~/.config/Yubico` to create the directory
3. Run `pamu2fcfg > ~/.config/Yubico/u2f_keys`, when the Yubikey blinks press the button.
  * For each additional physical Yubikey you want to have as backups run the following `pamu2fcfg -n >> ~/.config/Yubico/u2f_keys`

**For Root**:
1. Switch to the root user using `su`. This will require the root credentials.
2. Change to the root directory via `cd /root`
3. Execute `mkdir -p ~/.config/Yubico` to create the directory (-p creates the parent dir as well if it does not exist)
4. Run `pamu2fcfg > ~/.config/Yubico/u2f_keys`, when the Yubikey blinks press the button.
  * For each additional physical Yubikey you want to have as backups run the following `pamu2fcfg -n >> ~/.config/Yubico/u2f_keys`

Upon completion each user will have the file `~/.config/Yubico/u2f_keys` in thier home folder (`/root` for root user) which will contain the keys for each physical Yubikey they set.

## Testing Keys/Mapping
To ensure the steps taken for key setup/mapping has worked, enabling 2FA on sudo is the fastest and safest method for testing.

To test, follow the steps given in the Sudo 2FA section. Make sure to keep the terminal window you make the change in open, if you close this and are unable to login as sudo you may en up being locked out of sudo for good.

Open a second terminal window and execute `sudo echo test`. You should be prompted to enter your password and touch your Yubikey button before seeing "test" in stdout. If not recheck the steps from Sudo 2FA in the first terminal window, close and reopen a second terminal window and try again.

If it works and you do not want to enforce 2FA on sudo long term, simply remove/comment out the line added in the directions to `/etc/pam.d/sudo`.

## Implement 2FA
This is where we will enable using the Yubikey as 2FA via U2F for each login type we desire. All the files that control this are located in `/etc/pam.d` and changes will apply to all users.

### Sudo 2FA
To enforce 2FA using U2F with your Yubikey for `sudo`, do the following:

1. `sudo vi /etc/pam.d/sudo`
2. Change the line `auth include system-auth` to `auth substack system-auth`.
3. Below the line `auth substack system-auth` insert the following: `auth required pam_u2f.so cue`
4. To save and exit `:wq!`

Note that `cue` on the end of the added line displays a prompt in the terminal when it's time to press the button on your Yubikey. The message reads "Please touch the device".

When attempting to use `sudo` in the terminal, you will now be prompted for your password and then be prompted to touch the button on the Yubikey. If the key is not connected or you do not press the button in a reasonable time frame authentication will fail and not allow executing the command as sudo.

### SU 2FA
To enforce 2FA using U2F with your Yubikey for `su`, do the following:

1. `sudo vi /etc/pam.d/su`
2. Below the line `auth substack system-auth` insert the following: `auth required pam_u2f.so cue`
3. To save and exit `:wq!`

Note that `cue` on the end of the added line displays a prompt in the terminal when it's time to press the button on your Yubikey. The message reads "Please touch the device".

For `su` to work with the Yubikey, whichever user you attempt to su into needs to have a key assigned to it. This means that the user needs to have a `u2f_keys` file you are trying to su to.

When attempting to use `su` in the terminal, you will now be prompted for your password and then be prompted to touch the button on the Yubikey. If the key is not connected or you do not press the button in a reasonable time frame authentication will fail and not allow executing the su command.

### KDE Initial Login 2FA
To enforce 2FA using U2F with your Yubikey for the initial login to KDE via SDDM. This will not require the Yubikey to login when the screen has been locked.

1. `sudo vi /etc/pam.d/sddm`
2. Below the line `auth substack password-auth` insert the following: `auth required pam_u2f.so`
3. To save and exit `:wq!`

Note that `cue` does not work for the login screen like it does for sudo/su.

When initially logging into the GUI, you will be prompted for your password. There will be no prompt to press the button on the Yubikey, but the button will start blinking after typing the password and hitting enter or clicking the login button on screen. Once the password is entered/submitted and the button blinks, press it to finish logging in. If the key is not connected or you do not press the button in a reasonable time frame authentication will fail and not login.

To be clear, the only indication of when and that the Yubikey is required and needs the button pressed is after entering and submitting the password. It is only in the form of the button on the Yubikey blinking at that time. If you fail to press it or do not have the key inserted there maybe a message about failing to login.

### KDE Lock Screen 2FA
To enforce 2FA using U2F with your Yubikey for the lock screen of KDE/SDDM. This will require the Yubikey to login any time the screen has been locked.

1. `sudo vi /etc/pam.d/kde`
2. Below the line `auth substack system-auth` insert the following: `auth required pam_u2f.so`
3. To save and exit `:wq!`

Note that `cue` does not work for the login screen like it does for sudo/su.

When logging in from the lock screen, you will be prompted for your password. There will be no prompt to press the button on the Yubikey, but the button will start blinking after typing the password and hitting enter or clicking the login button on screen. Once the password is entered/submitted and the button blinks, press it to finish logging in. If the key is not connected or you do not press the button in a reasonable time frame authentication will fail and not login.

To be clear, the only indication of when and that the Yubikey is required and needs the button pressed is after entering and submitting the password. It is only in the form of the button on the Yubikey blinking at that time. If you fail to press it or do not have the key inserted there will maybe a message about failing to login.

### PolicyKit KDE Agent 2FA
To enforce 2FA using U2F with your Yubikey for authentication windows that pop-up when using specific GUI application in KDE that require credentials to run (ex: Firewall Configuration GUI program).

1. `sudo vi /etc/pam.d/polkit-1`
2. Above the line `auth include system-auth` insert the following: `auth required pam_u2f.so`
3. To save and exit `:wq!`

Note that `cue` does not work for the login screen like it does for sudo/su.

When running a GUI program that requires authentication, you will be prompted for your password in a pop-up window. There will be no prompt to press the button on the Yubikey, but the button will start blinking after typing the password and hitting enter or clicking the ok button in the window. Once the password is entered/submitted and the button blinks, press it to finish logging in. If the key is not connected or you do not press the button in a reasonable time frame authentication will fail and not proceed.

To be clear, the only indication of when and that the Yubikey is required and needs the button pressed is after entering and submitting the password. It is only in the form of the button on the Yubikey blinking at that time. If you fail to press it or do not have the key inserted there will maybe a message about failing to login.

## Resources
Here are some of the resources that helped me piece this together. Many are not specific to Fedora or KDE but gave me a point of reference. Some are also using different methods/protocols while others are dated or incomplete. The Fedora specific Yubico support article is pretty bad.

I created this guide as a resource for myself and others as I could not find clear, up-to-date directions specific to Fedora and KDE/SDDM to setup my Yubikeys for 2FA using U2F for logins.

* My reddit post in which the user abitstick was very helpful in working this out. Thanks! [How Do I Setup Yubikey with Fedora 30 KDE/SDDM (and maybe LUKS v2)?](https://www.reddit.com/r/linuxquestions/comments/bwcn78/how_do_i_setup_yubikey_with_fedora_30_kdesddm_and/?utm_source=share&utm_medium=web2x)
* Another reddit post of mine in which aoeudhtns was very helpful. Thanks! [Yubikey, KDE U2F, Will This Work On Other Distros?](https://www.reddit.com/r/linuxquestions/comments/cdzzfs/yubikey_kde_u2f_will_this_work_on_other_distros/?utm_source=share&utm_medium=web2x). This made me aware of `authselect`, more on this below.
* The Yubico support article for RHEL using challenge/response vs U2F. [Red Hat Linux Login Guide - Challenge Response](https://support.yubico.com/support/solutions/articles/15000011445-red-hat-linux-login-guide-challenge-response)
* The Yubico support article for Ubuntu using U2F. [Ubuntu Linux Login Guide - U2F](https://support.yubico.com/support/solutions/articles/15000011356-ubuntu-linux-login-guide-u2f)
* A blog post I found specific to Mint and U2F. Helped with mapping and has clear examples. [TREZOR/U2F Login into Your Linux Mint](https://blog.trezor.io/trezor-u2f-login-into-your-linux-mint-bd3684d4a8ba)

Regarding `authselect`, which is included with Fedora, CentOS and RHEL, this tool appears to be aimed more at Gnome and less so other DE's. As such, it will work for KDE with a little effort but I could not find a way to have it provide a cue for sudo/su and still work. I made a reddit post with info [Help, authselect, U2F, Yubikey](https://www.reddit.com/r/Fedora/comments/cgfc4h/help_authselect_u2f_yubikey/?utm_source=share&utm_medium=web2x). The gist is `authselect` tries to use `system-auth` and `password-auth` files to spill down the U2F settings to all files that look at them which does not allow using cue only in sudo and su. With cue in sddm/kde (or any other GUI authentication), login fails on the login screen and lock screen as SDDM does not support queing for the 2FA. My manual approach applies cue just to sudo/su and also allows the 2FA prompt to occur **after** entering the password, not before it. Hence why I have a guide with these manual steps vs recommending using authselect.

For those interested, I have another guide [Fedora KDE Minimal Install](https://github.com/Zer0CoolX/Fedora-KDE-Minimal-Install-Guide) to install Fedora Linux with the KDE Plasma Desktop Environment (DE) from a minimal Fedora installation. I do this to limit the bloat and have more control over what gets installed on my system versus using the official KDE spin.

## Final Thoughts
This is a work in progress. I intend to update it as I gain a better understanding of how it all works. Should you find a problem or have a suggestion for how to make the guide better please submit an issue, providing as much detail as you can.

It is my hope that this guide will help others accomplish the same goals I have set out to accomplish. I wanted to leverage my Yubikey using U2F 2FA for my logins on Fedora 30 KDE and was unable to find a concise source for directions to accomplish this.

Thanks!
