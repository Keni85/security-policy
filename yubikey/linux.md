# YubiKey Support for GNU/Linux

This is the GNU/Linux specific documentation for [YubiKey](/yubikey).

Please help make this page more useful by adding links you found useful (describe exactly how they are useful) and and specific steps you used to install, configure, and test your YubiKey.

Table of Contents
=================

* [Install packages](#install-packages)
  * [Arch](#arch)
  * [Fedora](#fedora)
  * [Ubuntu, Xubuntu](#ubuntu-xubuntu)
* [Personalize your YubiKey](#personalize-your-yubikey)
  * [Add a challenge-response slot](#add-a-challenge-response-slot)
* [Set up PAM TFA](#set-up-pam-tfa)
* [Screen lock on lid closed (X server)](#screen-lock-on-lid-closed-x-server)
  * [Screen lock with xss-lock](#screen-lock-with-xss-lock)
  * [Screen lock with xautolock](#screen-lock-with-xautolock)
	* [Arch](#arch-1)
* [YubiKey removal lock](#yubikey-removal-lock) (optional)
* [Away detection ideas](#away-detection-ideas) (optional)

## Install packages

### Arch
_See also: https://wiki.archlinux.org/index.php/yubikey_
```
$ pacaur -S perl-net-ldap-server    # this is a prerequisite
$ pacaur -S yubikey-neo-manager-git yubico-pam
```

### Fedora
_See also: https://fedoraproject.org/wiki/Using_Yubikeys_with_Fedora_
```
dnf copr enable jjelen/yubikey-neo-manager 
dnf copr enable spartacus06/yubikey-utils 
dnf install yubikey-neo-manager yubioath-desktop yubikey-personalization-gui
```

### Ubuntu, Xubuntu
_See e.g.: https://developers.yubico.com/yubico-pam/Authentication_Using_Challenge-Response.html_

tbd...

## Personalize your YubiKey
This allows you to use your Yubikey with Google TFA (new fangled U2F), as well as LastPass (which uses the OTP application).
_Note: these checkboxes may already be checked on YubiKey 4 devices._

### Add a challenge-response slot
```
$ neoman
# Enable OTP, U2F, CCID checkboxes if needed, follow instructions to add and remove key.

​$ ykpersonalize -2 -ochal-resp -ochal-hmac -ohmac-lt64 -oserial-api-visible
```

## Set up PAM TFA
PAM is the Pluggable Authentication Module used by GNU/Linux and Mac OS X to manage login authentication.

This will require the Yubikey (Two Factor Authentication) to be inserted to authenticate via PAM (login, sudo or screen unlock). Test this carefully in an alternate console session to ensure you don't lock yourself out!

See [Yubico GitHub](https://github.com/Yubico/yubico-pam/blob/b0e243835e61418bfa760e57c3d313b2e9452e87/doc/Authentication_Using_Challenge-Response.adoc) page for complete documentation.
```
$ ykpamcfg -2 -v
```

Ubuntu autoconfiguration during installation of `libpam-yubico` may already have placed a line like the following in either `/etc/pam.d/common-auth` or `/etc/pam.d/system-auth`. If not using Ubuntu (or the line is not there), edit `/etc/pam.d/system-auth` (will need to `sudo`) and add the following line at the top of the file:
```
auth      required  pam_yubico.so   mode=challenge-response
```

## Screen lock on lid closed (X server)
After a period of inactivity and (for laptops) when you close the lid, the screen must blank (or be replaced with a background image.

### Screen lock with xss-lock
Links:
- https://wiki.archlinux.org/index.php/Power_management#xss-lock
- http://manpages.ubuntu.com/manpages/xenial/man1/xss-lock.1.html

This uses xss-lock (the brains behind the venerable xscreensaver function) and [i3lock](http://i3wm.org/i3lock/) as the screen locker, but you can substitute this with another locker such as [xsecurelock](https://github.com/google/xsecurelock). xss-lock subscribes to the systemd-events `suspend`, `hibernate`, `lock-session`, and `unlock-session` with appropriate actions (run locker and wait for user to unlock or kill locker). xss-lock also reacts to DPMS events and runs or kills the locker in response.

If your system is already set up to sleep after a timeout and suspend when you close the lid, this may be all you need.

Start xss-lock when you start your window manager, passing it the name of the screen locker you want to use. So you could put this in your ~/.xinitrc file:
```
xss-lock -- i3lock -n -c 000000
```

Or if running the i3 windor manager, put this in your ~/.i3/config file:
```
exec --no-startup-id xss-lock -- i3lock -n -c 000000
```

### Screen lock with xautolock
This uses [xsecurelock](https://github.com/google/xsecurelock) (recommended screen lock) together with [xautolock](http://linux.die.net/man/1/xautolock) (simple away command runner tool) to lock the screen after 10 minutes when away from home network. It also suspends after 30 mins, adds a hot corner to block locking (useful if watching a video, for example) and adds a notification (using `dunst` and `notify-send`) before locking. Note that pretty much all of these pieces are optional (you could use `gnome-screensaver` or `xscreensaver` for away detection for instance), but using `xsecurelock` for locking is strongly recommended since other lock screens have had vunerabilities.

Install packages as needed (`dunst` and `libnotify` optional -- you may already have a notification system):

#### Arch
```
$ pacaur -S xsecurelock-git xautolock dunst libnotify
```

Next make sure dunst (if using for notifications) and xautolock (if using) are started on X login.
For example, you can adapt the following to start when your window manager starts, e.g., add to `~/.xinitrc`:
```
dunst &
xautolock -time 10 -corners -000 -locker '/usr/bin/xsecurelock auth_pam_x11 saver_blank' -killtime 30 -killer 'systemctl suspend' -notify 30 -notifier "notify-send -- 'Locking screen in 30 seconds'" &
```

## YubiKey removal lock
For additional security, you may want to immediately lock the screen when the YubiKey is removed.

This locks the laptop immediately when any Yubikey is removed. If you are not using xautolock as your "away detector", replace xautolock with a command to trigger your screen lock with the "away detector" that you do use. This is inspired by https://vtluug.org/wiki/Yubikey

Assuming `~/bin/` is in your `$PATH`, create executable file `~/bin/ykgone`:
```
#!/bin/bash
USER=`whoami`
getuser ()
    {
     export DISPLAY=":0"
     user=$(who | grep "$DISPLAY" | awk '{print $1}' | tail -n1)
     export XAUTHORITY=/home/$USER/.Xauthority
     eval $1=$USER
    }

getuser "$USER"
su $USER -c 'xautolock -locknow' &
```

Next, create (with sudo) a device notification file `/etc/udev/rules.d/90-yubikey.rules`:
```
ACTION=="remove", ATTRS{idVendor}=="1050", RUN+="/home/$USER/bin/ykgone"
```

## Away detection ideas
Exceptions to the "idle timeout lock" can be made if you are on your home network and feel that it is secure. Adapt the below script if you only want to lock your screen when you are away from home.

Assuming `~/bin/` is in your `$PATH`, create executable file `~/bin/out-lock` and replace `xsecurelock` or `i3lock` above with `~/bin/out-lock`:
```
#!/bin/sh
# Not home (you will need to adjust to some reliable/secure test for your home network).
# In this case, an internal NAT addressable file hoe.txt has the given sha256sum value.
if ! curl -s 'http://192.168.1.99/home.txt' | sha256sum | grep 6094dd1d56b9d8638bc0e8e630683787151b81320d81568d97ec8daecb370bca > /dev/null; then
  # Not already locked.
  if ! pidof xsecurelock > /dev/null; then
    # Lock screen.
    /usr/bin/xsecurelock auth_pam_x11 saver_blank
  fi
fi
```