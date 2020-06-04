---
description: The best local development environment option for platform.sh, the fastest way to build modern web apps.
---

# Platform.sh **(experimental)**

[Platform.sh](https://platform.sh/) is the end-to-end web platform for agile teams. with it you can build, evolve, and scale your website fleet—with zero infrastructure management investment. Get hosting, CI/CD, automated updates, global 24x7 support. And much more.

This is currently a _very experimental_ integration that has the following _serious caveats_:

* This should be considered at an `alpha` level of readiness or below
* This has _only_ been tested for sites built on top of the [platform.sh Drupal 8 Template](https://github.com/platformsh-templates/drupal8)
* This currently _only_ supports platform.sh's `php` application container
* this currently _only_ supports platform.sh's `memcached`, `mongodb`, `mariadb`, `mysql`, `postgresql`, `redis` and `solr` service containers
* It's not yet clear how much customization to your project is currently supported

However, if you'd like to try it out and give your feedback on what worked and what didn't then please continue.

You can report any issues or feedback [over here](https://github.com/lando/lando/issues/new/choose).

[[toc]]

## Getting Started

:::warning EXPERIMENTAL FEATURE
To access this feature you will need:

  * [Lando 3.0.5](./../help/2020-changelog.md) or higher or Lando [installed from source](./../basics/installation.md#from-source).
  * [Experimental mode](./experimental.md) turned on
:::

Before you get started with this recipe we assume that you have:

1. [Installed Lando](./../basics/installation.md) and gotten familiar with [its basics](./../basics/)
2. [Initialized](./../basics/init.md) a [Landofile](./../config/lando.md) for your codebase for use with this recipe
3. Read about the various [services](./../config/services.md), [tooling](./../config/tooling.md), [events](./../config/events.md) and [routing](./../config/proxy.md) Lando offers.

However, because you are a developer and developers never ever [RTFM](https://en.wikipedia.org/wiki/RTFM), you can also run the following commands to try out this recipe on one of your platform.sh sites.

```bash
# Go through interactive prompts to get your site from platformsh
lando init --source platformsh

# OR do it non-interactively
# NOTE: You will want to make sure you set $PLATFORMSH_CLI_TOKEN
# and $PLATFORMSH_SITE_NAME to values that make sense for you
lando init \
  --source platformsh \
  --platformsh-auth "$PLATFORMSH_CLI_TOKEN" \
  --platformsh-site "$PLATFORMSH_SITE_NAME"

# Start it up
lando start

# Import any relevant relationships or mounts
# NOTE: You will likely need to change the below to specify
# relationships and mounts that make sense for your application
# See further below for more information about lando pull
lando pull -r database -m web/sites/default/files

# List information about this app.
lando info
```

## Configuration

While Lando [recipes](./../config/recipes.md) set sane defaults so they work out of the box, they are also [configurable](./../config/recipes.md#config).

Here are the configuration options, set to the default values, for this recipe's [Landofile](./../config/lando.md). If you are unsure about where this goes or what this means we *highly recommend* scanning the [recipes documentation](./../config/recipes.md) to get a good handle on how the magicks work.

```yaml
recipe: platformsh
config:
  id: YOURSITEID
```

You will immediately notice that the default `platformsh` recipe Landofile does not contain much. This is because Lando uses the exact same images and configuration mechanisms locally as platform.sh does in production.

This means that instead of modifying your Landofile to add, edit or remove the services, dependencies, build steps, etc you need to run your application you will want to modify your platform.sh configuration according to their documentation and then do the usual `lando rebuild` for those changes to be applied.

Of course, since this is still a Lando recipe you can continue to [extend and override](./../config/recipes.md#extending-and-overriding-recipes) your Landofile in the usual way for any additional power you require locally.

Here are some details on how Lando interprets the various platform.sh configuration files:

### routes.yaml

Lando will load your [routes.yaml](https://docs.platform.sh/configuration/routes.html) and use for its own [proxy](./proxy.md) configuration.

```yaml
# routes.yaml

"https://{default}/":
  type: upstream
  upstream: "app:http"
  cache:
    enabled: true
    # Base the cache on the session cookie and custom Drupal cookies. Ignore all other cookies.
    cookies: ['/^SS?ESS/', '/^Drupal.visitor/']

"https://www.{default}/":
  type: redirect
  to: "https://{default}/"
```

The above `routes` configuration example will produce the following Lando pretty proxy URLs, assuming `{default}` resolves to `my-app.lndo.site`.

```bash
http://my-app.lndo.site
https://my-app.lndo.site
http://www.my-app.lndo.site
https://www.my-app.lndo.site
```

### services.yaml

Lando will load your [services.yaml](https://docs.platform.sh/configuration/services.html) and spin up _exactly_ the same things there as you have running on your platform.sh site, including any advanced configuration options you may have specified for each like `schemas`, `endpoints`, `extensions`, `properties`, etc.

This means that Lando knows how to handle more complex configuration such as in the below example:

```yaml
# services.yaml

db:
  type: mariadb:10.4
  disk: 2048
  configuration:
    schemas:
      - main
      - legacy
    endpoints:
      admin:
        default_schema: main
        privileges:
          main: admin
          legacy: admin
db2:
  type: postgresql:12
  disk: 1025
  configuration:
    extensions:
      - pg_trgm
      - hstore
```

We currently only support the below services and we _highly recommend_ you consult the platform.sh docs for how to properly configure each.

* [Memcached](https://docs.platform.sh/configuration/services/memcached.html)
* [MongoDB](https://docs.platform.sh/configuration/services/mongodb.html)
* [MariaDB/MySQL](https://docs.platform.sh/configuration/services/mysql.html)
* [PostgreSQL](https://docs.platform.sh/configuration/services/mysql.html)
* [Redis](https://docs.platform.sh/configuration/services/redis.html)
* [Solr](https://docs.platform.sh/configuration/services/solr.html)

Also note that you will need to run a `lando rebuild` for configuration changes to manifest in the same way you normally would for config changes to your Landofile.

### .platform.app.yaml

Lando will load your [.platform.app.yaml](https://docs.platform.sh/configuration/routes.html) and spin up _exactly_ the same things there as you have running on your platform.sh site. This means that similarly to platform.sh Lando will also:

* Install any dependencies specificed in the `build.flavor` or `dependencies` keys
* Run any `build` or `deploy` hooks
* Set up needed `relationships`, `variables`, `web` config, `cron` tasks, etc.

We currently only support the below langauges and we _highly recommend_ you consult the platform.sh docs for how to properly configure each.

* [PHP](https://docs.platform.sh/languages/php.html)

Also note that you will need to run a `lando rebuild` for configuration changes to manifest in the same way you normally would for config changes to your Landofile.

### Other considerations

#### Multiapp

Multiapp configurations _should theoretically_ work but are not currently supported.

#### Environment variables

Application containers running on Lando will also set up the same [platform.sh provided environment variables](https://docs.platform.sh/development/variables.html#platformsh-provided-variables) so any service connection configuration, like connecting your Drupal site to `mysql` or `redis`, you use on platform.sh with these variables _should_ also automatically work on Lando.

## Platform CLI

Every application container will contain the [Platform CLI](https://docs.platform.sh/development/cli.html); automatically authenticated for use with the account and project you selected during `lando init`.

```bash
# Who am i?
lando platform auth:info

# Tell me about my project
lando platform project:info
```

If you find yourself unauthenticated for whatever reason. You should try the following:

```bash
# Reauthenticate using already pulled down code
lando init --source cwd --recipe platformsh

# Rebuild your lando app
lando rebuild -y
```

## Application Tooling

Lando will also setup useful [tooling commands](./tooling.md) based on the `type` of your application container.

These can be used to both relevant tooling and utilities that exist _inside_ the application container. Here are the defaults we provide for the `php` application container.

```bash
lando composer    Runs composer commands
lando php         Runs php commands
```

### Usage

```bash
# Install some composer things
lando composer require drush/drush

# Run a php script
lando php myscript.php
```

Of course the user can also `lando ssh` and work directly inside _any_ of the containers Lando spins up for your app.

```bash
# Attach to the closest applicaiton container
lando ssh

# Attach to the db service
lando ssh -s db
```

Note that Lando will surface commands for the _closest application_ it finds. Generally, this will be the `.platform.app.yaml` located in your project root but if you've `cd multiappsubdir` then it would use that instead.


### Adding additional tooling

While Lando will set up tooling routes for the _obvious_ utilities for each application `type` it tries to not overwhelm the user with _all the commands_ by providing a minimally useful set. It does this because it is very easy to specify more tooling commands in your Landofile.

```yaml
tooling:
  # Here are some utilities that should exist in every application
  # container
  node:
    service: app
  npm:
    service: app
  ruby:
    service: app

  # And some utilities we installed in the `build.flavor`
  # or `dependencies` key
  grunt:
    service: app
  sass:
    service: app
  drush:
    service: app

```

Note that the `service` should match the `name` of your application in the associated `.platform.app.yaml`. Very often this is just `app`.

Now run `lando` again and see that extra commands!

```bash
lando composer      Runs composer commands
lando drush         Runs drush commands
lando grunt         Runs grunt commands
lando node          Runs node commands
lando npm           Runs npm commands
lando php           Runs php commands
lando ruby          Runs ruby commands
lando sass          Runs sass commands
```

```bash
lando drush cr
lando npm install
lando grunt compile:things
lando ruby -v
lando node myscript.js
```

If you are not sure whether something exists inside your application container or not you can easily test using the `-c` option provided by l`lando ssh`

```bash
# Does yarn exist?
lando ssh -c "yarn"
```

Also note that Lando tooling is hyper-powerful so you might want to [check out](./tooling.md) some of its more advanced features.


## Accessing relationships

Lando will also set up tooling commands so you can directly access the `relationships` specified in your `.platform.app.yaml`.

These are contextual so they will connect via the tool that makes the most sense eg `mysql` for `mariadb` and `redis-cli` for `redis`.

As an example say you have the following relationships in your `.platform.app.yaml`.

```yaml
relationships:
  database: 'db:mysql'
  redis: 'cache:redis'
```

Then you'd expect to see the following commands and usage:

```bash
lando database  Connects to the database relationship
lando redis     Connects to the database relationship
```

```bash
# Drop into the mysql shell using the database relationship creds
lando database

# Drop into the redis-cli shell using the redis relationship creds
lando redis
```

Note that some services eg `solr` provide `web` based interfaces. In these cases Lando will provide a `localhost` address you can use to access that interface.

## External access

If you would instead like to connect to your database, or some other service, from your host using a GUI client like SequelPro, instead of via the Lando CLI you can run [`lando info`](./../cli/info.md) and use the `external_connection` information and any relevant `creds` for the service you want to connect to.

Here is example connection info for a multi-endpoint `mariadb` service called `db` below:

```bash
lando info --service db --format default

  { service: 'db',
    urls: [],
    type: 'platformsh-mariadb',
    healthy: true,
    creds:
     [ { internal_hostname: 'database2.internal',
         password: '3ac01938c66f0ce06304a6357da17c34',
         path: 'main',
         port: 3306,
         user: 'admin' },
       { internal_hostname: 'reports.internal',
         password: 'd0c99f580a0d646d62904568573f5012',
         port: 3306,
         user: 'reporter' },
       { internal_hostname: 'imports.internal',
         password: 'a6bf5826a81f7e9a3fa42baa790207ef',
         path: 'legacy',
         port: 3306,
         user: 'importer' } ],
    internal_connection: { host: 'db', port: '3306' },
    external_connection: { host: '127.0.0.1', port: '32915' },
    config: {},
    version: '10.4',
    meUser: 'app',
    hasCerts: false,
    hostnames: [ 'db.landod8.internal' ] },
```

Note that you _must_ have a relationship from your app to a given service in order for it to have credentials.

Also note that this is slightly different than the normal output from `lando info` because `platformsh` services work slightly different. While you _can_ use the `internal_connection:host` and `internal_connection:port` for internal connections we recommend you use the `host` and `port` indicated for the relevant `cred` you want to connect to instead.

So if you wanted to connect to the `main` db you would use the following depending on whether you are connecting externally or internally:

**external creds**

```yaml
host: 127.0.0.1
port: 32915
user: admin
password: 3ac01938c66f0ce06304a6357da17c34
database: main
```

**internal creds**

```yaml
host: database2.internal
port: 3306
user: admin
password: 3ac01938c66f0ce06304a6357da17c34
database: main
```

Of course, it is always preferrable to just use `PLATFORM_RELATIONSHIPS` for all your internal connections anyway.

## Pulling relationships and mounts

Lando also provides a _currently rudimentary_ wrapper command called `lando pull` that you can use to import data and download files from your remote platform.sh site.

```bash
lando pull

Pull relationships and/or mounts from platform.sh

Options:
  --help              Shows lando or delegated command help if applicable
  --verbose, -v       Runs with extra verbosity
  --mount, -m         A mount to download
  --relationship, -r  A relationship to import
```

```bash
# Import the remote database relationship and drupal files mount
lando pull -r database -m web/sites/default/files

# Import multiple relationships and mounts
lando pull -r database -r migrate -r readonly -m tmp -m private

# You can also specify a target for a given mount using -m SOURCE:TARGET
lando pull -m tmp:/var/www/tmp -m /private:/somewhere/else
```

## Caveats and known issues

Since this is a currently a pre-alpha level recipe there are a few known issues, and workarounds, to be aware of. We also recommend you consult GitHub for other [platform.sh tagged issues](https://github.com/lando/lando/issues?q=is%3Aopen+is%3Aissue+label%3Aplatformsh
).

We also _highly encourage_ you to [post an issue](https://github.com/lando/lando/issues/new/choose) if you see a problem that doesn't already have an issue.

### `$HOME` considerations

Platform.sh sets `$HOME` to `/app` by default. This makes sense in a read-only hosting context but is problematic for local development since this is also where your `git` repository lives and you probably don't want to accidentally commit your `$HOME/.composer` cache into your repo.

Lando changes this behavior and sets `$HOME` to its own default of `/var/www` for most _user initiated_ commands and automatic build steps.

It also will override any `PLATFORM_VARIABLES` that should be set differently for local dev. For a concrete example of this platform.sh's Drupal 8 template will set the Drupal `/tmp` directory to `/app/tmp`, Lando will instead set this to `/tmp`.

However, it's _probable_ at this early stage that we have not caught all the places where we need to do both of the above. As a result you probably want to:

#### 1. Look out for caches, configs, or other files that might normally end up in `$HOME`.

Do you due diligence and make sure you `git status` before you `git add`. If you see something that shouldn't be there [let us know](https://github.com/lando/lando/issues/new/choose) and then add it to your `.gitignore` until we have resolved it.


#### 2. Consider LANDO specific configuration

If you notice your application is _not working quite right_ it's possible you need to tweak some of the defaults for your application's configuration so they are set differently on Lando. We recommend you do something like the below snippet.

`settings.local.php`

```php
$platformsh = new \Platformsh\ConfigReader\Config();

if ($config->environment === 'lando') {
  $settings['file_private_path'] = '/tmp';
  $config['system.file']['path']['temporary'] = '/tmp';
}

```

Note that the above is simply meant to be illustrative.

### Local considerations

There are some application settings and configuration that platform.sh will automatically set if your project is based on one of their boilerplates. While most of these settings are fine for local development, some are not. If these settings need to be altered for your site to work as expected locally then Lando will modify them.

For example if your project is based on the [Drupal 8 Template](https://github.com/platformsh-templates/drupal8) then Lando will set the `tmp` directory and set `skip_permissions_hardening` to `TRUE`.

Lando will likely _not_ do this in the future in favor of a better solution but until then you can check out what we set over [here](https://github.com/lando/lando/blob/master/experimental/plugins/lando-platformsh/lib/overrides.js).

### platformsh.agent errors

When you run `lando start` or `lando rebuild` you may experience either Lando hanging or an error being thrown by something called the `platformsh.agent`. We are attempting to track down the causes of some of these failures but they are generally easy to identify and workaround:

```bash
# Check if a container for your app has exited
docker ps -a

# Inspect the cause of the failure
#
# Change app to whatever you named your application
# in your .platform.app.yaml
lando logs -s app

# Try again
# Running lando start again seems to work around the error
lando start
```

### Persistence across rebuilds

We've currently only verified that data will persist across `lando rebuilds` for the MariaDB/MySQL and PostgreSQL services. It _may_ persist on other services but we have not tested this yet so be careful before you `lando rebuild` on other services.

## Development

If you are interested in working on the development of this recipe we recommend you check out:

* The Lando [contrib docs](./../contrib/contributing.md)
* The [Dev Docs](https://github.com/lando/lando/tree/master/experimental/plugins/lando-platformsh) for this recipe

<RelatedGuides tag="Platformsh"/>
