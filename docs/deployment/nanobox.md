# Deploying with Nanobox

The main goal for this guide is to show you how to deploy a Phoenix application to production using Nanobox.

## What you'll need

All you need is an existing Phoenix application. If you don't have one handy, we've created two Phoenix "quickstarts" that you can use to follow along:

- [nanobox-phoenix](https://github.com/nanobox-quickstarts/nanobox-phoenix) - A vanilla Phoenix app with a PostgreSQL database ready to go.
- [nanobox-phoenix-example](https://github.com/nanobox-quickstarts/nanobox-phoenix-example) - A Phoenix "Todo" app with a PostgreSQL database and basic CRUD operations.

You can also reference our [from scratch](https://guides.nanobox.io/elixir/phoenix/from-scratch/) guide, or Phoenix's own [Up and Running guide](http://www.phoenixframework.org/docs/up-and-running) to create a new application.

### The first time
If this is your first time using Nanobox there are a few things you'll want to have before you get started.

First, you'll need to [create a Nanobox account](https://dashboard.nanobox.io/users/register) and then [download and install Nanobox](https://dashboard.nanobox.io/download).

Nanobox is ***free*** *for open source and personal projects*, with [flexible plans](https://nanobox.io/pricing/) available for when your app is ready to scale.

Next, you'll want to make sure you have an account with the cloud provider of your choice (AWS, DigitalOcean, etc.).

Finally, you'll need to [link your cloud host account with your Nanobox account](https://dashboard.nanobox.io/users/provider_accounts).

## Steps

There are four steps when deploying an app to production using Nanobox:

- Create a new app on [Nanobox](https://dashboard.nanobox.io)
- Configure your project
- Stage your application
- Link and Deploy to production

## Create a Nanobox Application

Create a [new application](https://dashboard.nanobox.io/apps/new) from the Nanobox dashboard.

During the app creation process, you'll be prompted to name your application, and select a host and region where you what your application to live.

> Note: Remember the name of your application as you'll be using it later to link your local codebase before you deploy.

## Configure your project

> Note: All nanobox commands are run from the root of your application

### The boxfile.yml

Nanobox uses a simple config file called a `boxfile.yml` when provisioning development and production environments.

To prepare your project for deployment with Nanobox, create a `boxfile.yml` at the root of your project with the following:

```yaml
run.config:

  # elixir runtime
  engine: elixir

  # ensure inotify exists for hot-code reloading
  dev_packages:
    - nodejs
    - inotify-tools

  # cache node_modules
  cache_dirs:
    - node_modules

  # add node_module bins to the $PATH
  extra_path_dirs:
    - node_modules/.bin

  # enable the filesystem watcher
  fs_watch: true

# deployment options
deploy.config:

  # generate the static assets digest
  extra_steps:
    - mix phoenix.digest

  # migrate the database just before the new process comes online
  before_live:
    web.main:
      - mix ecto.create --quiet
      - mix ecto.migrate

# add a postgres data component
data.db:
  image: nanobox/postgresql

# add a web component with a start command
web.main:
  start: node-start mix phoenix.server
```

> Note: At this point, if you wanted to, you could use `nanobox run` to have Nanobox provision you a local development environment for your application.

Sample `nanobox run` output:
```bash
$ nanobox run
Preparing environment :
  * Mounting codebase

Root privileges are required to modify network shares. Your password may be requested...

  ✓ Mounting codebase

--------------------------------------------------------------------------------
+ HEADS UP:
+ This is the first build for this project and will take longer than usual.
+ Future builds will pull from the cache and will be much faster.
--------------------------------------------------------------------------------

Building runtime :
  ✓ Starting docker container
  ✓ Preparing environment for build
  ✓ Gathering requirements
  ✓ Mounting cache_dirs
  ✓ Installing binaries and runtimes
  ✓ Packaging build

nanobox-phoenix (local) :
  ✓ Reserving IPs

Syncing data components :
  Removing old :
    ✓ Skipping (up-to-date)
  Launching new :
    data.db :
      ✓ Reserve IP
      ✓ Starting docker container
      ✓ Gathering requirements
      ✓ Configuring services

Building dev environment :
  ✓ Starting docker container
  ✓ Configuring

                                   **
                                ********
                             ***************
                          *********************
                            *****************
                          ::    *********    ::
                             ::    ***    ::
                           ++   :::   :::   ++
                              ++   :::   ++
                                 ++   ++
                                    +
                    _  _ ____ _  _ ____ ___  ____ _  _
                    |\ | |__| |\ | |  | |__) |  |  \/
                    | \| |  | | \| |__| |__) |__| _/\_

--------------------------------------------------------------------------------
+ You are in a Linux container
+ Your local source code has been mounted into the container
+ Changes to your code in either the container or desktop will be mirrored
+ If you run a server, access it at >> 172.19.0.21
--------------------------------------------------------------------------------

/app $ mix phoenix.server

```

### Environment Variables

Nanobox makes use of [environment variables](https://guides.nanobox.io/elixir/phoenix/production-evars/) (evars) as a way of keeping secret information, well... secret. To use these evars you'll need to make a few modifications to some of your app's config files.

First, make sure the secret key is loaded from the Nanobox evars rather than the `config/prod.secret.exs` by adding it to the `config/prod.exs`:

```elixir
config :nanobox_phoenix, PhoenixCrud.Endpoint,
  http: [port: 8080],
  url: [host: "example.com", port: 80],
  cache_static_manifest: "priv/static/manifest.json",
  secret_key_base: System.get_env("SECRET_KEY_BASE")
```

Next, add a production database configuration to `config/prod.exs`:

```elixir
# Configure your database
config :nanobox_phoenix, PhoenixCrud.Repo,
  adapter: Ecto.Adapters.Postgres,
  username: System.get_env("DATA_DB_USER"),
  password: System.get_env("DATA_DB_PASS"),
  hostname: System.get_env("DATA_DB_HOST"),
  database: "nanobox_phoenix",
  pool_size: 20
```

Comment out the following line in your `/config/prod.exs` since config is now handled via evars:

```elixir
import_config "prod.secret.exs"
```

## Create Environment Variables in Nanobox

Any environment variables that need to be added for both local development and staging can be done with the `evar add` command:

```bash
$ nanobox evar add local KEY=VALUE
$ nanobox evar add dry-run KEY=VALUE
```

Adding evars to production is done through the [Nanobox dashboard](https://dashboard.nanobox.io/config/evars).

Nanobox generates the `DATA_DB_USER`, `DATA_DB_PASS`, and `DATA_DB_HOST` for you, but you'll need to add the `SECRET_KEY_BASE` (and any others you might have).

Generate a secret key, and add it to your evars on Nanobox:

```bash
$ mix phoenix.gen.secret
Cg8bzCBBn99SsEvFazw9Z/UMF1uAvS4Ke1OT1Rb+Ow9RvNWSXHwuXjOQf/Xw6jno
```

> Note: Your random string will be different from the one above.

## Stage your application

Nanobox allows you to stage a production deploy, locally, with the `dry-run` command:

```bash
$ nanobox deploy dry-run
```

Sample `nanobox deploy dry-run` output:

```bash
$ nanobox deploy dry-run
Compiling application :
  ✓ Starting docker container
  ✓ Preparing environment for compile
  ✓ Compiling code

nanobox-phoenix (dry-run) :
  ✓ Reserving IPs

Syncing data components :
  Removing old :
    ✓ Skipping (up-to-date)
  Launching new :
    data.db :
      ✓ Reserve IP
      ✓ Starting docker container
      ✓ Gathering requirements
      ✓ Configuring services

Starting components :
  Logger :
    ✓ Reserve IP
    ✓ Starting docker container
    ✓ Gathering requirements
    ✓ Configuring services
  Message Bus :
    ✓ Reserve IP
    ✓ Starting docker container
    ✓ Gathering requirements
    ✓ Configuring services
  Router :
    ✓ Reserve IP
    ✓ Starting docker container
    ✓ Gathering requirements
    ✓ Configuring services
  Storage :
    ✓ Reserve IP
    ✓ Starting docker container
    ✓ Gathering requirements
    ✓ Configuring services

Deploying app :
  ✓ Starting docker container
  ✓ Uploading

Syncing code components :
  Removing old :
  Starting new :
    web.main :
      ✓ Starting docker container
      ✓ Fetching build from warehouse
      ✓ Starting services

Finalizing deploy :
  ✓ Running before_live hooks
  ✓ Updating router
  ✓ Running after_live hooks

--------------------------------------------------------------------------------
+ Your app is running in simulated production environment
+ Access your app at >> 172.19.0.23
--------------------------------------------------------------------------------

Connected to streaming logs:
ctrl + c to quit
------------------------------------------------
waiting for output...
```

You can either visit your application at the IP given from the `dry-run` command, or generate a dns alias with the `dns add` command:

```bash
$ nanobox dns add dry-run nanobox-phoenix.stage
```

Now try visiting your app at [nanobox-phoenix.stage](nanobox-phoenix.stage/)

## Link and Deploy to production

Before deploying your application, you'll first need to link the local codebase to the app you created on Nanobox.

> Note: The first time you try and link the application you'll be asked to login using your Nanobox credentials.

```bash
$ nanobox remote add [app-name]
$ nanobox deploy
```

Sample `nanobox deploy` output:
```bash
$ nanobox deploy
Compiling application :
  ✓ Starting docker container
  ✓ Preparing environment for compile
  ✓ Compiling code

--------------------------------------------------------------------------------
+ HEADS UP:
+ This is the first deploy to this app and the upload takes longer than usual.
+ Future deploys only sync the differences and will be much faster.
--------------------------------------------------------------------------------

Deploying app :
  ✓ Starting docker container
  ✓ Uploading

✓ Success, this deploy is on the way!
  Check your dashboard for progress.

```

Nanobox will build your application locally and send it up to your Nanobox app. It will then use the `boxfile.yml` to provision and configure your applications environment and start your app.

That's it!

## Useful Nanobox Commands

## Troubleshooting
