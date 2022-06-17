# restic-systemd-units

A flexible systemd unit template for using Restic to backup your data to multiple repositories. It was designed to be lightweight and customizable with systemd overrides. If you want to have notifications on backup failures you will need to setup [Blazon](https://github.com/quickvm/blazon) (a simple systemd unit template for setting up `OnFailure=` notifications). See the Notifications section in this README for more info.

## Install the unit

**Note:** Make sure you have [restic](https://restic.readthedocs.io/en/latest/020_installation.html) installed!

Automatically:
1. Run `sudo mkdir /etc/restic`
1. Run `sudo ./install`

Manually:
1. Run `sudo mkdir /etc/restic`
1. `sudo cp mv units/* /etc/systemd/system/`
1. run `systemctl daemon-reload`


## Create backup configuration files

Below are examples for setting up Local, AWS, and Google Cloud backups. Be sure to create a `/etc/restic/exclude.conf` file and customize it to exclude files you do not want to backup. See the `examples/` directory for inspiration.

### Local

Create `/etc/restic/local.conf` and configure your local backups as you see fit. Since your configuration file might contain your `RESTIC_PASSWORD` or other sensitive information you will want to `chmod 0600 /etc/restic/local.conf` this file. Also make sure the path to `RESTIC_REPOSITORY` exits and your user has permissions to write to it.

```bash
RESTIC_BACKUP_EXCLUDES="--exclude-file /etc/restic/exclude.conf --exclude-if-present .restic_exclude"
RESTIC_BACKUP_PATHS="/home/bingo"
RESTIC_RETENTION_POLICY="--keep-within-hourly 24h --keep-within-daily 7d --keep-within-weekly 1m --keep-within-monthly 1y --keep-within-yearly 7y"
RESTIC_PASSWORD=mysupersecretbackuppassword
RESTIC_REPOSITORY=/mnt/storage/backups/bluey
```

### AWS

Setup an AWS S3 bucket with an IAM user and policy that has access to the bucket. See the [Restic AWS documentation](https://restic.readthedocs.io/en/latest/080_examples.html) for more information. Then create `/etc/restic/aws.conf` and add the below configuration:

```bash
AWS_ACCESS_KEY_ID="AAAAAAAAAAAAAAAAAAAA"
AWS_DEFAULT_REGION="us-east-2"
AWS_SECRET_ACCESS_KEY="BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB"
RESTIC_BACKUP_EXCLUDES="--exclude-file /etc/restic/exclude.conf --exclude-if-present .restic_exclude"
RESTIC_BACKUP_PATHS="/home/bingo"
RESTIC_PASSWORD="anothersuperscretpassword"
RESTIC_REPOSITORY="s3:https://s3.amazonaws.com/mycool-restic-backups-s3-bucket/bluey"
RESTIC_RETENTION_POLICY="--keep-within-hourly 24h --keep-within-daily 7d --keep-within-weekly 1m --keep-within-monthly 1y --keep-within-yearly 7y"
```

Since your `/etc/restic/aws.conf` configuration file might contain your `RESTIC_PASSWORD` or other sensitive information you will want to `chmod 0600 /etc/restic/aws.conf` this file.

### GCP

Create a [Google Cloud Storage bucket](https://cloud.google.com/storage/docs/creating-buckets) and then create a new Service Account that has access to that bucket. See the [Restic GCP documentation](https://restic.readthedocs.io/en/latest/030_preparing_a_new_repo.html#google-cloud-storage) for more information. Then create `/etc/restic/gcp.conf` and add the below configuration:

```bash
GOOGLE_PROJECT_ID=123123123123
GOOGLE_APPLICATION_CREDENTIALS=/etc/restic/gcs-secret-restic-key.json
RESTIC_BACKUP_EXCLUDES="--exclude-file /etc/restic/exclude.conf --exclude-if-present .restic_exclude"
RESTIC_BACKUP_PATHS="/home/bingo"
RESTIC_PASSWORD="themostsecretpasswordever"
RESTIC_REPOSITORY="gs:mycool-restic-backups-gcs-bucket:/bluey"
RESTIC_RETENTION_POLICY="--keep-within-hourly 24h --keep-within-daily 7d --keep-within-weekly 1m --keep-within-monthly 1y --keep-within-yearly 7y"
```

Since your `/etc/restic/gcp.conf` configuration file and your `/etc/restic/gs-secret-restic-key.json` Service Account JSON file might contain your `RESTIC_PASSWORD` or other sensitive information you will want to `chmod 0600 /etc/restic/aws.conf /etc/restic/gs-secret-restic-key.json` these two files.

## Initialize the repository

### Local

```
sudo ./bin/restic-wrapper /etc/restic/local.conf init
```

### AWS

```
sudo ./bin/restic-wrapper /etc/restic/aws.conf init
```

### GCP

```
sudo ./bin/restic-wrapper /etc/restic/gcp.conf init
```

## Enable and start the backup timers

### Local

```
sudo systemctl enable --now restic@local.timer
```

### AWS

```
sudo systemctl enable --now restic@aws.timer
```

### GCP

```
sudo systemctl enable --now restic@gcp.timer
```

## Interacting with your repositories

You can use `./bin/restic-wrapper /path/to/your.config` to run manual commands on your repositories. Be sure you have permissions to read the config file as your user.

**Run a check**

```bash
sudo ./bin/restic-wrapper /etc/restic/local.conf check
using temporary cache in /tmp/restic-check-cache-2107648289
repository 63c2e3d8 opened successfully, password is correct
created new cache in /tmp/restic-check-cache-2107648289
create exclusive lock for repository
load indexes
check all packs
check snapshots, trees and blobs
[0:08] 100.00%  14 / 14 snapshots
no errors were found
```

**Run a dry run forget**
```bash
sudo ./bin/restic-wrapper /etc/restic/local.conf forget --dry-run --verbose --tag local --group-by "paths,tags" --keep-within-hourly 24h --keep-within-daily 7d --keep-within-weekly 1m --keep-within-monthly 1y --keep-within-yearly 7y
repository 63c2e3d8 opened successfully, password is correct
Applying Policy: keep hourly snapshots within 24h, daily snapshots within 7d, weekly snapshots within 1m, monthly snapshots within 1y, yearly snapshots within 7y
snapshots for (tags [local], paths [/home/bingo]):
keep 14 snapshots:
ID        Time                 Host        Tags        Reasons            Paths
-------------------------------------------------------------------------------------
e2f01b0d  2022-06-16 00:37:45  bluey     local       hourly within 24h  /home/bingo
e0af1b23  2022-06-16 01:28:03  bluey     local       hourly within 24h  /home/bingo
f4fa0b64  2022-06-16 02:37:18  bluey     local       hourly within 24h  /home/bingo
8e05e9a9  2022-06-16 03:50:34  bluey     local       hourly within 24h  /home/bingo
d2b2f067  2022-06-16 04:50:52  bluey     local       hourly within 24h  /home/bingo
65ff1396  2022-06-16 06:00:22  bluey     local       hourly within 24h  /home/bingo
c205a7d1  2022-06-16 07:07:37  bluey     local       hourly within 24h  /home/bingo
c5630498  2022-06-16 08:17:23  bluey     local       hourly within 24h  /home/bingo
62a05617  2022-06-16 09:23:10  bluey     local       hourly within 24h  /home/bingo
e155b037  2022-06-16 10:37:52  bluey     local       hourly within 24h  /home/bingo
0bd39494  2022-06-16 11:46:23  bluey     local       hourly within 24h  /home/bingo
6bfccf17  2022-06-16 13:00:17  bluey     local       hourly within 24h  /home/bingo
73de7644  2022-06-16 14:07:52  bluey     local       hourly within 24h  /home/bingo
4421dcd0  2022-06-16 15:03:47  bluey     local       hourly within 24h  /home/bingo
                                                       daily within 7d
                                                       weekly within 1m
                                                       monthly within 1y
                                                       yearly within 7y
-------------------------------------------------------------------------------------
14 snapshots
```

## Enable backup failure notifications with Blazon 📣📣

We will be setting up local desktop and Slack notifications in this example. You need to install the Blazon [systemd units](https://github.com/quickvm/blazon#install-the-systemd-unit-template) and you will need to create a Slack Incoming Webhook on your Slack org that posts messages to a channel. See the [repository](https://github.com/quickvm/blazon) for more information on using Blazon for configuring this systemd template unit.

**Configure Blazon for local and AWS backups**

Create systemd overrides for each `blazon@restic@` and service:

```
systemctl edit blazon@restic@local.service
systemctl edit blazon@restic@aws.service
```

and add the following to each:

```
[Service]
Environment=BLAZON_NOTIFY_DESKTOP=true
Environment=BLAZON_NOTIFY_SLACK=true
Environment=BLAZON_SLACK_WEBHOOK_URL="https://hooks.slack.com/services/AAAAAAAAAAA/AAAAAAAAAAA/HHHHHHHHHHHHHHHHHHHHHHHHH"
```

Create overrides for each `restic@.service`:

```
systemctl edit restic@local.service
systemctl edit restic@aws.service
```

and add the following to each:

```
[Unit]
OnFailure=blazon@%N.service
```

Now you should get desktop notifications and a Slack message if any of your backups units fail!

# License

MIT License

Copyright (c) 2022 QuickVM

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
