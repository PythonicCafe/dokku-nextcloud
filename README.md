# Nextcloud 22 on Dokku

With recent versions of Nextcloud, it became super easy to run it under
[Dokku](https://dokku.com/) using [Nextcloud Docker
image](https://hub.docker.com/_/nextcloud).

## Requirements

You'll need access a server where you have Dokku installed with
[dokku-postgres](https://github.com/dokku/dokku-postgres) and
[dokku-letsencrypt](https://github.com/dokku/dokku-letsencrypt) plugins
enabled.

## Installtion

Connect to your Dokku server and set the following environment variables with
the values for your deployment:

```sh
# Execute on SERVER

APP_NAME="nextcloud"
PG_SERVICE_NAME="pg_${APP_NAME}"
APP_DOMAIN="nextcloud.example.com"
ADMIN_EMAIL="admin@example.com"
DATA_PATH="/var/lib/dokku/data/storage/${APP_NAME}"
```

Now, execute the following commands:

```sh
# Execute on SERVER

dokku apps:create $APP_NAME
dokku postgres:create $PG_SERVICE_NAME
dokku postgres:link $PG_SERVICE_NAME $APP_NAME
```

Then, from your machine, add a `dokku` remote pointing to your server/app and
do the first push:

```sh
# Execute on CLIENT (env vars set on server are not available here, so replace
# <...> with the correct values)

git clone <this-repo-url>
cd dokku-nextcloud
git remote add dokku dokku@<your-server>:<your-app-name>
git push dokku main
```

[comment]: <> (TODO: we may change to the docker image deployment instead of git repository)

Now back to your dokku server to finish the configuration!

```sh
# Execute on SERVER

dokku domains:add $APP_NAME $APP_DOMAIN
dokku config:set $APP_NAME DOKKU_LETSENCRYPT_EMAIL=$ADMIN_EMAIL
dokku letsencrypt:enable $APP_NAME

for directory in config custom_apps data; do
	mkdir -p "$DATA_PATH/$directory"
	chown www-data:www-data "$DATA_PATH/$directory"
	dokku storage:mount $APP_NAME "$DATA_PATH/$directory:/var/www/html/$directory"
done

dokku ps:restart $APP_NAME
```

You're nearly done! Now:
- Visit `<your domain>` to finish configuration.
- Set up an admin account.
- Click on "Storage & database".
- Leave the data folder unchanged (`/var/www/html/data`)
- For the database, **select Postgres**; you'll find the database credentials
  by running `dokku config $PG_SERVICE_NAME` and looking for DATABASE_URL:
  it'll look like `postgres://USER:PASSWORD@HOST:PORT/DBNAME`, just fill-in the
  fields on the webpage. For instance, with
  `postgres://postgres:ae9e02101f9977e1fabb19f09605e486@dokku-postgres-nextcloud:5432/nextcloud`:
  - Database user: `postgres`
  - Database password: `ae9e02101f9977e1fabb19f09605e486`
  - Database name: `nextcloud`
  - Database host: `dokku-postgres-nextcloud:5432`
- Click on **finish setup**.


## Tweaks

### Cache

You can make pages load faster (and save some CPU cycles) by adding a cache
layer to Nextcloud. It can be done by running [Redis](https://redis.io/) via
[dokku-redis](https://github.com/dokku/dokku-redis).

[comment]: <> (TODO: add commands to install redis plugin, create a Redis instance, link to the app and configure it on Nextcloud)

### Upload size

If you're planning to use desktop client, because you're running with Dokku as
a reverse proxy, you'll need to change your `config.php` file under
`$CONFIG_PATH` to add this:

```
'overwriteprotocol' => 'https'
```

Make sure the value for `'overwrite.cli.url'` starts with https.

You'll also need to allow large uploads to your server:

```sh
mkdir /home/dokku/$APP_NAME/nginx.conf.d/
echo 'client_max_body_size 50000m;' > /home/dokku/$APP_NAME/nginx.conf.d/upload.conf
echo 'proxy_read_timeout 600s;' >> /home/dokku/$APP_NAME/nginx.conf.d/upload.conf
chown dokku:dokku /home/dokku/$APP_NAME/nginx.conf.d/upload.conf
service nginx reload
```

### Cron

For some tasks you'll need to install a crontab. For this to work, you must
create an `app.json` file inside this repository, commit it and push to dokku's
app repository on your server. Check [Dokku docs on how to configure a cron
job](https://dokku.com/docs/processes/scheduled-cron-tasks/#dokku-managed-cron).

[comment]: <> (TODO: add sample `app.json`)
[comment]: <> (TODO: may use <https://github.com/crisward/dokku-require> and provide a sample app.json)

### Executing `occ` command

```sh
# Execute on the SERVER

container_id=$(docker ps | grep dokku/${APP_NAME}:latest | awk '{print $1}')
docker exec -it --user www-data $container_id php occ ...
```

## Uninstalling

If you're unhappy with your setup, this'll remove everything *forever*:

```sh
dokku apps:destroy $APP_NAME
dokku postgres:destroy $APP_NAME
rm -rf "$DATA_PATH" "$CONFIG_PATH"
```

### Possible problems

#### Invalid private key

If the users are getting the annoying message "Invalid private key for
encryption app", update your private key password in your personal settings to
recover access to your encrypted files", look at [this issue
comment](https://github.com/nextcloud/server/issues/8546#issuecomment-371795233).


#### Could not establish a connection

If the users are getting the message "Could not establish a connection with at
least one participant" during videoconferences, then a TURN server might be
needed for your scenario, check more details at [Nextcloud TURN
docs](https://nextcloud-talk.readthedocs.io/en/latest/TURN.html).
