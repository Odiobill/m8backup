# m8backup
Dependency free backup server with data de-duplication using rsync and ssh

## About
*m8backup* is a very quick solution for deploying a central backup server which
offers advanced features like *data de-duplication* out of the box.
Its main purpose is to run backups from the server to many remote hosts over
SSH, so that the backup data will always be encrypted in transit.

## Usual backup procedure
m8backup is a pretty simple wrapper around rsync that should do the following:

* It's invoked by hand or, most probably, with a cron entry.
* You provide a target directory to it as a command line parameter (I.E.: m8backup */srv/backupSet1*).
* m8backup will look if a configuration file is found that "backup set" (I.E.: */srv/backupSet1.m8backup*). If found, it will override its default config settings for that specific backup set.
* The target directory will contain a subdirectory for each server that we need to backup (I.E.: */srv/backupSet1/host1.example.com*).
* m8backup will look if a configuration file is found that server (I.E.: */srv/backupSet1/host1.example.com/server.conf*).
* If found, it will override its default config settings for that specific host.
* m8backup will use *rsync*, that will open an ssh connection to the target host, using key-based authentication. The public key related to the root user of the backup server must be imported into the */root/.ssh/authorized_keys* file.

Remote hosts needs to allow connection as *root* from the SSH key of the central
server. To avoid exposing your data to unwanted audience, you should restrict
*root login* on any server that you want to backup.

## Usage
	Usage:
		./m8backup -h|--help
		./m8backup -o|--only <path> <target>
		./m8backup <path>

## Server deployement guide
Let's see how easy is to start taking your backups.

### sshd configuration (client side)
Edit the SSH server configuration file, usually */etc/ssh/sshd_config*, and
configure it for accepting *root connections* only for a limited amount of tasks:

	PermitRootLogin forced-commands-only

### Adding the right ssh key (client side)
Edit root's authorised-keys file */root/.ssh/authorized*keys*, adding the ssh key
of the backup server. For security measures, it should not allow regular SSH
sessions, and should be trimmed out to only allow the backup operation. Example:

	from="<server ip address>",command="rsync --server --sender -ulogDtpre.iLsfx --numeric-ids . /",no-agent-forwarding,no-port-forwarding,no-user-rc,no-X11-forwarding,no-pty ssh-rsa <root's ssh key> backup@example.com

### Mount your storage volume
You can backup your data to any file-system which supports *hard links*. Ensure
that you have enough disk-space in the destination directory of your choice.
In this guide, we'll use */srv/backupSet1*, so that you can add another, distincted
backup set for other, different hosts, for example using */srv/backupSet2* or
*/srv/what-you-like-more*.

### Adjusting m8backup's default parameters
*m8backups* will operate according to a few values, identified by the following variables:

#### SNAPSHOTS
How many older snapshots to keep. Default: 30.

#### SNAPSNAME
Basename of snapshot directories. Default: "snapshot".

#### EXCLUDING
A file that contains any path to exclude. Example: "/srv/backupSet1.exclude-list". 
If not provided, m8backup will exclude the following list:

* /dev/*
* /proc/*
* /run/*
* /sys/*
* /tmp/*
* /var/lock/*
* /var/run/*

#### RSYNCPATH
Path to the rsync executable. Default: "/usr/bin/rsync".

#### RSYNCOPTS
Command line arguments for rsync. Default: "-uae ssh --delete --numeric-ids".

#### OVERRIDER
Name of the file in the target directory used to override any of above settings for that entry. Default: "server.conf".

#### WSTARTCMD
Path to an external command (usually a shell script) that will be executed if *m8backup* is running on Monday.
The WSTARTCMD command will receive the path of the latest snapshot as single parameter.

#### MSTARTCMD
Path to an external command (usually a shell script) that will be executed if *m8backup* is running on the first day of the month.
The MSTARTCMD command will receive the path of the latest snapshot as single parameter.

#### YSTARTCMD
Path to an external command (usually a shell script) that will be executed if *m8backup* is running on the first day of the year.
The YSTARTCMD command will receive the path of the latest snapshot as single parameter.

### Create a generic options file for your first backup-set
We want to change a few parameters for our backup set: with the exception of a specific host,
we want to keep only 7 in-line snapshots of our servers.  The other options are fine with us
so we don't need to change them.

Let's create **/srv/backupSet1.m8backup** with the following content:

	SNAPSHOTS=7

### Create a directory for each server that we want to backup
Assuming that we want to backup three different servers, we just have to create a
directory for each of them:

	# mkdir /srv/backupSet1/host1.example.com
	# mkdir /srv/backupSet1/host2.example.com
	# mkdir /srv/backupSet1/host3.example.com

We know that **host3.example.com** is more critical than the others, so we want to keep
30 snapshots instead of the usual 7 for the other hosts. Just create an *override file*:

	# mkdir /srv/backupSet1/host3.example.com/server.conf

With the following content:

	SNAPSHOTS=30

### Taking snapshots of local content
We also want to take snapshot of the */home* directory of the backup server, for no
other reason than to learn how to do that.

	# mkdir /srv/backupSet1/localPath1

The **override file** */srv/backupSet1/localPath1/server.conf* should contain the following:

	LOCALPATH="/home"


### Execute the first backup run
Everything is ready, so we can just execute m8backup:

	# ./m8backup /srv/backupSet1

### Adding a new target to our backup-set
To add a new target, we just need to create its directory:

	# mkdir /srv/backupSet1/host4.example.com
	
If we want to take a first dump of this new host only, we can use the *--only* option:

	# ./m8backup --only /srv/backupSet1 host4.example.com

### Exclude a host from the backup list
If you want to temporally avoid to backup one of your hosts, just edit its *override file*
and add the following parameter:

	EXECUTEIT=0

