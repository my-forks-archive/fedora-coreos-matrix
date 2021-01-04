# Fedora CoreOS Config to host a Matrix homeserver

Example Fedora CoreOS config to host a Matrix homeserver. This will setup:
  * nginx with let's encrypt for HTTPS support
  * synapse with postgresql and elements-web

For this setup, you need a domain name and two sub-domains:
  * example.tld
  * matrix.example.tld
  * chat.example.tld

For Let's Encrypt support, those domains must be configured beforehand to
resolve to the IP address that will be assigned to your server. If you do not
know what IP address will be assigned to your server in advance, you might want
to use another ACME challenge method to get Let's Encrypt certificates (see
[DNS Plugins][plugins]).

If you already have certificates from Let's Encrypt or another provider, see
the [Alternative with certificates](#alternative-with-certificates) section.


## How to use

To generate the Ignition configs, you need `make` and `fcct`.

Then, you need to provide values for each variable in the secrets file:

```
$ cp secrets.example secrets
$ ${EDITOR} secrets
# Fill in values not marked as generated by synapse
```

### Configuring Synapse

The synapse configuration requires to setup a few secrets, you can generate these secret using
the following command :

```
$ source secrets
$ mkdir generated
$ podman run -it --rm -v $PWD/generated:/data:z \
      -e SYNAPSE_SERVER_NAME="${DOMAIN_NAME}" \
      -e SYNAPSE_REPORT_STATS=yes \
      docker.io/matrixdotorg/synapse:latest \
      generate
```
This command will generate 3 files

- `generated/homeserver.yaml`
- `generated/my.matrix.host.log.config`
- `generated/my.matrix.host.signing.key`

#### Configuration

A template version of the generated `homeserver.yaml` is included in
`template/synapse/homeserver.yaml`.

First we want to replace the secrets with values that were generated for you by
synapse.

In the `secrets` file, edit the following variables:

- `SYNAPSE_REGISTRATION_SHARED_SECRET`, with the content of
  `registration_shared_secret` in `homeserver.yaml`
- `SYNAPSE_MACAROON_SECRET_KEY`, with the content of `macaroon_secret_key` in
  `homeserver.yaml`
- `SYNAPSE_FORM_SECRET`, with the content of `form_secret` in `homeserver.yaml`
- `SYNAPSE_SIGNING_KEY`, with the content of `my.matrix.host.signing.key`

```
SSH_PUBKEY="ssh-rsa AAAA..."
POSTGRES_PASSWORD=a_passpharse_for_my_database
DOMAIN_NAME=my.matrix.domain
EMAIL=root@example.com
SYNAPSE_REGISTRATION_SHARED_SECRET=a_very_long_string_generated_by_synapse
SYNAPSE_MACAROON_SECRET_KEY=a_very_long_string_generated_by_synapse
SYNAPSE_FORM_SECRET=a_very_long_string_generated_by_synapse
SYNAPSE_SIGNING_KEY=a_key_generated_by_synapse
```

If you wish to change other synapse settings you can edit directly
`template/synapse/homeserver.yaml` and `template/synapse/synapse.log.config` to
change the logging configuration.

### System and container updates

By default, Fedora CoreOS systems are updated automatically to the latest
released update. This makes sure that the system is always on top of security
issues (and updated with the latest features) wthout any user interaction
needed. The containers, as defined in the systemd units in the config, are
updated on each service startup. They will thus be updated at least once after
each system update as this will trigger a reboot approximately every two week.

To maximise availability, you can set an [update strategy][updates] in
Zincati's configuration to only allow reboots for updates during certain
periods of time.  For example, one might want to only allow reboots on week
days, between 2 AM and 4 AM UTC, which is a timeframe where reboots should have
the least user impact on the service. Make sure to pick the correct time for
your timezone as Fedora CoreOS uses the UTC timezone by default.

See this example config that you can append to `config.yaml`:

```
[updates]
strategy = "periodic"

[[updates.periodic.window]]
days = [ "Mon", "Tue", "Wed", "Thu", "Fri" ]
start_time = "02:00"
length_minutes = 120
```

## Generate the ignition configuration

Finally, you can generate the final Ignition config with:

```
$ make
```

You are now ready to deploy your Fedora CoreOS Matrix home server.

## Deploying

See the [Fedora CoreOS docs][deploy] for instructions on how to use this
Ignition config to deploy a Fedora CoreOS instance on your prefered platform.

## Registering new users

Registration is disabled by default to avoid issues. You can login to the
server and add accounts with:

```
$ sudo podman run --rm --tty --interactive \
      --pod=matrix \
      -v /var/srv/matrix/synapse:/data:z,ro \
      --entrypoint register_new_matrix_user \
      docker.io/matrixdotorg/synapse:latest \
      -c /data/homeserver.yaml http://127.0.0.1:8008
```

## PostgreSQL major version updates

Major PostgreSQL version updates require manual intervention to dump the
database with the current version and then import it in the new version. We
thus can not use the `latest` tag for this container image and manual
intervention will be required approximately once a year to update the PostreSQL
container version.

See this example to dump the current database and import it in the new version:

```
# Stop Synapse server to ensure no-one is writing to the database
$ systemctl stop synapse

# Dump the database
$ mkdir /var/srv/postgres.dump
$ cat /etc/postgresql_synapse
$ podman run --read-only --pod=matrix --rm \
      --tty --interactive \
      -v /var/srv/matrix/postgres.dump:/var/data:z \
      docker.io/library/postgres:13 \
      pg_dump --file=/var/data/dump.sql --format=c --username=synapse \
      --password --host=localhost synapse

# Stop the PostgreSQL container
$ systemctl stop postgres

# Edit the PostgreSQL unit to update the container version
$ vi /etc/systemd/system/postgres.service

# Start the new PostgreSQL container
$ systemctl start postgres

# Import the existing database
$ podman run --read-only --pod=matrix --rm \
      --tty --interactive \
      -v /var/srv/matrix/postgres.dump:/var/data:ro,z \
      docker.io/library/postgres:13 \
      pg_restore --username=synapse --password --host=localhost \
      --dbname=synapse /var/data/dump.sql

# Start Synapse again
$ systemctl start synapse

# Cleanup once everything is confirmed working
$ rm -rf /var/srv/matrix/postgres.dump
```

[deploy]: https://docs.fedoraproject.org/en-US/fedora-coreos/getting-started/
[plugins]: https://certbot.eff.org/docs/using.html#dns-plugins
[updates]: https://coreos.github.io/zincati/usage/updates-strategy/#periodic-strategy
