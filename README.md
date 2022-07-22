# RaspberryPi-Movie-Server

How to setup a Raspberry Pi with Jellyfin, OpenMediaVault and ddclient and secure it with fail2ban

## Setup

Setup the Raspberry Pi with a fresh copy of the latest headless OS (Debian) and enable SSH. Connect it to your local Network via Ethernet if possible.
IMPORTANT: For OpenMediaVault to work, you have to install a distribution **without** a desktop environment!

Update your Raspberry Pi’s operating system by using the following two commands:
```
sudo apt update
sudo apt full-upgrade
```

Make sure to not plug in more than one harddisk without own power supply.

## Install Jellyfin

Follow the instructions 1. - 6. from https://jellyfin.org/docs/general/administration/installing.html#debian.

Summed up commands are:
```
sudo apt install extrepo
sudo extrepo enable jellyfin
sudo apt update
sudo apt install jellyfin
```

Managing the service:
```
sudo service jellyfin status
sudo systemctl restart jellyfin
sudo /etc/init.d/jellyfin stop
```

Go to your webbrowser and open `<<raspberryIP>>:8096` or `<<hostname>>:8096`. Follow the instructions to setup Jellyfin.

## Install OpenMediaVault

In order to be able to manage the moviefiles on the harddisks which are connected via USB to the Raspberry Pi, we have to make them available in the Network (SMB). The easiest way to do it is with OpenMediaVault (OMV).

To install OMV on a Raspberry Pi, execute this command from https://github.com/OpenMediaVault-Plugin-Developers/installScript:
```
sudo wget -O - https://github.com/OpenMediaVault-Plugin-Developers/installScript/raw/master/install | sudo bash
```

This may take a while. After it finished, reboot the system with
```
sudo reboot
```

Go to your webbrowser and open `<<raspberryIP>>` or `<<hostname>>`. Login with the username `admin` and password `openmediavault`.

Change admin password, disable SSH access for root and enable SSL!

Don't forget to enable SMB ;-)

### Problems with exFAT hard disks

If you want to share a folder on a exFAT formatted hard disk, you run into a problem, that as soon as you reboot the Raspberry Pi, you won't have write permissions anymore. To solve this problem, follow this steps:

1. Install exfat-fuse and reboot
```
sudo apt-get install exfat-fuse
sudo apt-get install -f
sudo reboot
```

2. Change the fileformat in the fstab entry from openmediavault from `exfat` to `exfat-fuse`:
```
sudo nano /etc/fstab
```

The fstab file could look like this:
```
# >>> [openmediavault]
/dev/disk/by-uuid/0E33-DD1D             /srv/dev-disk-by-uuid-0E33-DD1D exfat-fuse      defaults,nofail 0 2
# <<< [openmediavault]
```

Note that writing on that hard disk needs a lot of CPU usage. You might consider to have a NTFS (yes, NTFS works fine with Jellyfin and OMV) or ext4 formatted hard disk.

## Secure SSH and Jellyfin with fail2ban

In case youu want to access your Raspberry Pi or Jellyfin from the internet, you should setup fail2ban to block IP addresses who try to Brute force into your Raspberry.

### SSH

There is a nice tutorial here: https://pimylifeup.com/raspberry-pi-fail2ban/

Summed up commands are:
```
sudo apt update
sudo apt upgrade
sudo apt install fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Open jail.local with `sudo nano /etc/fail2ban/jail.local` and find:
```
[sshd]

port    = ssh
logpath = %(sshd_log)s
backend = %(sshd_backend)s
```

Add following lines below `[sshd]` (copy and replace the whole part):
```
[sshd]
enabled = true
filter = sshd
port = ssh
banaction = iptables-multiport
bantime = -1
maxretry = 3
logpath = %(sshd_log)s
backend = %(sshd_backend)s
```

Save the file and restart fail2ban:
```
sudo service fail2ban restart
```

Try to get yourself banned to check if it works! Also, if your Router allows, change public SSH port to something else then port 22. You could for example map the port 3303 to 22. This prevents A LOT of attacks.

### Jellyfin

Follow this official tutorial: https://jellyfin.org/docs/general/networking/fail2ban.html

Summed up commands are:
```
sudo nano /etc/fail2ban/jail.d/jellyfin.local
```

Add to this new empty file:
```
[jellyfin]

backend = auto
enabled = true
port = 8096,8920
protocol = tcp
filter = jellyfin
maxretry = 3
bantime = -1
findtime = 43200
logpath = /var/log/jellyfin/jellyfin*.log
```

Save and exit nano.

```
sudo nano /etc/fail2ban/filter.d/jellyfin.conf
```

Add to this new empty file:
```
[Definition]
failregex = ^.*Authentication request for ".*" has been denied \(IP: "<ADDR>"\)\.
```

save and exit nano. Then reload fail2ban:
```
sudo systemctl restart fail2ban
```

Try to get yourself banned to check if it works!

### Userful commands

Here are some useful commands to check banned IPs, unban IPs etc:
```
sudo fail2ban-client status sshd
sudo fail2ban-client status jellyfin
sudo fail2ban-client set sshd unbanip 194.230.155.108
sudo fail2ban-client set jellyfin unbanip 194.230.155.108
sudo zgrep 'Ban' /var/log/fail2ban.log*
sudo iptables -L INPUT -v -n | less
sudo zgrep 'Accepted'  /var/log/auth.log
sudo zgrep 'Failed'  /var/log/auth.log
```

## Setup ddclient

Create an account at a dynamic DNS service provider (for example noip.com)

Install ddclient:
```
sudo apt-get install ddclient
```

Enter your dynamic DNS service provider information into the installation prompt.

Open the config file with `sudo nano /etc/ddclient.conf` and delete the backslashes `\`. No idea why they are there, but ddclient doesn't work until they're deleted. Also, add `daemon=600` on top. The file could look like this:
```
# Configuration file for ddclient generated by debconf
#
# /etc/ddclient.conf

daemon=600
protocol=noip
use=web
web=checkip.dyndns.org
login=username
password='password'
myname.ddns.net
```

Test ddclient with:
```
sudo ddclient -daemon=0 -debug -verbose -noquiet
```

Restart the client with:
```
sudo /etc/init.d/ddclient restart
```

To check update logs:
```
zgrep 'ddclient' /var/log/syslog
```

## SSL for Jellyfin

TODO

## Troubleshooting

### Space usage resource limit
Check for space usage warning:
```
zgrep 'space usage' /var/log/syslog
```
you probably see something like this:
```
Jul 22 11:46:40 <hostname> monit[1048]: 'filesystem_srv_dev-disk-by-uuid-xxx' space usage 95.5% matches resource limit [space usage > 95.0%]
```

If a hard disk's space usage is above the set threshold (standard 80% in OMV), it can cause some write permission errors (even on other connected hard disks with enough free space!). To solve this, check if something is red with:
```
sudo monit status
```

Add to `/etc/default/openmediavault` following line:
```
OMV_MONIT_SERVICE_FILESYSTEM_SPACEUSAGE=<percentage>
```

Execute following:
```
sudo monit reload
sudo reboot
```

` zgrep 'space usage' /var/log/syslog` will still show the warnings, but it seems to solve the write permission errors.
