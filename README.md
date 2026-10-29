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
- docker compose --env-file .env up -d
< go have a sandwich >
- http://ipaddress/admin/ to login as admin (see "The admin password" below)
```

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
