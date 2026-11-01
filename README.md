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
