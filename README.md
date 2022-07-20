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

Follow the instructions 1. - 6. from https://jellyfin.org/docs/general/administration/installing.html#debian:
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

Go to your webbrowser and open `<<raspberryIP>>` or `<<hostname>>`. Follow the instructions to setup OMV.

Change admin password and disable SSH access for root!

## Secure SSH and Jellyfin with fail2ban

In case youu want to access your Raspberry Pi or Jellyfin from the internet, you should setup fail2ban to block IP addresses who try to Brute force into your Raspberry.

TODO

## Setup ddclient
