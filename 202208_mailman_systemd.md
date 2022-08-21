
# Replacing Mailman’s persistent queue-runners with systemd

I run some very low traffic mailing lists using Mailman 2.x.  So low traffic that I haven’t yet looked into a migration to Mailman 3.

The way that queuing works in mailman is that commands are processed into python `.pck` files, and deposited in a subdirectory of (for example) `/var/lib/mailman/qfiles/`. Other processes, "queue runners", poll these directories for new files and trigger other actions when they're found.

When you install the [Ububtu Mailman package](https://packages.ubuntu.com/focal-updates/mailman) it'll install a systemd job to start `mailmanctl` which in turn spins up eight queue runners.

```console

$ systemctl status mailman
● mailman.service - Mailman Master Queue Runner
[…]
     Tasks: 9 (limit: 1129)
     Memory: 102.6M
        CPU: 3.646s
     CGroup: /system.slice/mailman.service
             ├─2597688 /usr/bin/python2 /usr/lib/mailman/bin/mailmanctl -s start
             ├─2597689 /usr/bin/python2 /var/lib/mailman/bin/qrunner --runner=ArchRunner:0:1 -s
             ├─2597690 /usr/bin/python2 /var/lib/mailman/bin/qrunner --runner=BounceRunner:0:1 -s
             ├─2597691 /usr/bin/python2 /var/lib/mailman/bin/qrunner --runner=CommandRunner:0:1 -s
             ├─2597692 /usr/bin/python2 /var/lib/mailman/bin/qrunner --runner=IncomingRunner:0:1 -s
             ├─2597693 /usr/bin/python2 /var/lib/mailman/bin/qrunner --runner=NewsRunner:0:1 -s
             ├─2597694 /usr/bin/python2 /var/lib/mailman/bin/qrunner --runner=OutgoingRunner:0:1 -s
             ├─2597695 /usr/bin/python2 /var/lib/mailman/bin/qrunner --runner=VirginRunner:0:1 -s
             └─2597696 /usr/bin/python2 /var/lib/mailman/bin/qrunner --runner=RetryRunner:0:1 -s
```

On a low traffic installation this is overkill. Some of those queues will rarely if ever trigger, for example the NNTP relay, and really only the Incoming and Outgoing queues benefit from being processed immediately.

## Replacing queues with a timer

My first attempt to reduce the number of persistent jobs was to set the number of runners to zero (for all but in and out) in `mm_cfg.py`

```python
QRUNNERS = [
    ('ArchRunner',     0), # messages for the archiver
    ('BounceRunner',   0), # for processing the qfile/bounces directory
    ('CommandRunner',  0), # commands and bounces from the outside world
    ('IncomingRunner', 1), # posts from the outside world
    ('NewsRunner',     0), # outgoing messages to the nntpd
    ('OutgoingRunner', 1), # outgoing messages to the smtpd
    ('VirginRunner',   0), # internally crafted (virgin birth) messages
    ('RetryRunner',    0), # retry temporarily failed deliveries
    ]
```

And then added a systemd service in `/etc/systemd/system/mailman-qrunner-all.service`

```systemd
[Unit]
Description=Mailman qrunner all

[Service]
Type=oneshot
User=list
ExecStart=/usr/lib/mailman/bin/qrunner -r All -o
PrivateTmp=true
```

And a corresponding every-quarter-hour timer in `/etc/systemd/system/mailman-qrunner-all.timer`

```systemd
[Unit]
Description=Mailman qrunner all

[Timer]
OnCalendar=*:0,15,30,45
Persistent=true

[Install]
WantedBy=timers.target
```

Since we expect that the queues (except maybe `in` and `out`) are empty, there’s no problem with just running all queues at the same time every 15 minutes, and relying on the queue runners to pick up on new mail immediately.

Restarting the `mailman` service now spawns fewer processes:

```console
     CGroup: /system.slice/mailman.service
             ├─33886 /usr/bin/python2 /usr/lib/mailman/bin/mailmanctl -s start
             ├─33887 /usr/bin/python2 /var/lib/mailman/bin/qrunner --runner=IncomingRunner:0:1 -s
             └─33888 /usr/bin/python2 /var/lib/mailman/bin/qrunner --runner=OutgoingRunner:0:1 -s
```

## Replacing queues with path

Those final two queues could be replaced by anything able to trigger `qrunner` in the  event that a file appeared in them.

Systemd has this capacity in the form of [.path](https://www.freedesktop.org/software/systemd/man/systemd.path.html) units.  We can check the named directories for new files, and if they appear, trigger the previously defined `mailman-qrunner-all.service`. (Note that Path uses inotify, so it won’t trigger on remote directories, such as NFS mounts.)

Create, start and enable, `/etc/systemd/system/mailman-qrunner-all.path`

```systemd
[Unit]
Description = Mailman qrunner all

[Path]
PathChanged = /var/lib/mailman/qfiles/in/
PathChanged = /var/lib/mailman/qfiles/out/
PathChanged = /var/lib/mailman/qfiles/archive/
PathChanged = /var/lib/mailman/qfiles/bounces/
PathChanged = /var/lib/mailman/qfiles/commands/
PathChanged = /var/lib/mailman/qfiles/retry/
PathChanged = /var/lib/mailman/qfiles/virgin/
PathChanged = /var/lib/mailman/qfiles/news/
MakeDirectory = true
DirectoryMode = 2770

[Install]
WantedBy=network.target
```

At which point you can disable `mailman.service` and `mailman-qrunner-all.timer` (or perhaps just reduce the frequency) and have no mailman python2 processes running persistently.
