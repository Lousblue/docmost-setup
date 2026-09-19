# docmost-setup

Deploy Docmost with Docker Compose, backups included.

Docmost is a good wiki, and its compose file is three services and a dozen
variables. What it does not come with is the part that matters a year later:
a backup you can actually restore, taken as often as you want, kept where you
want - a NAS included - and rotated so it never fills the disk. This script
sets up the stack, asks for what it needs, and gives you `--backup`,
`--restore`, `--update` and a timer, all from one file. Run it again to
change anything: the folder, the frequency, the port, the mail server.

```
$ docmost-setup

System
  docker        29.8.1, compose 5.5.1
  docmost       not installed

Questions
  press enter to accept the value in brackets

  The address people will open. Docmost writes it into links and mails, so it must be the real one, https included.
  Public URL: https://docs.example.com

  The port Docmost listens on, on this machine. Your reverse proxy or tunnel points here.
  Port [3000]:

  A long random string that signs sessions and tokens. Leave it empty and one is generated for you.
  App secret (hidden, empty = generate):

  The Postgres password, used only between the containers. Leave it empty and one is generated.
  Database password (hidden, empty = generate):

  Docmost sends invitations and password resets by mail. Without a server, they are written to its log instead.
  Send mail through an SMTP server? [y/N]

  Where backups go: any folder, and a folder on a NAS mount is the point. The newest stays at the root, older ones move to Old. Change it here any time; backups already made stay where they are.
  Backup folder [/var/backups/docmost]: /mnt/nas/Backups/Docmost
    on the network share mounted at /mnt/nas

  How many older backups Old keeps before the oldest is deleted. With a backup every 6 hours, 8 means two days back.
  Backups kept in Old [8]:

  How often the backup runs by itself. A run missed because the machine was off happens at the next boot.
  Automatic backup: never, hourly, 2h 3h 4h 6h 8h 12h, daily, weekly [daily]: 6h

To apply
  docmost       installed in /opt/docmost
  url           https://docs.example.com, port 3000
  secrets       app secret generated, database password generated
  mail          none, written to the log
  backups       /mnt/nas/Backups/Docmost, on the network share mounted at /mnt/nas, 8 kept in Old
  schedule      every 6 hours

  Apply these 7 steps? [Y/n]

┌─ [1/7] Backup folder  22:19:50
│  /mnt/nas/Backups/Docmost, on the network share mounted at /mnt/nas
└─ ✓ writable, 0 backups there

┌─ [2/7] Environment  22:19:50
│  /opt/docmost/.env, mode 600
└─ ✓ app secret generated, database password generated

┌─ [3/7] Compose file  22:19:50
│  /opt/docmost/docker-compose.yml
│  docmost on port 3000, postgres and redis reachable from the compose network only
└─ ✓ written

┌─ [4/7] Validation  22:19:50
└─ ✓ syntax valid, secrets resolved from .env

┌─ [5/7] Start  22:19:50
│  3 containers running
└─ ✓ docmost, postgres and redis up

┌─ [6/7] Check  22:20:56
└─ ✓ http 200 on port 3000

┌─ [7/7] Automatic backup  22:21:00
│  /usr/local/bin/docmost-setup --backup, as louis
│  every 6 hours, up to 5 min later; a run missed while the machine was off happens at the next boot
│  waits for /mnt/nas to be mounted
└─ ✓ next: Sun 2026-09-20 00:02:41 CEST

──────────────────────────────────────────────────────────────
Done in 1 min 10 s, 0 warnings
  url           https://docs.example.com
  local         http://10.10.40.30:3000
  backups       /mnt/nas/Backups/Docmost, every 6 hours
  change        run docmost-setup again: the values in place are the defaults
  log           /opt/docmost/docmost-setup.log
```

## Install

```bash
git clone https://github.com/Lousblue/docmost-setup.git
sudo install -m 755 docmost-setup/docmost-setup /usr/local/bin/
```

One bash script. It needs Docker with the compose plugin, reachable without
`sudo` by the user who runs it (see [docker-setup](https://github.com/Lousblue/docker-setup)),
and `sudo` for the timer.

## Use

```bash
docmost-setup                       # ask, then install or reconfigure
docmost-setup --status              # look, change nothing
docmost-setup --backup              # dump the database and the uploads now
docmost-setup --restore             # put the newest backup back
docmost-setup --restore 2026-09-01  # or that day's last one
docmost-setup --restore --from /mnt/old-nas/Docmost   # or one from another folder
docmost-setup --update              # back up, then pull newer images
```

Pass any setting and it stops asking - which is also what happens without a
terminal. What you did not mention keeps the value already in
`/opt/docmost/.env`, or takes the built-in default on a first run. So
changing one thing is one option, and nothing else moves:

```bash
docmost-setup --backup-dir /mnt/nas/Backups/Docmost   # move the backups to the NAS
docmost-setup --every 6h                              # back up four times a day
docmost-setup --every weekly --at 04:15               # or once a week
docmost-setup --every never                           # or only by hand
docmost-setup --port 3001
```

A first install without questions:

```bash
docmost-setup --url https://docs.example.com --backup-dir /mnt/nas/Backups/Docmost --every 6h --yes
APP_SECRET=... DB_PASSWORD=... docmost-setup --url https://docs.example.com --yes
```

### Settings

| Setting | Default | Option |
|---|---|---|
| Public URL | required | `--url` |
| Port on this machine | `3000` | `--port` |
| Backup folder | `/var/backups/docmost` | `--backup-dir` |
| Backups kept in `Old` | `8` | `--keep` |
| Automatic backup | `daily` | `--every never\|hourly\|2h\|3h\|4h\|6h\|8h\|12h\|daily\|weekly` |
| Time of day, for daily and weekly | `03:30` | `--at HH:MM` |
| Restore the newest backup after a fresh install | no | `--restore-latest` |
| SMTP | none, mails go to the log | `--smtp-host`, `--smtp-port`, `--smtp-user`, `--smtp-from`, `--smtp-name`, `--smtp-secure` |

Hour counts are the ones that divide a day, so the runs fall at the same
times every day: `6h` is 00:00, 06:00, 12:00 and 18:00.

### Secrets

Read from the environment, asked with the input hidden when a terminal is
there, refused otherwise:

| Variable | If empty |
|---|---|
| `APP_SECRET` | generated, 64 hex characters |
| `DB_PASSWORD` | generated, 32 hex characters |
| `SMTP_USERNAME` | asked with the other SMTP settings |
| `SMTP_PASSWORD` | asked, only when SMTP has a user |

On a machine already set up, the values in `.env` are kept. The Postgres
password in particular cannot change after the database exists: a different
`DB_PASSWORD` in the environment is ignored, with a warning that says so.

They live in `/opt/docmost/.env`, mode 600, and nowhere else. They are never
printed and never logged. Any tool that can put a secret in the environment
works: `pass-cli run --env-file docmost.env -- docmost-setup`.

## Backups

A backup is two files named by date and time: `prod-2026-09-20-0330.sql`, a
`pg_dump` of the database, and `docmost-prod-data-2026-09-20-0330.tgz`, the
uploads. The newest pair sits at the root of the backup folder; when a new
one arrives, the previous pairs move to `Old`, which keeps the last N and
deletes the rest. Both files are checked before they count: the dump must
start like a PostgreSQL dump, the archive must pass `gzip -t`, and the copies
must match the originals in size.

### On a NAS

Give it a folder on a mounted share, `/mnt/nas/Backups/Docmost`. The script
notices that the folder lives on a network filesystem (cifs, nfs, sshfs),
says so, and from then on protects you from the quiet failure of that setup:
when the share is not mounted, the folder still exists, empty, on the local
disk, and a backup written there looks like a success. So:

- `--backup` checks the mount first and refuses with a clear message
- the timer's service carries `RequiresMountsFor=`, so systemd mounts the
  share before the job, and the job fails in the journal if it cannot
- `--status` says `IS NOT MOUNTED, backups will fail` instead of showing an
  empty folder
- a path under `/mnt` or `/media` that turns out to be local gets a warning
  when you type it: that is almost always a share that is not mounted

Mounting the share is not this script's job;
[vm-init](https://github.com/Lousblue/vm-init) does it, or a line in
`/etc/fstab`.

### Changing the folder or the frequency

Run `docmost-setup` again and answer differently, or pass the one option.
The plan shows the old folder next to the new one; backups already made stay
where they are, and `--restore --from OLD_FOLDER` still reads them. The
timer is rewritten, or removed for `never`.

### Restore

`--restore` lists what it found, root and `Old`, newest first, and asks you
to type `YES` before it empties the schema, imports the dump, replaces the
uploads and restarts Docmost. Then it counts users, spaces, pages and
attachments so you can see it worked. `--yes` skips the question for scripts.

Pick a backup by date, `2026-09-01`, which takes that day's last one, or by
date and time, `2026-09-01-0330`. `--from DIR` reads another folder than the
configured one.

**Moving to a new machine**: install there with the backup folder pointing at
the same share. The wizard sees the backups and offers to restore the newest
once Docmost is up; without questions, that is `--restore-latest`.

### Update

`--update` always backs up first, because a new Docmost version may migrate
the database schema, and there is no way back from that without a dump. Its
last line tells you which backup to `--restore` if the update goes wrong.

### The timer

A systemd timer, not a crontab line: `systemctl status docmost-backup.timer`
tells you when it runs next, its output goes to
`journalctl -u docmost-backup.service`, and a run missed because the machine
was off happens at the next boot. `--uncron` removes it, `--cron` puts it
back from the saved schedule.

## Files it writes

| | |
|---|---|
| `/opt/docmost/.env` | settings and secrets, mode 600 |
| `/opt/docmost/docker-compose.yml` | the stack: docmost, postgres 16, redis 7 |
| `/opt/docmost/docmost-setup.log` | everything compose, pg_dump and tar had to say |
| `/etc/systemd/system/docmost-backup.{service,timer}` | the automatic backup |
| the backup folder | `prod-*.sql`, `docmost-prod-data-*.tgz`, `Old/` |

Data lives in three named Docker volumes, `docmost_appdata`, `docmost_db_data`
and `docmost_redis_data`. A reconfigure never touches them.

## Tested on

Ubuntu 24.04 with Docker Engine 29, against the real images: install through
the wizard, backup, rotation, restore, update, reconfigure with the secrets
kept. The backup logic added since - schedules, the network share checks,
moving the folder, `--from`, `--restore-latest` - is covered by a harness
that stubs docker and systemd; it has not yet run against a physical NAS.

## Caveats

- **Docmost needs its real URL.** Invitations and reset links are built from
  it. `http://10.0.0.5:3000` works on a LAN; a domain behind a reverse proxy
  or a tunnel is the usual setup, and the proxy points at the port here.
- **Port 3000 is published on every interface** of the machine. Put a
  firewall in front if that machine is reachable from further than you want.
- **Restore replaces everything** in the database and the uploads. There is
  no merge. Take a `--backup` first if the current state might matter.
- **Backups made before this naming** (`prod-2026-09-01.sql`, no time) are
  still listed and restored; they sort with the others by date.

## Licence

MIT.
