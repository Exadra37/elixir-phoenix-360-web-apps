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
* `links.local` - the standalone website that will also be available at `app.local/links`.
* `notes.local` - the standalone website that will also be available at `app.local/notes`.

The web server for `app.local` will also serve the requests for `links.local` and `notes.local`, and will dispatch(not redirect or forward) any request to `app.local/links` and `app.local/notes` into the same application that runs `links.local` and `notes.local` respectively.

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


## Phoenix 360 Web Apps Secrets

In order for sessions, websockets and tokens to work across all 360 web apps it's necessary to use the same secrets across all of them, otherwise the websockets at `app.local/links` and `app.local/notes` will not work, because the csrf tokens aren't signed with the same secrets used by `links.local` and `notes.local`.

The following secrets must be the same for all the 360 web apps:

* `secret_key_base` at `config/config.exs`.
* `signing_salt` for Live View at `config/config.exs`.
* `signing_salt` for the session options at `lib/app_web/endpoint.ex`.

First, you need to set the environment variables with:

```
echo "PHOENIX360_SECRET_KEY_BASE=$(mix phx.gen.secret 64)" >> .env
echo "PHOENIX360_LIVE_VIEW_SIGNING_SALT=$(mix phx.gen.secret 32)" >> .env
echo "PHOENIX360_SESSION_SIGNING_SALT=$(mix phx.gen.secret 32)" >> .env
```

> **IMPORTANT:** We will store the environment variables in the `.env` file, because in production you want to keep the same secrets across deployments, otherwise all connected sessions will be dropped, websockets will need to reconnect, and users will need to re-authenticate.

> **SECURITY:** The use of a `.env` file is a widely used practice, but more secure solutions exist, like vaults, and one of the most used is the [vaultproject.io](https://www.vaultproject.io/).

Now export them to your environment with:

```
export $(grep -v '^#' .env | xargs -0)
```

Confirm they are set with:

```
env | grep PHOENIX360 -
```

Output should be similar to this:

```
PHOENIX360_SECRET_KEY_BASE=BJi83QuLvA/Avxx2FbqW2dhY1H/V/PWjNUmMe2igf1RzvVw9Lo/tfPZcDUzQHfVq
PHOENIX360_LIVE_VIEW_SIGNING_SALT=RpQae1VGBO9C6R8El33YtzIIvoxvgngk
PHOENIX360_SESSION_SIGNING_SALT=S3JZE2IgcR5rp9Ou2XUc62SdSUEPTsuM
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
  signing_salt: System.fetch_env!("PHOENIX360_SESSION_SIGNING_SALT")
]
```

[Home](/README.md) | [TOC](#toc)


## Phoenix 360 Web Apps Routing Dispatch

Let's add the configuration for the new apps:

```elixir
# file: ./phoenix360/config/config.exs

phoenix360_http_port = System.fetch_env!("PHOENIX360_HTTP_PORT")
phoenix360_host = System.fetch_env!("PHOENIX360_HOST")
links_host = System.fetch_env!("LINKS_HOST")
notes_host = System.fetch_env!("NOTES_HOST")

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
          #   For example a request to app.local/links/* will be dispatched to
          #   the `/links/*` endpoint for the LinksWeb.Router.
          #
          # {:path_match, :handler, :initial_state}
          {"/links/[...]", Phoenix.Endpoint.Cowboy2Handler, {LinksWeb.Endpoint, []}},
          {"/notes/[...]", Phoenix.Endpoint.Cowboy2Handler, {NotesWeb.Endpoint, []}},

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

[Home](/README.md) | [TOC](#toc)


## Live Code Reload

### Phoenix 360 Web App Config for Development

```elixir
# file: ./phoenix360/config/dev.exs

config :phoenix360, Phoenix360Web.Endpoint,
  ### REMOVE THE LINE FOR SETTING THE HTTP PORT ###
  # The http port is now set in `config.exs` via the env var.
  # http: [port: 4000],
  debug_errors: true,


config :phoenix360, Phoenix360Web.Endpoint,
  live_reload: [
    patterns: [
      ~r"priv/static/.*(js|css|png|jpeg|jpg|gif|svg)$",
      ~r"priv/gettext/.*(po)$",
      ~r"lib/phoenix360_web/(live|views)/.*(ex)$",
      ~r"lib/phoenix360_web/templates/.*(eex)$",

      # @PHOENIX_360_WEB_APPS - we need to give the relative path to each of the
      # included 360 web apps, otherwise live code reload will not work at the
      # top level, eg: app.local/links.
      ~r"apps/notes/lib/notes_web/(live|views)/.*(ex)$",
      ~r"apps/notes/lib/notes_web/templates/.*(eex)$",
      ~r"apps/links/lib/links_web/(live|views)/.*(ex)$",
      ~r"apps/links/lib/links_web/templates/.*(eex)$"
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
  # Adds a prefix to the static assets path, eg: app.local/links/css/app.css.
  static_url: [path: "/links"]
```

For the `:notes` 360 included web app:

```elixir
# file: ./phoenix360/apps/notes/config/config.exs

# @PHOENIX_360_WEB_APPS
config :notes, NotesWeb.Endpoint,
  # Adds a prefix to the static assets path, eg: app.local/notes/css/app.css.
  static_url: [path: "/notes"]
```

Next, we need to enable live reload for each 360 web app static assets:

```elixir
# @file: ./phoenix360/apps/links/lib/links_web/endpoint.ex

# @PHOENIX_360_WEB_APPS - The prefix use for the top level static assets needs to be
#   the same here.
plug Plug.Static,
    #at: "/",
    at: "/links",
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
    at: "/notes",
    from: :notes,
    gzip: false,
    only: ~w(css fonts images js favicon.ico robots.txt)
```

[Home](/README.md) | [TOC](#toc)


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
