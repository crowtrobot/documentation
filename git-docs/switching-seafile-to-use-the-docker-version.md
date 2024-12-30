---
description: >-
  The Seafile developers have announced their intent to only release future
  versions as docker container images.  I don't like this plan, but I will need
  to learn to work with it.
---

# Switching Seafile to use the Docker version

## The current system

I will start with a small description of the current setup. My Seafile server is a VM. I never felt the need to run in a container, because it is more thoroughly isolated by being in its own VM, and (for my setup at least) backups of the VM is much easier. There is a small disk for the OS deployed from the template used for almost all servers in the environment. Then 2 disks were added, a small one to hold the mariadb database (/database), and a large one to hold the seafile data (/seafile-data). These disks are encrypted, which makes the backups of all data encrypted without any extra effort needed at backup or restore time.&#x20;

This VM is for Seafile only, the reverse proxy and other servers are in other VMs isolated as much as possible from each other by firewalls, and vlans, etc. There is also an Authelia server providing OAUTH logins for Seafile for users in the appropriate group on ldap, which I mention because some of the problems I had and choices I made in doing this upgrade came from trying to get OAUTH working again with version 12.

## The steps

* Install docker. You have to follow a lot of instructions from docker on how to do this right, but I suggest instead using podman. It is as easy to install as just "sudo apt install podman-docker", or at least it was for me on Debian 12. (see the "[not using root](switching-seafile-to-use-the-docker-version.md#not-using-root)" section below for how and why I switched to using podman for better security and performance).
* Reconfigure mariadb to allow "remote" network connections. Edit /etc/mysql/mariadb.conf.d/50-server.cnf to change the bind-address line to have the server's real IP (not the docker IP, but server's main IP). Then `sudo systemctl restart mariadb.service` to restart the database so that change to the config can take effect. &#x20;
* reconfigure memcached to allow "remote" network connections Edit /etc/memcached.conf to change "-l 127.0.0.1" to instead have the server's IP. Then restart memcached with `sudo systemctl restart memcached.service`
* create firewall rules to allow docker to connect to mariadb and memcached, but block everything from the real network trying to talk to these ports.  I don't have steps to put here since there are so many different firewalls.  One thing to note, docker seems to be famous for messing with firewall configurations (like inserting its own allows above your denies without warning). So be sure to actually test this, don't just assume that because you wrote the rule it is good. One article I read said podman is better behaved, but I would still test to be sure.
* Stop older seafile and seahub version and disable those services in systemd: \
  `systemctl stop seafile.service` \
  `systemctl stop seahub.service` \
  `systemctl disable seafile.service` \
  `systemctl disable seahub.service`
* My databases were using the old default names, so I had to rename them.  See the "[renaming databases](switching-seafile-to-use-the-docker-version.md#renaming-databases)" section below for details on how I did this.  I had some trouble getting everything to work with those old names, so I renamed them from names like ccnet-db to ccnet\_db.&#x20;
* Modify seafile user in the database to be able to log in remotely.  The user was originally only able to log in locally, but it will look to the database like it is connecting over the network when connecting from the container \
  `mariadb` \
  `GRANT ALL PRIVILEGES ON *.* TO 'seafile'@'%' IDENTIFIED BY 'PASSWORD' WITH GRANT OPTION;`
* Give the seafile user permission to the databases when connected remotely:\
  `GRANT ALL PRIVILEGES ON ccnet_db.* to 'seafile'@'%';`\
  `GRANT ALL PRIVILEGES ON seafile_db.* to 'seafile'@'%';` \
  `GRANT ALL PRIVILEGES ON seahub_db.* to 'seafile'@'%';`
* Fix missing tables.  \
  It turns out my database is missing a table (or maybe some tables) from several versions ago, but things were working so I didn't know it. I guess the code treated this as optional until version 12. I needed to make sure I have all the tables. To do this I grabbed the .sql files from /opt/seafile/seafile-server-12.0.6/sql/mysql, (ccnet.sql and seafile.sql), and ran their instructions on the appropriate database. \
  \
  `mariadb ccnet_db < /tmp/ccnet.sql` \
  `mariadb seafile_db < /tmp/seafile.sql`
* Make a directory for the docker configs (.env, seafile-server.yml, seadoc.yml, notification-server.yml). I put these in /seafile-data/docker on my system
* Edit these docker files to remove the caddy stuff\
  As described in the instructions in the manual for not using caddy ( https://manual.seafile.com/12.0/setup/use\_other\_reverse\_proxy/ ). I made some additional changes beyond what was described there. I also removed the db and memcached sections since I already have a working database and memcached and don't need to install another one of either. I also changed the ports that get forwarded out of the seafile-server.yml to look like this: \
  `ports:` \
  &#x20; `- "8000:8000"` \
  &#x20; `- "8082:8082"`\
  \
  This will let us talk directly to Seafile without going through the nginx inside the container. This seemed necessary to get OAUTH working, but I now suspect it wasn't. However, it still made it easier because this means the nginx config I was using on my reverse proxy works without changes on this new Seafile version. \
  It also makes troubleshooting easier, because this way it is easier to use wireshark to see exactly what is going on. I don't like that the container is still running an nginx process I don't need, but at least that process is now out of the way and not causing problems (beyond wasting resources).  Maybe someday a variable can added to the docker config that could let us not start nginx.  \
  \
  I also added the "NON\_ROOT=true" in the .env file as part of my efforts to improve the security of this setup. More details about his in the  "[not using root](switching-seafile-to-use-the-docker-version.md#not-using-root)" section below.
* Move the seahub-data from the old version, to the docker location: \
  `mkdir -p /seafile-data/persistent-data/seafile/seahub-data` \
  `mv /seafile-data/seafile/seahub-data/* /seafile-data/persistent-data/seafile/seahub-data`
* Move the Seafile data from the old version to where docker expects it. \
  I tried with both of these to just set up docker to use those files where they already are, but that didn't work. There were several problems, but mostly they were either giving the docker container access to files it doesn't need, or putting in an extra mount-point or two, which broke uploads by making it impossible to move the temp files into the storage directory.  \
  \
  `mv /seafile-data/data /seafile-data/persistent-data/seafile/seafile-data`\

* Then add the "current\_version" file. \
  This doesn't appear in the documentation (that I could find). I found it while reading through scripts in the container. This is how it will know that you are doing an upgrade and so it needs to run the upgrade/upgrade\_11.0\_12.0.sh file for you. For me this was done with:\
  \
  `echo 11.0.13 > /seafile-data/persistent-data/seafile/seafile-data/current_version`\
  \
  This should be the same directory that contains the "storage" directory. Obviously put the right version number for the version you are upgrading from there.
* Create the logs directory. If you let the container create the logs directory for itself, the permissions will not be set on it correctly, and seafile won't start (at least if you are using the "NON\_ROOT=true" option). This should be fixed in future versions, but is easy enough for us to just work around now. We will make this dir ourselves first: mkdir /seafile-data/persistent-data/seafile/logs
* Set file permissions.  \
  Now we will need the container to have access to the files in the "persistent-data" directory.  The admin guide says to run "`chmod -R a+rwx /seafile-data/persistent-data`" to give everyone access to these files and directories. This means that ever user on this server has full access to read, add and remove Seafile files. I don't really like that idea, but if you're running most of the code on this server as root, then file permissions aren't going to do much to contain any problems anyway.\
  \
  But if you decide to run without root, it makes sense to instead just change these files to be owned by the seafile user inside the container. This is the subid for the user that the docker podman container will run as, plus 7999. I don't know exactly why that number, but I think it is the seafile UID inside the container (8000) + start of subuid range for the user (get that from /etc/subuid, mine is 165536) - 1 (I think because root inside is the user's id outside, and not a subuid, maybe?). \
  \
  Anyway "`chown -R 173535:173535 /seafile-data/persistent-data`" and wait a few minutes for that to finish.&#x20;
* Make sure you have the right storage driver\
  If you are using the podman without root, you might need to tell it to use the "overlay" storage driver instead of the much slower vfs one. In theory overlay is the default, but mine tried to use vfs anyway.  Maybe that's a bug or something in the version currently shipping in Debian 12?  Whatever, easy enough to work around I guess.  Just log into the podman user, and create the \~/.config/containers/storage.conf file, with this content: \
  `[storage]`\
  `#Default Storage Driver, Must be set for proper operation.`\
  `driver = "overlay"`
* Start the container and watch for problems. \
  The admin guide says to run "docker compose up -d" from the directory with the .env and other docker files. The "-d" tells it run in the background, which means you don't get to see what's going on. Since this is the first start and we want to see if any errors appear, I suggest instead running "`docker compose up`", and you can consider using -d when you set it to start automatically on boot.\
  \
  Note: if you switched to using podman instead of docker as I suggest in the "not using root" section below, this command is slightly different. You would run "`docker-compose up -d`" or just "`docker-compose up`". And because I saw a weird timeout once while testing, I decided to turn up the timeout limit, so my command looks like "`COMPOSE_HTTP_TIMEOUT=300 docker-compose up`".\
  \
  You should see a brief log of things starting up, which might include stuff about upgrading database tables and such. It should end like this:\
  \
  `seafile           | Done.` \
  `seafile           |` \
  `seafile           | Starting seahub at port 8000 ...` \
  `seafile           |` \
  `seafile           | Seahub is started` \
  `seafile           |` \
  `seafile           | Done.` \
  `seafile           |`
* Setup to start containers on boot.\
  This is probably the part that's the most different between plain docker, and podman, and podman without root.  I never got to this step with setups other than the podman without root, so if you aren't doing the rootless podman, this step is left up to you to figure out.  The instructions for the rootless podman are in the "[not using root](switching-seafile-to-use-the-docker-version.md#not-using-root)" section. &#x20;



## Renaming databases&#x20;

This isn't hard, but can take some time.  The basic process is to dump the database out into a backup, modify the database names, and then restore from the backup. &#x20;

Dump out the databases, and rename them on their way into this backup file. \
`mysqldump --all-databases | sed 's/ccnet-db/ccnet_db/g' | sed 's/seafile-db/seafile_db/g' | sed 's/seahub-db/seahub_db/g' > all_databases.sql`&#x20;

Then delete the old databases: \
`mariadb` \
`drop database ccnet-db ;` \
`drop database seafile-db ;` \
`drop database seahub-db ;`&#x20;

Re-import the backup which will automatically create the databases with the new names \
`mysql < all_databases.sql`



## Not using root

Once I had it finally tested and working, I set about redesigning the plan all over again to allow running the container without root access. That turned out to be hard to do with docker because of what seem to me like some poor design choices in docker, so I switched to podman. I picked podman because it could work with the files I had already set up for docker without needing to change them. It also seems like the more secure option because unlike docker it doesn't run a daemon as root. In fact, it doesn't run a daemon at all. Podman made it possible to set this up so that nothing more runs as the real root, and very little runs as the "pretend" root inside the container.

Also, I was surprised to find that the container starts a bit faster in podman. It's a small change of about 1 second to start the containers but I was still happy to see it.

This process of running with as little root as possible helps to address these problems that come with the docker version of Seafile:

* Seafile with docker runs as root by default. \
  It is isolated from other parts of the system by the container stuff, but not the normal user permissions. This can be helped a bit by not running as root inside the containers (with the NON\_ROOT=true), but there are still parts running as root (including docker itself).
* Docker itself runs as root. \
  There's more attack surface with a new daemon running with root access than there used to be.
* Exposing mariadb and memecached to the network.  \
  Mariadb and memcached need to be bound to the network address instead of localhost, making it easier for a typo with firewalls to expose them to the network. So now I plan to make extra tests in my monitoring system to test and send an alert if it can connect to either over the network.&#x20;
* I decided not to include seadoc. It appears to be written in javascript, and I don't have time to check over 400 npm packages aren't subject to the famous npm supply-chain attacks. Also seadoc doesn't seem to be opensource (or at least I can't find the source and license anywhere), so I can't expect that anyone besides the original developer has checked either. So that's a solid pass for me, at least for now.

What's different in the process? Not much. Instead of a lot of steps to install docker, just "apt install podman-docker uidmap". I also created a user I called "podman". I put the home directory on the /seafile-data disk because the OS disk is small, and this is where the files downloaded to make the docker containers will be stored, it's likely that the default home directory will work for you.&#x20;

The container processes will run as this podman user instead of running as root. We will also turn on "linger" for this user so that their podman stuff can continue to run when they aren't logged in.\
\
`sudo adduser --home /seafile-data/podman-home \`\
&#x20;   `--shell /usr/sbin/nologin podman` \
`sudo loginctl enable-linger podman`

Check that this user was added to the /etc/subuid and /etc/subgid, and if not, add them. Should look like:&#x20;

`podman:165536:65536`

Now we need to log in to this podman user in a way that will activate any systemd user services, and dbus and such. If you log in through the terminal it should work, but if you have logged in with SSH, you might need to take extra steps.  The easiest option I found was "`apt install systemd-container`", then "`machinectl shell podman@`".  In that shell we will do a few things to setup the service. First, we need podman available for the service, so: \
`systemctl --user enable podman.socket` \
`systemctl --user start podman.socket`

Find out the uid of this user by running the id command.  In my case that was 1001, and we will need that in a moment, as we add the service file (where it has /run/user/1001/podman it needs that number to be this user's uid):

`mkdir -p ~/.config/systemd/user` \
`nano ~/.config/systemd/user/seafile-containers.service`

Paste in:&#x20;

```
[Unit]
Description=Seafile Podman containers via docker-compose
Wants=network-online.target
After=network-online.target
RequiresMountsFor=/seafile-data
Requires=podman.socket mariadb.service memcached.service 

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Environment=PODMAN_USERNS=keep-id
Environment=COMPOSE_HTTP_TIMEOUT=300
Restart=always
TimeoutStartSec=60
TimeoutStopSec=60
ExecStart=/usr/bin/docker-compose -H unix:///run/user/1001/podman/podman.sock up
ExecStop=/usr/bin/docker-compose -H unix:///run/user/1001/podman/podman.sock down
Type=simple
WorkingDirectory=/seafile-data/docker

[Install]
WantedBy=default.target
```

save and exit. Then activate that service: \
`systemctl --user daemon-reload` \
`systemctl --user enable seafile-containers` \
`systemctl --user start seafile-containers`

And there we go, container running with as little use of root privs as possible.

Not yet addressed concerns:

* It is now harder to look at what libraries are being installed with Seafile, and black boxes are never good for security. But I'm not a good programmer, and I was never likely to spot any significant problems before it was too late anyway.
* Logging to the network's central logging server. This worked with the old version, but I think we can get by for now and fix that later.
* I had some wrapper scripts to easily kick off a seaf-fsck and seaf-gc that now need to be rewritten. I have tested that I can manually run both, so this should be easy to do once I have time.
* Monitoring system needs some additional sensors to detect container state, and check for firewall issue exposing mariadb or memecached to the network.

## Additional notes

It looks like running a seaf-fsck or seaf-gc should be pretty easy to script.  It seems to work to do:\
`sudo -u podman DOCKER_HOST=unix:///run/user/1001/podman/podman.sock docker exec -it seafile su seafile -c "/opt/seafile/seafile-server-latest/seaf-fsck.sh -r"`

And for seaf-gc:

`sudo -u podman DOCKER_HOST=unix:///run/user/1001/podman/podman.sock docker exec -it seafile su seafile -c "/opt/seafile/seafile-server-latest/seaf-gc.sh --rm-fs -t 8"`&#x20;

