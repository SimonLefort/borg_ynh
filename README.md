<!--
N.B.: This README was automatically generated by https://github.com/YunoHost/apps/tree/master/tools/README-generator
It shall NOT be edited by hand.
-->

# Borg Backup for YunoHost

[![Integration level](https://dash.yunohost.org/integration/borg.svg)](https://dash.yunohost.org/appci/app/borg) ![Working status](https://ci-apps.yunohost.org/ci/badges/borg.status.svg) ![Maintenance status](https://ci-apps.yunohost.org/ci/badges/borg.maintain.svg)
[![Install Borg Backup with YunoHost](https://install-app.yunohost.org/install-with-yunohost.svg)](https://install-app.yunohost.org/?app=borg)

*[Lire ce readme en français.](./README_fr.md)*

> *This package allows you to install Borg Backup quickly and simply on a YunoHost server.
If you don't have YunoHost, please consult [the guide](https://yunohost.org/#/install) to learn how to install it.*

## Overview

Backup your server on a host server using Borg

**Shipped version:** 1.1.16~ynh29
## Disclaimers / important information

## :warning: NB. : This doc is partially obsolete and should be reworked! :warning:

## How to backup your server with this app?

You want to backup a critical "guest" Server A onto a remote "host" Server B, you need:
* Domain name of server B: `host.serverb`
* Name of the server B SSH user (to be created by `borgserver`) for connection from Server A: `borgservera`
* **Strong passphrase** to encrypt your backups on host Server B. And to **restore your backups**!!
* IDs of YunoHost apps you want to backup
* Regular time schedule for your backups, see below
* Install Borg Backup App (`borg`) on guest Server A
* Install Borg Server App (`borgserver`) on host Server B
* Save the passphrase in another place than your server. Without the passphrase, you won't be able to restore data.

You should received an email after the first backup succeeded.

### Set up Borg Backup App on guest Server A
Firstly, set up the Borg Backup App (`borg`) on the guest Server A you want to backup:

```
$ yunohost app install borg
In which borg repository location do you want to backup your files ?: user@host.serverb:/remote/repository
Provide a strong passphrase to encrypt your backups. No blank space:
Should Borg backup your YunoHost configuration? [yes | no] (default: yes):
Should Borg backup emails and user home directory? [yes | no] (default: yes):
Which apps should Borg backup ? (default: all):
With which regular time schedule should the backups be performed? (see systemd OnCalendar format) (default: Daily):
Do you want admin to receive mail notifications on backups ? [always | errors_only | never]: never
```

#### Syntax to define a backup time schedule
You can schedule regular backups at specific time. Only one regular time schedule is possible for one `borg` instance, see below for workaround. Some examples:
* Monthly :
* Weekly :
* Daily : Daily at midnight
* Hourly : Hourly o Clock
* Sat *-*-1..7 18:00:00 : The first saturday of every month at 18:00
* 4:00 : Every day at 4 AM
* 5,17:00 : Every day at 5 AM and at 5 PM
See here for more info : https://wiki.archlinux.org/index.php/Systemd/Timers#Realtime_timer

#### Information generated by Borg Backup
At the end of the installation, the Borg Backup App (``borg``) displays the SSH public key and the SSH user to give to the person who has access to the host Server B and will set up Borg Server App.
```
You should now install the "Borg Server" app on host.serverb and fill questions like this:
User: servera
Public key: ssh-ed25519 AAAA[...] root@guest.servera
```
This information is also sent by email to the admin of guest Server A.
If you don't find the email and you don't see the message in the log bar you can find the SSH public key with this command:
```
$ cat /root/.ssh/id_borg_ed25519.pub
ssh-ed25519 AAAA[...] root@guest.servera
```

### Set up Borg Server App on host Server B
Secondly, set up the Borg Server App (``borgserver``) on the host Server B that will store your backups:
```
$ yunohost app install borgserver
Indicate the ssh user to create: servera
Indicate the public key given by Borg Backup app (borg) setup: ssh-ed25519 AAAA[...] root@guest.servera
Indicate the storage quota: 5G
```

### Test the Borg Apps setup
At this step your backup should run at the scheduled time. Note that the first backup can take very long, as much data has to be copied through ssh. Following backups are incremental: only newly generated data since last backup will be copied.

If you want to test correct Borg Apps setup before scheduled time, you can start a backup manually on guest Server A:
```
$ systemctl start borg
```

Next you can check presence of your backup repository on host Server B:
```
$ BORG_RSH="ssh -i /root/.ssh/id_borg_ed25519 -oStrictHostKeyChecking=yes " borg list servera@host.serverb:~/backup
```
You will need the passphrase to run ``borg`` commands on the backup repository created on the host Server B.

### History Depth and Pruning
The default setup of this app is to have a **one-year history**, with decreasing frequency of backups kept (for each app and each setup's part): 2 in the last hours, 7 in the last days, 8 in the last weeks and 12 in the last months (so 12 in total). All backup older than 1 year are deleted.

It relies on the [pruning behaviour of borg](https://borgbackup.readthedocs.io/en/stable/usage/prune.html) : `--keep-hourly 2 --keep-daily=7 --keep-weekly=8 --keep-monthly=12` implemented [here](https://github.com/YunoHost-Apps/borg_ynh/blob/testing/conf/backup_method#L55).
You may change this behaviour only via the CLI, by modifiyng the file in`/etc/yunohost/apps/borg(__x)/conf/backup_method`, although it will not persist between upgrades.

If your backup server is full, the app will fail and will **not** add new backup while deleting the old ones.

## Check regularly your backup
If you want to be sure that you can restore your server, you should try to restore archives regularly. But this process takes a long time.

You should at least:
 * Keep your apps up to date (if the apps are too old, they might be difficult to restore to a newer version on YunoHost)
 * Regularly check the presence of `info.json` and `db.sql` or `dump.sql` in the archives of your applications
```
borg list ./::ARCHIVE_NAME | grep info.json
borg list ./::ARCHIVE_NAME | grep db.sql
borg list ./::ARCHIVE_NAME | grep dump.sql
```
 * Be sure to have your passphrase available even if your server is completely broken

## How to restore a complete system

*For infos on restoring process, check [this yunohost forum thread](https://forum.yunohost.org/t/restoring-whole-yunohost-from-borg-backups/12705/3) and [that one](https://forum.yunohost.org/t/how-to-properly-backup-and-restore/12583/3), also [using Borg with sshkeys](https://thisiscasperslife.wordpress.com/2017/11/28/using-borg-backup-across-ssh-with-sshkeys/), the [`borg extract` documentation](https://borgbackup.readthedocs.io/en/stable/usage/extract.html), and this [general tutorial on Borg Backup](https://practical-admin.com/blog/backups-using-borg/).*

In the following explanations:
- the server to backup/restore will be called: `yuno`
- the remote server that receives and store the back will be called: `rem`
- `rem` is accessible at the domain `rem.tld`
- the remote user on `rem` which owns the Borg backups will be called `yurem`
- backup files will be stored in `rem` in the directory: `/home/yurem/backup`


### Overview

The idea here, if you need to restore a whole yunohost system is:

1. Install a new Debian VM
2. Install YunoHost in it the usual way
3. Go through YunoHost postinstall (parameters you will supply are not crucial, as they will be replaced by the restore)
4. Install Borg
5. Setup `rem` to accept ssh connections from `yuno`
6. Use Borg to import backups from `rem` to `yuno`
7. Restore Borg backups with the `yunohost backup restore` command, first config, then data, then each app one at a time
8. Remove the Borg app and restore it

### Make it possible for `yuno` to connect to `rem` with Borg

At this stage, we will assume that `yuno` is a freshly installed YunoHost (based on Buster in my case). You should also have performed the YunoHost postinstall.

If you don't want to restore the whole system, just some apps, you can skip some of the steps below.

#### Install the Borg YunoHost app in `yuno`

The idea here is just to install Borg, not in order to create backups, but only to use Borg commands to import remote backups.

So for example, you can install it doing the following:
```bash
sudo yunohost app install borg -a "server=rem.tld&ssh_user=yurem&conf=0&data=0&apps=hextris&on_calendar=2:30"
```

#### Make sure that `rem` accepts ssh connections from `yuno`

In `yuno` you will need to get the ssh key that borg just created while installing: `sudo cat /root/.ssh/id_borg_ed25519.pub`, copy it to clipboard.

Connect via ssh to `rem`, go to `/home/yurem/.ssh/authorized_keys`, and past the Borg public key you got at previous step.

Now to make sure this worked, you can try to SSH from `yuno` to `rem`.
In `yuno` : `ssh -i /root/.ssh/id_borg_ed25519 yurem@rem.tld` . If you can get into `rem` , without it prompting for a password, then you're good to continue :)

### Restore backups to `yuno`

⚠️ For the commands in the following section to work, you will need to be root in `yuno` (you can become root running `sudo su`).

⚠️ Restoration of backups can take quite a while, you'd better do them in a separate process, so that it doesn't stop if your terminal session gets closed. For this, you can for example use [tmux](https://www.howtogeek.com/671422/how-to-use-tmux-on-linux-and-why-its-better-than-screen/).

In `yuno` now, you should be able to list backups in `rem` with the following command:

```bash
SRV=yurem@rem.tld:/home/yurem/backup
BORG_RSH="ssh -i /root/.ssh/id_borg_ed25519 -oStrictHostKeyChecking=yes " borg list $SRV
```

You can then reimport one to `yuno` with:

```bash
BORG_RSH="ssh -i /root/.ssh/id_borg_ed25519 -oStrictHostKeyChecking=yes " borg export-tar $SRV::auto_BACKUP_NAME /home/yunohost.backup/archives/auto_BACKUP_NAME.tar.gz
```

And then restore the archive in `yuno` with:

```bash
yunohost backup restore auto_BACKUP_NAME --system # for config and data backups
yunohost backup restore auto_BACKUP_NAME --apps # for other backups (=apps)
```

### And Nextcloud? It's super heavy!!

For Nextcloud, the best is probably to reimport the backup without the data. And to import the data manually.

For that, you can do the following (as root):

```bash
SRV=yurem@rem.tld:/home/yurem/backup

# export the app without data
BORG_RSH="ssh -i /root/.ssh/id_borg_ed25519 -oStrictHostKeyChecking=yes " borg export-tar -e apps/nextcloud/backup/home/yunohost.app $SRV::auto_nextcloud_XX_XX_XX_XX:XX /home/yunohost.backup/archives/auto_nextcloud_XX_XX_XX_XX:XX.tar.gz

# extract the data from the backup to the nextcloud folder
cd /home/yunohost.app/nextcloud
BORG_RSH="ssh -i /root/.ssh/id_borg_ed25519 -oStrictHostKeyChecking=yes " borg extract $SRV::auto_nextcloud_XX_XX_XX_XX:XX apps/nextcloud/backup/home/yunohost.app/nextcloud/
mv apps/nextcloud/backup/home/yunohost.app/nextcloud/data data
rm -r apps

# now you can simply restore nextcloud app
yunohost backup restore auto_nextcloud_XX_XX_XX_XX:XX --apps
```

### Restore Borg

Once you've restored the whole system, you will probably want to restore the Borg app as well.

For that, remove the "dummy" Borg you installed to do the restoration, and restore Borg the same ways as for other apps:

```bash
sudo yunohost app remove borg
sudo yunohost backup restore auto_borg_XX_XX_XX_XX:XX --apps
```

## Tips

### Edit the list of YunoHost apps to backup
``yunohost app setting borg apps -v "nextcloud,wordpress"``

### Other usefull borg commands
[Get the storage space used by the backup repository on the host server](https://borgbackup.readthedocs.io/en/stable/usage/info.html)
``borg info /home/servera/backup``

### Backup YunoHost apps with different criticallity levels

If you want to backup your guest server:
* with different YunoHost apps
* at different regular time schedule
* on different host servers

Then you can set up multiple instances of the Borg Apps on same servers.
For instance:
* Borg Backup instance ``borg``: backup nextcloud daily on host Server B
* Borg Backup instance ``borg__2``: backup all other YunoHost apps weekly on host Server C

### Settings location
Settings of apps are in /etc/yunohost/apps/*/settings.yml, so are the settings for borg_ynh as well. Be careful though to not savagely manually edit them if you don't know what you're doing.

## Documentation and resources

* Official app website: <https://borgbackup.readthedocs.io>
* YunoHost documentation for this app: <https://yunohost.org/app_borg>
* Report a bug: <https://github.com/YunoHost-Apps/borg_ynh/issues>

## Developer info

Please send your pull request to the [testing branch](https://github.com/YunoHost-Apps/borg_ynh/tree/testing).

To try the testing branch, please proceed like that.

``` bash
sudo yunohost app install https://github.com/YunoHost-Apps/borg_ynh/tree/testing --debug
or
sudo yunohost app upgrade borg -u https://github.com/YunoHost-Apps/borg_ynh/tree/testing --debug
```

**More info regarding app packaging:** <https://yunohost.org/packaging_apps>
