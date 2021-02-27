---
title: "Making Miniflux portable"
date: 2021-02-26T23:22:24Z
tags:
    - Applications
---

# Making Miniflux portable

After hunting for a while for a good offline RSS reader I decided on
[Miniflux](https://miniflux.app/). It is designed to be hosted on a home server or
something, but it functions just fine as a standalone desktop application. The use of
the browser to access content is one of my favourite things about it, as it means all my
[qutebrowser](https://qutebrowser.org/) keyboard combinations will work.

There's just one thing that it's missing: it's not portable. It uses PostgresSQL for the
database; no option for SQLite or any other kind of portable format. Let's fix that! 

## Overview

- Configure Postgres to run as the current user, and put all it's data in a single
  directory
- Configure Miniflux to connect to this instance of Postgres
- Archive the data directory when not in use
- Only launch Postgres and Miniflux when needed
- Wrap it all up in a script

## Disclaimer
I don't know much about Postgres, I haven't used it much. And I'm relatively amateur at
shell scripting, so there may be better ways to achieve this.

Also this guide is only for Linux. I assume you'd be able to achieve the same thing on
Windows or Mac by modifying the commands, but I have no interest in either of those
platforms myself.

## Prerequisites
- A Linux machine
- PostgresSQL installed
- Miniflux installed
- Be comfortable using the command line

## Steps
### 1: Initial PostgresSQL setup
- Create your database directory, I recommend `~/.local/share/miniflux`
`mkdir ~/.local/share/miniflux`

- Initialise the database in this directory
`initdb ~/.local/share/miniflux`

- Open up the contained `postgresql.conf` and uncomment the line `#unix_socket_directories`
- Set it to the _full path_ to your data directory
`unix_socket_directories = '/home/b/.local/share/miniflux'`
    - This will tell Postgres to put it's lock file in this directory, which will avoid
      permissions issues and also prevent conflict with any other Postgres databases


### 2: Create our initial script
Create a new script (make it executable), this will eventually become our Miniflux
executable.  
Postgres can be configured using environment variables, these are the ones we need:
- `PGROOT` - Root directory to store the data
- `PGPORT` - Port for Postgres to listen on; pick anything you like
- `PGSSLMODE` - We don't need SSL when it's all local  

The last line launches Postgres in the `$PGROOT` directory

```bash
# miniflux.sh
#!/bin/sh

export PGROOT=~/.local/share/miniflux
export PGPORT=5434
export PGSSLMODE=disable

/usr/bin/postgres -D $PGROOT
```

### 3: Configure the database for Miniflux
Launch your script, Postgres should start up and listen for connections.
```console
$ ./miniflux.sh
2021-02-27 11:39:39.482 GMT [20666] LOG:  starting PostgreSQL 13.2 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 10.2.0, 64-bit
2021-02-27 11:39:39.484 GMT [20666] LOG:  listening on IPv6 address "::1", port 5434
2021-02-27 11:39:39.484 GMT [20666] LOG:  listening on IPv4 address "127.0.0.1", port 5434
2021-02-27 11:39:39.584 GMT [20666] LOG:  listening on Unix socket "/home/b/.local/share/miniflux/.s.PGSQL.5434"
2021-02-27 11:39:39.687 GMT [20673] LOG:  database system was shut down at 2021-02-27 11:37:29 GMT
2021-02-27 11:39:39.749 GMT [20666] LOG:  database system is ready to accept connections
```

In a new terminal we are going to loosely follow the [Miniflux Data Configuration
guide](https://miniflux.app/docs/installation.html). As we are using Postgres in a
non-standard configuration we need to specify the host and port for each of the
commands.  
Enter a password when prompted; doesn't really matter what it is, I just set
it to 'miniflux'.

```console
$ createuser -h 127.0.0.1 -p 5434 -P miniflux
Enter password for new role:
Enter it again:

$ createdb -h 127.0.0.1 -p 5434 -O miniflux miniflux
```

Set the `DATABASE_URL` environment variable, Miniflux will read this variable to
determine the database connection string.  
Run the Miniflux migration and create the admin user. Doesn't really matter what the
admin user and password are, as it's only a local installation.

```console
$ export DATABASE_URL="host=/home/b/.local/share/miniflux port=5434 user=miniflux dbname=miniflux sslmode=disable"

$ miniflux -migrate
-> Current schema version: 0
-> Latest schema version: 44
* Migrating to version: 1
* Migrating to version: 2
...
* Migrating to version: 43
* Migrating to version: 44

$ miniflux -create-admin
Enter Username: admin
Enter Password:
```

Test miniflux.
```console
$ miniflux

[INFO] Starting Miniflux...
[INFO] Starting scheduler...
[INFO] Listening on "127.0.0.1:8080" without TLS
```

Open up `127.0.0.1:8080` in your browser. If you see Miniflux prompting you to log in,
all is working. You should be able to log in with your admin credentials.
![Miniflux login screen](/img/miniflux-login-screen.png)

### 4: Create an archive
We're going to achieve portability by archiving the database directory when not in use.
We can create it once now, then automate the archiving and unarchiving in our final
script.

Shut down Miniflux and then Postgres before proceeding.

The archive file should be placed where you want it to be portable. In my case I'm
putting it in my `~/sync` directory, which is synchronised between my machines.


```console
$ tar -xvf ~/sync/miniflux.tar.gz -C ~/.local/share/miniflux .

./
./postmaster.opts
./pg_notify/
...
./pg_logical/snapshots/
./pg_logical/replorigin_checkpoint
```

### 5: Wrap it all up in a script
When we want to launch Miniflux we need to extract the archive, launch Postgres, launch
Miniflux, and launch the browser. When we're done browsing/reading our feeds, we want to
shut them down and update the archive. Update your `miniflux.sh` script to add in all
the following steps.

Pay special attention to the browser command in the middle. This is the process that the
script will wait for before shutting down Miniflux/Postgres. You ideally want this to be
a standalone process, i.e. not a command that just opens a new tab in an existing
browser instance, else Miniflux and Postgres won't shut down until all your tabs are
closed. I am using [qutebrowser](https://qutebrowser.org/) and am achieving this by
launching it with a different profile directory just for Miniflux. Other browsers should
have a similar alternative, you'll have to look it up for your browser.

```bash
#!/bin/sh

# Exit immediately if there's any errors
set -e

# Postgres settings
export PGROOT=~/.local/share/miniflux
export PGPORT=5434
export PGSSLMODE=disable

# Miniflux settings
export DATABASE_URL="host=/tmp port=5434 user=miniflux dbname=miniflux sslmode=disable"
export LISTEN_ADDR="127.0.0.1:8081"

# Extract database from archive
/usr/bin/tar -xvf ~/sync/miniflux.tar.gz -C $PGROOT --overwrite

# Launch postgres
/usr/bin/postgresql-check-db-dir $PGROOT
/usr/bin/postgres -D $PGROOT &
PG_PID=$!

# Wait for postgres to be fully up before starting miniflux
# Not sure how else to achieve this!
sleep 1

# Launch miniflux
/usr/bin/miniflux &
MINIFLUX_PID=$!

# Wait for miniflux to be fully up before refreshing
# Not sure how else to achieve this either
sleep 1

# Browser
/usr/bin/qutebrowser http://localhost:8081/unread --target window --basedir ~/.local/share/qutebrowser/miniflux

# Shutdown miniflux
kill -INT $MINIFLUX_PID
wait $MINIFLUX_PID

# Shutdown postgres
kill -INT $PG_PID
wait $PG_PID

# Trash the old archive
/usr/bin/rm ~/sync/miniflux.tar.gz

# Re-archive the database
/usr/bin/tar -zcvf ~/sync/miniflux.tar.gz -C $PGROOT .
```

## Testing it
Lets test it.

```console
$ ./miniflux.sh

./
./pg_hba.conf
...
./pg_xact/0000
2021-02-27 12:29:29.279 GMT [29396] LOG:  starting PostgreSQL 13.2 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 10.2.0, 64-bit
2021-02-27 12:29:29.279 GMT [29396] LOG:  listening on IPv6 address "::1", port 5434
2021-02-27 12:29:29.279 GMT [29396] LOG:  listening on IPv4 address "127.0.0.1", port 5434
2021-02-27 12:29:29.282 GMT [29396] LOG:  listening on Unix socket "/home/b/.local/share/miniflux/.s.PGSQL.5434"
2021-02-27 12:29:29.290 GMT [29398] LOG:  database system was shut down at 2021-02-27 11:10:16 GMT
2021-02-27 12:29:29.307 GMT [29396] LOG:  database system is ready to accept connections
[INFO] Starting Miniflux...
[INFO] Starting scheduler...
[INFO] Listening on "127.0.0.1:8081" without TLS
```

At this point your new browser instance should launch. You can log in and interact with
Miniflux.

When you're done, close your browser instance. You should see Miniflux and Postgres
shutdown, and your archive updated.

```console
[INFO] Shutting down the process...
[INFO] Process gracefully stopped
2021-02-27 12:32:59.880 GMT [29396] LOG:  received fast shutdown request
2021-02-27 12:32:59.896 GMT [29396] LOG:  aborting any active transactions
2021-02-27 12:32:59.899 GMT [29396] LOG:  background worker "logical replication launcher" (PID 29404) exited with exit code 1
2021-02-27 12:32:59.899 GMT [29399] LOG:  shutting down
2021-02-27 12:32:59.941 GMT [29396] LOG:  database system is shut down
./
./pg_hba.conf
...
./pg_xact/
./pg_xact/0000
```

## Limitations
- **Feeds don't automatically refresh**  
Because Miniflux is never open in the background, it can't refresh feeds automatically.
You need to manually refresh the feeds each time you launch. There's probably a way
around this, but I haven't found it enough of a problem to try figure it out yet.
![Manually refresh feeds](/img/miniflux-refresh-feeds.png)

- **Simple script has no recovery in the case of errors**  
This is a simple shell script. If something goes wrong during execution, it has no
recovery included. You have to go and fix it yourself.

- **No backups included by default**  
There's no backups done or anything, I just delete the old archive and replace it with
the new one. This carries a risk that the whole database could be lost if something
breaks. You should probably back up your archive occasionally.

## Conclusion
Now I can use Miniflux as a standalone application between multiple machines, and port
my data between them!

You can see a copy of this script in [my
dotfiles](https://github.com/{{< github-user >}}/dotfiles/tree/master/bin/miniflux), which get
synced between my machines too.
