# TRY IT OUT

Let's see how the Phoenix 360 Web Apps methodology can be used in practice by implementing the [White Paper](/WHITE_PAPER.md#an-usage-example) usage example.

## TOC

* [Project context](#project-context)
* [Hosts setup](#hosts-setup)
* [Create a project](#create-a-phoenix-360-web-apps-project)
* [Secrets](#phoenix-360-web-apps-secrets)

[Home](/README.md)


## Project Context

You will build a Phoenix 360 web apps project, that will consist of three websites:

* `app.local` - will be the main web app and the only one that runs a web server.
* `links.local` - the standalone website that will also be available at `app.local/_links`.
* `notes.local` - the standalone website that will also be available at `app.local/_notes`.

The web server for `app.local` will also serve the requests for `links.local` and `notes.local`, and will dispatch(not redirect or forward) any request to `app.local/_links` and `app.local/_notes` into the same application that runs `links.local` and `notes.local` respectively.

[Home](/README.md) | [TOC](#toc)


## Hosts Setup

In order to follow the Phoenix 360 Web Apps example you will need to add 3 entries to `/etc/hosts`:

```
sudo sh -c 'echo "127.0.0.1 app.local" >> /etc/hosts'
sudo sh -c 'echo "127.0.0.1 links.local" >> /etc/hosts'
sudo sh -c 'echo "127.0.0.1 notes.local" >> /etc/hosts'
```

You can check that they were added with:

```
$ cat /etc/hosts | tail -3
127.0.0.1 app.local
127.0.0.1 links.local
127.0.0.1 notes.local
```

[Home](/README.md) | [TOC](#toc)


## Create a Phoenix 360 Web Apps Project

Now you will create the three 360 web apps, install and fetch its dependencies, thus don't for get to reply **Y** to `Fetch and install dependencies? [Yn] Y` each time you run `mix phx.new ...`.

Create the root Phoenix project:

```
mix phx.new phoenix360 --no-ecto --live
cd phoenix360
```

Start the server:

```
iex -S mix phx.server
```

Test it works by visiting http://localhost:4000.

Next, lets create the app `:links`:

```
mix phx.new apps/links --no-ecto --live
```

Next, time to create the app `:notes`:

```
mix phx.new apps/notes --no-ecto --live
```

[Home](/README.md) | [TOC](#toc)

## Config Cleanup and Security Improvements

Elixir as now the `runtime.exs` configuration file that is invoked each time the application is started, thus making it the ideal place to have all configuration that is not strictly required at compile time, and the ideal place to put all configuration that may be changed on the targeted production environment.

The files `config.prod` and `prod.secret.exs` will be deleted and everything on them will be moved into `runtime.exs`. Configuration values and secrets on `runtime.exs` will be fetched directly from the environment.

> **IMPORTANT:** Using a `prod.secret.exs` at compile time for secrets is not the best approach in terms of security, because the release binaries will contain the secrets, therefore if the secrets are leaked in the CI/CD process or in any other way, then the release binary that goes into production is compromised. Also, any binary can be reverse-engineered to extract those secrets. Secrets are best to only be retrieved from the production environment where the release will run.

The file `config.exs` will also be depleted from all runtime configuration, that will be moved into `runtime.exs`, thus it will look like this:

```elixir
# This file is responsible for configuring your application
# and its dependencies with the aid of the Mix.Config module.
#
# This configuration file is loaded before any dependency and
# is restricted to this project.

# General application configuration
use Mix.Config

# Use Jason for JSON parsing in Phoenix
### THIS IS THE ONLY REQUIRED COMPILE TIME CONFIGURATION ###
config :phoenix, :json_library, Jason

# Import environment specific config. This must remain at the bottom
# of this file so it overrides the configuration defined above.
import_config "#{Mix.env()}.exs"
```

So, the `runtime.exs` configuration file needs to be created with:

```elixir
import Config

# Behind a proxy the host is for example `localhost` and port `4000`, but when
# the server is facing directly the Internet then will be `example.com` and port `443`.
phoenix360_host = System.fetch_env!("PHOENIX360_HOST")
phoenix360_host_port = System.fetch_env!("PHOENIX360_HOST_PORT")

# This is how the browser sees the server, thus in development it can be
# `localhost` and port `4000`, but in production, behind a proxy or directly
# facing the Internet, it will be `example.com` and port `443`
phoenix360_public_url = System.fetch_env!("PHOENIX360_PUBLIC_URL")
phoenix360_public_port = System.fetch_env!("PHOENIX360_PUBLIC_PORT")

# Configures the endpoint
config :phoenix360, Phoenix360Web.Endpoint,
  secret_key_base: System.fetch_env!("PHOENIX360_SECRET_KEY_BASE"),
  render_errors: [view: Phoenix360Web.ErrorView, accepts: ~w(html json), layout: false],
  pubsub_server: Phoenix360.PubSub,
  live_view: [signing_salt: System.fetch_env!("PHOENIX360_LIVE_VIEW_SIGNING_SALT")],
  cache_static_manifest: "priv/static/cache_manifest.json",
  url: [host: phoenix360_host, port: phoenix360_http_port],
  http: [
    port: phoenix360_http_port,
    transport_options: [socket_opts: [:inet6]],
  ]

# ## SSL Support
#
# To get SSL working, you will need to add the `https` key
# to the previous section and set your `:url` port to 443:
#
#     config :phoenix360, Phoenix360Web.Endpoint,
#       ...
#       url: [host: "example.com", port: 443],
#       https: [
#         port: 443,
#         cipher_suite: :strong,
#         keyfile: System.fetch_env!("PHOENIX360_SSL_KEY_PATH"),
#         certfile: System.fetch_env!("PHOENIX360_SSL_CERT_PATH"),
#         transport_options: [socket_opts: [:inet6]]
#       ]
#
# The `cipher_suite` is set to `:strong` to support only the
# latest and more secure SSL ciphers. This means old browsers
# and clients may not be supported. You can set it to
# `:compatible` for wider support.
#
# `:keyfile` and `:certfile` expect an absolute path to the key
# and cert in disk or a relative path inside priv, for example
# "priv/ssl/server.key". For all supported SSL configuration
# options, see https://hexdocs.pm/plug/Plug.SSL.html#configure/1
#
# We also recommend setting `force_ssl` in your endpoint, ensuring
# no data is ever sent via http, always redirecting to https:
#
#     config :phoenix360, Phoenix360Web.Endpoint,
#       force_ssl: [hsts: true]
#
# Check `Plug.SSL` for all available options in `force_ssl`.

if config_env() == :prod do
  log_level =
    (System.get_env("PHOENIX360_PROD_LOG_LEVEL") || "info")
    |> String.downcase() |> String.to_atom()

  # Configures Elixir's Logger
  config :logger, :console,
    level: log_level,
    format: "$time $metadata[$level] $message\n",
    metadata: [:request_id]
end
```


## Phoenix 360 Web Apps Secrets

In order for sessions, websockets and tokens to work across all 360 web apps it's necessary to use the same secrets across all of them, otherwise the websockets at `app.local/_links` and `app.local/_notes` will not work, because the csrf tokens aren't signed with the same secrets used by `links.local` and `notes.local`.

The following secrets must be the same for all the 360 web apps:

* `secret_key_base` at `config/config.exs`.
* `signing_salt` for Live View at `config/config.exs`.
* `signing_salt` and `key` for the session options at `lib/app_web/endpoint.ex`.

First, you need to set the environment variables with:

```
echo "PHOENIX360_SECRET_KEY_BASE=$(mix phx.gen.secret 64)" >> .env
echo "PHOENIX360_LIVE_VIEW_SIGNING_SALT=$(mix phx.gen.secret 32)" >> .env
echo "PHOENIX360_SESSION_SIGNING_SALT=$(mix phx.gen.secret 32)" >> .env
echo "PHOENIX360_SESSION_COOKIE_KEY_NAME=phoenix360_web_app_key" >> .env
```

> **IMPORTANT:** We will store the environment variables in the `.env` file, because in production you want to keep the same secrets across deployments, otherwise all connected sessions will be dropped, websockets will need to reconnect, and users will need to re-authenticate.


Now export them to your environment with:

```
export $(grep -v '^#' .env | xargs -0)
```
> **SECURITY:** Setting secrets from a `.env` file is a widely used practice, but more secure solutions exist, like vaults, and one of the most used is the [vaultproject.io](https://www.vaultproject.io/).

Confirm they are set with:

```
env | grep PHOENIX360 -
```

Output should be similar to this:

```
PHOENIX360_SECRET_KEY_BASE=BJi83QuLvA...
PHOENIX360_LIVE_VIEW_SIGNING_SALT=RpQae1...
PHOENIX360_SESSION_SIGNING_SALT=S3JZE2I...
PHOENIX360_SESSION_COOKIE_KEY_NAME=phoenix360_web_app_key
```

Next, add the `.env` file to `.gitignore`:

```
echo ".env" >> .gitignore
```

Now, go to each configuration file and edit the value for the secret to be retrieved from the environment with:

```elixir
# file: ./phoenix360/config/config.exs
# file: ./phoenix360/apps/links/config/config.exs
# file: ./phoenix360/apps/notes/config/config.exs

secret_key_base: System.fetch_env!("PHOENIX360_SECRET_KEY_BASE"),
live_view: [signing_salt: System.fetch_env!("PHOENIX360_LIVE_VIEW_SIGNING_SALT")]
```

and for the endpoints:

```elixir
# file: ./phoenix360/lib/app_web/endpoint.ex
# file: ./phoenix360/apps/links/lib/app_web/endpoint.ex
# file: ./phoenix360/apps/notes/lib/app_web/endpoint.ex

@session_options [
  store: :cookie,
  key: "_" <> System.fetch_env!("PHOENIX360_SESSION_COOKIE_KEY_NAME"),
  signing_salt: System.fetch_env!("PHOENIX360_SESSION_SIGNING_SALT")
]
```

[Home](/README.md) | [TOC](#toc)


## Phoenix 360 Web Apps Routing Dispatch

First, let's add the configuration for the new apps:

```elixir
# file: ./phoenix360/config/config.exs

phoenix360_http_port = System.fetch_env!("PHOENIX360_HTTP_PORT")
phoenix360_host = System.fetch_env!("PHOENIX360_HOST")
links_host = System.fetch_env!("PHOENIX360_LINKS_HOST")
notes_host = System.fetch_env!("PHOENIX360_NOTES_HOST")

config :phoenix360, Phoenix360Web.Endpoint,
  http: [
    port: phoenix360_http_port,
    # @link https://ninenines.eu/docs/en/cowboy/2.7/guide/routing/
    dispatch: [
      {
        phoenix360_host,
        [
          # @PHOENIX_360_WEB_APPS - Each included 360 web app needs to have an
          #   endpoint in the top level 360 web app. This is necessary in order
          #   for the top level 360 web app to know how to dispatch the requests.
          #   For example a request to app.local/_links/* will be dispatched to
          #   the `/links/*` endpoint for the LinksWeb.Router.
          #
          # {:path_match, :handler, :initial_state}
          {"/_links/[...]", Phoenix.Endpoint.Cowboy2Handler, {LinksWeb.Endpoint, []}},
          {"/_notes/[...]", Phoenix.Endpoint.Cowboy2Handler, {NotesWeb.Endpoint, []}},

          # @PHOENIX_360_WEB_APPS - This tells that all other request must be
          #   dispatched to the Phoenix360Web.Router, aka the one for the top
          #   level 360 web app.
          {:_, Phoenix.Endpoint.Cowboy2Handler, {Phoenix360Web.Endpoint, []}},
        ]
      },
      {
        links_host,
        [
          # {:path_match, :handler, :initial_state}
          {:_, Phoenix.Endpoint.Cowboy2Handler, {LinksWeb.Endpoint, []}},
        ]
      },
      {
        notes_host,
        [
          # {:path_match, :handler, :initial_state}
          {:_, Phoenix.Endpoint.Cowboy2Handler, {NotesWeb.Endpoint, []}},
        ]
      }
    ]
  ]

# @PHOENIX_360_WEB_APPS - We need to import the configuration for each of the
#   included 360 web apps.
import_config "../apps/links/config/config.exs"
import_config "../apps/notes/config/config.exs"

# @PHOENIX_360_WEB_APPS - No need to have a server running for `:links`, because
#    the request will come through the main 360 web app `phoenix360`:
config :links, LinksWeb.Endpoint,
  server: false

# @PHOENIX_360_WEB_APPS - No need to have a server running for `:notes`, because
#    the request will come through the main 360 web app `phoenix360`:
config :notes, NotesWeb.Endpoint,
  server: false

```

Next, let's add support for the `_links` endpoint in the router for the `:links` 360 web app:

```elixir
# @PHOENIX_360_WEB_APPS - support the endpoint used in the root 360 web app to
#   dispatch the incoming request to `app.local/_links`.
scope "/_links", LinksWeb do
  pipe_through :browser

  live "/", PageLive, :index
end
```

Finally, let's add support for the `_notes` endpoint in the router for the `:notes` 360 web app:

```elixir
# @PHOENIX_360_WEB_APPS - support the endpoint used in the root 360 web app to
#   dispatch the incoming request to `app.local/_notes`.
scope "/_notes", NotesWeb do
  pipe_through :browser

  live "/", PageLive, :index
end
```

[Home](/README.md) | [TOC](#toc)


## Live Code Reload

### Phoenix 360 Web App Config for Development

```elixir
# file: ./phoenix360/config/dev.exs

config :phoenix360, Phoenix360Web.Endpoint,
  ### REMOVE THE LINE FOR SETTING THE HTTP PORT ###
  # @PHOENIX_360_WEB_APPS - The http port is now set in `config.exs` via an env
  #   var.
  # http: [port: 4000],
  debug_errors: true,
  code_reloader: true,
  check_origin: false,
  watchers: [
    node: [
      "node_modules/webpack/bin/webpack.js",
      "--mode",
      "development",
      "--watch-stdin",
      cd: Path.expand("../assets", __DIR__)
    ],
    # @PHOENIX_360_WEB_APPS - Watch for static assets changes in the `:links`
    #   360 web app.
    node: [
      "node_modules/webpack/bin/webpack.js",
      "--mode",
      "development",
      "--watch-stdin",
      cd: Path.expand("../apps/links/assets", __DIR__)
    ],
    # @PHOENIX_360_WEB_APPS - Watch for static assets changes in the `:notes`
    #   360 web app.
    node: [
      "node_modules/webpack/bin/webpack.js",
      "--mode",
      "development",
      "--watch-stdin",
      cd: Path.expand("../apps/notes/assets", __DIR__)
    ]
  ]

config :phoenix360, Phoenix360Web.Endpoint,
  live_reload: [
    patterns: [
      ~r"priv/static/.*(js|css|png|jpeg|jpg|gif|svg)$",
      ~r"priv/gettext/.*(po)$",
      ~r"lib/phoenix360_web/(live|views)/.*(ex)$",
      ~r"lib/phoenix360_web/templates/.*(eex)$",

      # @PHOENIX_360_WEB_APPS - we need to give the relative path to each of the
      # included 360 web apps, otherwise live code reload will not work at the
      # top level, eg: app.local/_links.
      ~r"apps/links/lib/links_web/(live|views)/.*(ex)$",
      ~r"apps/links/lib/links_web/templates/.*(eex)$",
      ~r"apps/notes/lib/notes_web/templates/.*(eex)$",
      ~r"apps/notes/lib/notes_web/(live|views)/.*(ex)$"
    ]
  ]
```

### Phoenix 360 Included Web Apps Config for Development

For the `:links` 360 included web app:

```elixir
# file: ./phoenix360/apps/links/config/dev.exs

# @PHOENIX_360_WEB_APPS
config :links, LinksWeb.Endpoint,
  # When this web app is being used as a dependency of another web app the code
  # will not recompile on a live reload event if we not explicitly enable it:
  reloadable_apps: [:links]
```

For the `:links` 360 included web app:

```elixir
# file: ./phoenix360/apps/notes/config/dev.exs

# @PHOENIX_360_WEB_APPS
config :notes, NotesWeb.Endpoint,
  # When this web app is being used as a dependency of another web app the code
  # will not recompile on a live reload event if we not explicitly enable it:
  reloadable_apps: [:notes]
```

[Home](/README.md) | [TOC](#toc)


## Static Assets

### Phoenix 360 Included Web Apps

For the `:links` 360 included web app:

```elixir
# file: ./phoenix360/apps/links/config/config.exs

# @PHOENIX_360_WEB_APPS
config :links, LinksWeb.Endpoint,
  # Adds a prefix to the static assets path, eg: app.local/_links/css/app.css.
  static_url: [path: "/_links"]
```

For the `:notes` 360 included web app:

```elixir
# file: ./phoenix360/apps/notes/config/config.exs

# @PHOENIX_360_WEB_APPS
config :notes, NotesWeb.Endpoint,
  # Adds a prefix to the static assets path, eg: app.local/_notes/css/app.css.
  static_url: [path: "/_notes"]
```

Next, we need to enable live reload for each 360 web app static assets:

```elixir
# @file: ./phoenix360/apps/links/lib/links_web/endpoint.ex

# @PHOENIX_360_WEB_APPS - The prefix use for the top level static assets needs to be
#   the same here.
plug Plug.Static,
    #at: "/",
    at: "/_links",
    from: :links,
    gzip: false,
    only: ~w(css fonts images js favicon.ico robots.txt)
```

and for the the `:notes` 360 web app:

```elixir
# @file: ./phoenix360/apps/notes/lib/notes_web/endpoint.ex

# @PHOENIX_360_WEB_APPS - The prefix use for the top level static assets needs to be
#   the same here.
plug Plug.Static,
    #at: "/",
    at: "/_notes",
    from: :notes,
    gzip: false,
    only: ~w(css fonts images js favicon.ico robots.txt)
```

[Home](/README.md) | [TOC](#toc)


## Personalization

Each of the three 360 web apps you created are all identical, because they are the default Phoenix template.

Now, to be able to distinguish that when you go to `app.local/_links` you are indeed in `links.local`, and that when you go to `app.local/_notes` you are at `notes.local`, a small CSS tweak will do the trick.

For the `:links` app add to the end of the `phoenix.css` file:

```css
# file: phoenix360/apps/links/assets/css/phoenix.css

header {background: #7fabbf};
```

For the `:notes` app add to the end of the `phoenix.css` file:

```css
# file: phoenix360/apps/notes/assets/css/phoenix.css

header {background: #bfb27f};
```

Fix the home page link to be always relative to the root of the 360 web app:

```
find . -type f -name 'root.html.*' -exec sed -i 's|href="https://phoenixframework.org/"|href="/"|g' {} +
```

## Mix Dependencies

Now let's add the the Phoenix 360 apps to `mix.exs`:

```elixir
def deps do
  [
    # @PHOENIX_360_WEB_APPS - When including a 360 web app as dependency it's
    #   mandatory to pass to it the `env` it will running in.
    {:links, path: "apps/links", env: Mix.env()},
    {:notes, path: "apps/notes", env: Mix.env()},
  ]
end
```

Get the new deps:

```
mix deps.get
```

[Home](/README.md) | [TOC](#toc)


## Running

Start the `:phoenix360` server:

```
iex -S mix phx.server
```

You can now use the same `iex` shell to check the Supervision tree with:

```
:observer.start()
```

[Home](/README.md) | [TOC](#toc)
