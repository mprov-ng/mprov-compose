# mprov-compose
A docker compose file to start up an entire mprov stack on a single node.

We recommend using docker-ce for your docker environment.


Pacakge Dependencies: 

- text editor (vim)
- tmux or screen
- epel-release
- git
- docker-ce

General installation instructions:
```
- install docker-ce 
  - https://docs.docker.com/engine/install/rhel/
- git clone https://github.com/mprov-ng/mprov-compose.git
- cd mprov-compose
- copy stack.env.dist to .env
- edit .env (set the database passwords first; the stack won't start without them)
- sudo touch /etc/genders
- sudo install -d /export /var/lib/mprov
- docker compose --env-file .env up -d
< go have a sandwich >
- http://ipaddress/admin/ to login as admin (see "The admin password" below)
```

The job server bind-mounts `/etc/genders`, `/export` and `/var/lib/mprov` from
the host, so other host-side tooling can see them outside the container. On a
fresh host none of those exist yet, and Docker creates a missing bind-mount
path itself before the container starts — for a file like `/etc/genders` it
creates a **directory** instead, which breaks libgenders. Run the `touch`/
`install -d` commands above once before the first `up` to avoid that.
(`/var/lib/mysql` doesn't need this: it's a named Docker volume, not a bind
mount, so the `db` container's own entrypoint initializes it correctly.)

## Setting up the provisioning network

Nodes network-boot from the job server, and they find it **by name**. The
job server tells iPXE to `chain tftp://<job server hostname>/menu.ipxe`,
where the hostname is the job server container's own, `MPROV_JOBSERVER_HOSTNAME`
in `.env`. A node that can't resolve that name stops at that line, then
PXE-boots again, over and over.

The job server's dnsmasq can answer for its own name, but only if the
provisioning network is set up for it in the mPCC. Under Networks, on the
network the nodes boot from, set:

- **Enable DHCP Server?** and **Enable Network Booting?** on, and a
  **DHCP Range Start** and **DHCP Range End**.
- **Manage DNS?** on. The job server only publishes its own name in a
  network's DNS when this is on.
- **Primary Nameserver** to the job server's address on that network.
- **Domain Name**. DHCP hands out the domain only when a Primary Nameserver
  is set too, and nodes need that domain to look up the job server's bare
  hostname.

The job server uses host networking, so it serves only networks where the
host itself has an address. The host needs an interface with an address in
that network's subnet, and that address is the one to use for
**Primary Nameserver**.

If the network's DNS comes from a nameserver of your own instead, leave
**Manage DNS?** off and point **Primary Nameserver** at that server. That
server then has to resolve `MPROV_JOBSERVER_HOSTNAME`, under the network's
**Domain Name**, to the job server's address on the provisioning network.

To check, boot a node and watch its console. It should load
`tftp://<job server hostname>/menu.ipxe` and carry on booting, not go back
to PXE.

## Image versions

`docker-compose.yml` pins the mPCC and job server images to release tags, a
pair from the same major.minor (e.g. mPCC `v0.7.3` with job server `v0.7.1`).
They have to match: the mPCC refuses a job server from another major.minor
with a `426`, and the job server won't run against it. The images used to
track `:dev` and `:latest`, which pulled whatever had been built last and
could pair an mPCC from one major.minor with a job server from another.

To upgrade, pull the new compose file, which moves both tags together, then
pull the images and recreate the containers:

```
git pull
docker compose pull
docker compose --env-file .env up -d
```

If you change the tags yourself, change both in the same edit, and keep them
on the same major.minor.

If your stack has already run the old `:dev` mPCC, its database may have
migrations newer than the pinned release, and an older mPCC can't safely run
against it. Keep `:dev` for the mPCC in your copy of `docker-compose.yml`
until the next release catches up, rather than moving back.

## Upgrading to 0.7.0

0.7.0 replaces the API key that the job server and every node shared with
an identity of their own
([mprov-ng/mprov_control_center#381](https://github.com/mprov-ng/mprov_control_center/issues/381)).
The mPCC runs its own certificate authority (CA). The job server enrolls
with it for a certificate the first time it starts. Nodes enroll as they
boot. Image builds get a short-lived credential instead of the key. The
[mPCC](https://github.com/mprov-ng/mprov_control_center/releases/tag/v0.7.0)
and [job server](https://github.com/mprov-ng/mprov_jobserver/releases/tag/v0.7.0)
release notes have the full list of changes; this section covers what
upgrading this stack takes.

Take a backup first (see "Backups" below) and copy it off the host. The
mPCC's first 0.7.0 start converts any database tables still on an old
character set to utf8mb4 in place, before migrating, and on a large
database that can take several minutes.

In this stack the job server enrolls by itself. The mPCC leaves a one-time
enrollment token and its CA certificate in `/etc/mprov` (the `mprov-conf`
volume), and the job server keeps its key and certificate on the
`jobserver-tls` volume. Enrolling needs HTTPS, so before upgrading, edit
`.env`:

- Change `MPROV_URL` to `https://`, keeping the same address, which must be
  one of `MPROV_MPCC_IPs`, e.g. `MPROV_URL=https://192.168.1.10/`. Nodes
  still boot over plain HTTP: the job server gives iPXE an `http://` URL for
  the same address.
- Remove `MPROV_APIKEY`. In this stack the job server always took the key
  from `/etc/mprov/apikey` instead, so the line did nothing.
- Remove `MPROV_SKIP_VERSION_CHECK` if you set it. The mPCC and job server
  are upgraded together here, and the check is what tells you when they
  aren't.

Then upgrade as in "Image versions" above, and check the job server
enrolled:

```
docker compose logs jobserver | grep -i enroll
```

Then, on the nodes' side:

1. Re-upload `install_mprov_script-runner.sh`, `setupBMC.py`,
   `register_IPv6_ll.py`, `localTempDisk.py` and `report_booted.py` from
   mprov_scripts into the mPCC's Scripts, **before** rebuilding any image.
   An image built by a 0.7.0 job server has no system `python3`, so an old
   copy of those scripts fails in it. Don't add `install_mprov_jobserver.sh`
   to an ordinary compute image: it's only for an image that runs as a
   second job server, and it reinstalls the job server from PyPI over the
   one the image already has.
2. Add `install_mprov_cert_renew.sh` to your images, so stateful nodes
   renew their certificate before it expires. A fresh install has it in the
   default compute image already.
3. If you use a custom iPXE script, add `{{ boot.enroll }}` to it, or nodes
   booted with it can't enroll.
4. Rebuild every image. An image built before 0.7.0 has the shared key in
   it, and its nodes can't reach the mPCC while the key is refused.

From 0.7.0 the mPCC refuses the shared key by default
(`MPROV_SHARED_KEY_MODE=off`). Nodes still running an image built before
0.7.0 lose access to the mPCC as soon as it's upgraded, until their image
is rebuilt. To keep them working meanwhile, set `MPROV_SHARED_KEY_MODE=nodes`
in `.env` (it's commented out in `stack.env.dist`): the mPCC then accepts the
key only for what a node does about itself. `all` accepts it for everything,
as before 0.7.0. Once every image is rebuilt, remove the line again, so the
default `off` applies, and revoke the key under API Keys in the mPCC admin.
0.8.0 removes it.

## Upgrading to 0.7.1

0.7.1 is a bug-fix release: nodes enroll more reliably when a whole fleet
boots at once, and a few enrollment and recovery problems are fixed. The
[mPCC](https://github.com/mprov-ng/mprov_control_center/releases/tag/v0.7.1)
and [job server](https://github.com/mprov-ng/mprov_jobserver/releases/tag/v0.7.1)
release notes have the details. If you're coming from before 0.7.0, do
"Upgrading to 0.7.0" above first.

Nothing in `.env` has to change. Upgrade as in "Image versions" above.

The mPCC now starts one Apache mod_wsgi process per CPU, from 2 to 8.
To choose the number yourself, set `MPROV_WSGI_PROCESSES` in `.env` (it's
commented out in `stack.env.dist`).

Then, on the nodes' side:

1. Re-upload `localTempDisk.py`, `register_IPv6_ll.py`, `report_booted.py`
   and `setupBMC.py` from mprov_scripts into the mPCC's Scripts. The copies
   a 0.7.0 install came with still report themselves as 0.6, so the mPCC
   logs a version-mismatch warning for every call they make. A fresh 0.7.1
   install has the current ones already.
2. Rebuild your images. A node whose certificate was revoked now picks up
   a "Let these systems re-enroll" window at its next renewal check, but
   only from an image built by 0.7.1. On a stateful node that's already
   installed, update `/usr/libexec/mprov/mprov_enroll.py` instead.

The other node-side fixes come from the mPCC itself, so nodes get them the
next time they boot, without a rebuild.

## Upgrading to 0.7.2

0.7.2 is an mPCC-only bug-fix release; the job server stays at `v0.7.1`.
Deleting a system, NIC or BMC, or changing a switch, now updates the DHCP
and DNS config, and saving a host only regenerates it when something that
affects it changed. The
[mPCC release notes](https://github.com/mprov-ng/mprov_control_center/releases/tag/v0.7.2)
have the details. If you're coming from before 0.7.1, do "Upgrading to
0.7.1" above first.

Nothing in `.env` has to change, and nodes don't need their images rebuilt.
Upgrade as in "Image versions" above.

## Upgrading to 0.7.3

0.7.3 is another mPCC-only bug-fix release; the job server stays at
`v0.7.1`. Paging through the Systems list (and the other admin lists with
live search) no longer jumps back to page 1. The
[mPCC release notes](https://github.com/mprov-ng/mprov_control_center/releases/tag/v0.7.3)
have the details.

Nothing in `.env` has to change, and nodes don't need their images rebuilt.
Upgrade as in "Image versions" above, then hard-refresh (Ctrl+Shift+R) an
admin list page so the browser picks up the new script.

## The admin password

On its first start against an empty database, the mPCC creates the `admin`
superuser. Unless you set a password, it generates a random one, prints it
once to the container log and saves it in `/etc/mprov/superuser_password` on
the `mprov-conf` volume. To read it:

```
docker compose exec mpcc cat /etc/mprov/superuser_password
```

or find it in the log, on the line after "Generated a random superuser
password":

```
docker compose logs mpcc | grep -A1 'Generated a random superuser password'
```

To choose the password yourself, set `MPROV_SUPERUSER_PASSWORD` in `.env`
before the first start (it's commented out in `stack.env.dist`):

- It's only read while the database has no superuser. Once `admin` exists,
  setting or changing it does nothing; change the password in the mPCC
  instead.
- Don't put a `$` in it, or write it as `$$`: docker compose expands `$` in
  `.env`, so `pa$word` would become `pa`.
- `.env` is loaded into all three containers, so the password is in the
  environment of `db` and `jobserver` too. Leaving it unset avoids that.

If the database is emptied but the `mprov-conf` volume is kept, the next
first start reuses the saved password.

## Backups

The mPCC writes backups to `/var/lib/mprov/backups`, which is the
`mpcc-backups` volume in `docker-compose.yml`, so they survive the container
being recreated — which is what an upgrade does
([mprov-ng/mprov_control_center#457](https://github.com/mprov-ng/mprov_control_center/issues/457)).
To take one and copy it out to the host:

```
docker compose exec mpcc bash -c 'cd /var/www/mprov_control_center && . bin/activate && \
  python manage.py mprov_backup'
docker compose cp mpcc:/var/lib/mprov/backups ./mprov-backups
```

Keep copying backups off the host as well: a named volume lives on the same
machine as the stack, so it doesn't survive losing that machine.

`mprov_restore` loads a backup back in. Backups hold users, API keys and
systems with their BMC credentials, so treat the copies as secrets — the mPCC
refuses to write them under `media/` or `static/`, which the web server
serves.

The mPCC image creates `/var/lib/mprov/backups` as mode `700`. If you created
the `mpcc-backups` volume with an mPCC older than 0.6.3, where that directory
didn't exist yet, Docker made it `755` instead and the mode isn't re-applied
when you upgrade. Tighten it once if so:

```
docker compose exec mpcc chmod 700 /var/lib/mprov/backups
```

### The mPCC's CA and HTTPS certificate

From mPCC 0.7.0 the mPCC makes its own certificate authority (CA) and an
HTTPS certificate signed by it at first start, in `/var/lib/mprov/tls` — the
`mpcc-tls` volume in `docker-compose.yml`
([mprov-ng/mprov_control_center#381](https://github.com/mprov-ng/mprov_control_center/issues/381)).
The volume keeps them across an upgrade; with an older mPCC it stays empty.

`mprov_backup` doesn't cover this directory. It saves the records of the
credentials the CA has issued, but not the CA itself (`ca.key`, `ca.crt`).
Once the mPCC issues job server and node credentials, losing the volume means
a new CA, and none of those credentials work any more. So copy it off the host
along with your backups, and treat the copy as a secret — `ca.key` is the CA's
private key:

```
docker compose cp mpcc:/var/lib/mprov/tls ./mprov-tls
```

To serve your own HTTPS certificate instead, mount it into the `mpcc` service
and set `MPROV_TLS_CERT` and `MPROV_TLS_KEY` in `.env` (see `stack.env.dist`).
The CA is still made and still needs keeping.

### The job server's key and certificate

From job server 0.7.0 the job server keeps the key and certificate it
enrolled for in `/var/lib/mprov/tls`: the `jobserver-tls` volume, not the
host's `/var/lib/mprov`. It renews the certificate by itself. There's no
need to back the volume up: if it's lost, enroll the job server again. Make
a new enrollment token, give it to the job server, and restart it:

```
docker compose exec mpcc bash -c 'cd /var/www/mprov_control_center && . bin/activate && \
  python manage.py mprov_enroll_token'
docker compose exec -T jobserver sh -c 'umask 077 && cat > /etc/mprov/enroll_token'
docker compose restart jobserver
```

The second command reads the token from standard input: paste the value
after `token:` from the first command's output, then press Ctrl-D. The token
works once, within 24 hours. Enrolling again revokes the lost certificate,
so its key can't be used even if a copy turns up; it shows as revoked in the
mPCC admin, under Agent Identities → Agent Credentials.

## The job server's config files

The job server's config template (`jobserver.yaml.j2`) and plugin configs
(`plugins/*.yaml`) live in `/etc/mprov`, on the `mprov-conf` volume it
shares with the mPCC.

From job server 0.7.0 it brings them up to date on every start
([mprov-ng/mprov-compose#21](https://github.com/mprov-ng/mprov-compose/issues/21),
[mprov-ng/mprov_jobserver#334](https://github.com/mprov-ng/mprov_jobserver/issues/334)),
so the order the services are started in doesn't matter, and an upgrade
brings the new defaults:

- A missing file is installed, and one an earlier release shipped is
  replaced, as long as you haven't edited it.
- A plugin config you've edited is kept, and the new default is written
  beside it as `<name>.yaml.new` (not loaded) for you to merge. The `.new`
  goes away once your file matches the default again.
- An edited `jobserver.yaml.j2` is replaced, and yours is kept as
  `jobserver.yaml.j2.bak`. Set things through `.env` or
  `plugins/local.yaml` instead of editing the template.
- The plugin code (`*.py`) earlier job server images copied into
  `/etc/mprov/plugins` is removed.
- Nothing else in `/etc/mprov` is touched: not the mPCC's keys and
  passwords, and not files of your own there.

After upgrading, see what the job server changed and look for `.new` and
`.bak` files to deal with:

```
docker compose logs jobserver | grep container-config
docker compose exec jobserver ls /etc/mprov /etc/mprov/plugins
```

It only logs when it changes or keeps something, so a start with nothing to
do prints no `container-config` lines.

A job server older than 0.7.0 only gets these files when Docker first seeds
the empty volume. Start the whole stack with one `up -d`, as above. An
upgrade to an older release keeps the old template, so `MPROV_CA_BUNDLE`
and `MPROV_LOG_LEVEL` do nothing. Don't delete the template to get a new one:
the volume isn't empty any more, so nothing replaces it and the job server
won't start. Copy the new image's template in instead:

```
docker compose pull jobserver
docker run --rm --entrypoint cat ghcr.io/mprov-ng/mprov-jobserver:<new tag> /etc/mprov/jobserver.yaml.j2 > jobserver.yaml.j2.new
docker compose up -d
docker compose cp jobserver.yaml.j2.new jobserver:/etc/mprov/jobserver.yaml.j2
docker compose restart jobserver
```

## Upgrading an existing install off the old `/var/lib/mysql` bind mount

Before this fix ([#15](https://github.com/mprov-ng/mprov-compose/issues/15)),
`db`'s data lived in a bind mount at `/var/lib/mysql` on the host. It's now a
named Docker volume (`db-data`) instead, so the database's own entrypoint can
initialize and own it correctly on a fresh host. If you have an existing
install using the old bind mount, migrate its data into the new volume before
pulling this change:

```
docker compose down
docker volume create mprov-compose_db-data
docker run --rm \
  -v /var/lib/mysql:/from \
  -v mprov-compose_db-data:/to \
  alpine sh -c 'cp -a /from/. /to/'
git pull
docker compose --env-file .env up -d
```

Adjust the volume name (`mprov-compose_db-data`) to match your compose
project name if you don't use the directory-derived default — check with
`docker volume ls`. Once you've confirmed the stack comes up and the data is
intact, the old `/var/lib/mysql` directory on the host is no longer used and
can be removed.
