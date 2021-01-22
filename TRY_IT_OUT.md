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

The web server for `app.local` will also serve the requests for `links.local` and `notes.local`, and will delegate(not redirect or forward) any request to `app.local/links` into `links.local`, and will do the same for the `:notes` 360 web app.

[Home](/README.md) | [TOC](#toc)


## Hosts Setup

In order to follow the Phoenix 360 Web Apps example you will need to add 3 entries to `/etc/hosts`:

```
echo "127.0.0.1 app.local" >> /etc/hosts
echo "127.0.0.1 links.local" >> /etc/hosts
echo "127.0.0.1 notes.local" >> /etc/hosts
```

[Home](/README.md) | [TOC](#toc)


## Create a Phoenix 360 Web Apps Project

Create the root Phoenix project:

```
mix phx.new phoenix360 --no-ecto --live
cd phoenix360
```

Start the server:

```
iex -S mix phx.server
```

Test it works by visiting http:localhost:4000.

Next, lets create the app `:links`:

```
mix phx.new apps/links --no-ecto --live
```

and reply **Y** to `Fetch and install dependencies? [Yn] Y`.

Next, time to create the app `:notes`:

```
mix phx.new apps/notes --no-ecto --live
```

Fix webpack not watching [symlinks](https://github.com/webpack/watchpack/issues/61#issuecomment-404112524):

```
find . -type f -name 'DirectoryWatcher.js' -exec sed -i 's/followSymlinks: false/followSymlinks: true/g' {} +
```

[Home](/README.md) | [TOC](#toc)


## Phoenix 360 Web Apps Secrets

In order for sessions, websockets and tokens to work accross all 360 web apps it's necessary to use the same secrets across all of them, otherwise the websockets at `app.local/links` and `app.local/notes` will not work, because the secrets used to sign the csrf tokens aren't the same that `links.local` and `notes.local` use.

The following secrets must be the same for all the 360 web apps:

* `secret_key_base` at `config/config.exs`.
* `signing_salt` for Live View at `config/config.exs`.
* `signing_salt` for the session options at `lib/app_web/endpoint.ex`

You can share them through environment variables:

```
export SECRET_KEY_BASE=$(mix phx.gen.secret 64)
export LIVE_VIEW_SIGNING_SALT=$(mix phx.gen.secret 32)
export SESSION_SIGNING_SALT=$(mix phx.gen.secret 32)
```

[Home](/README.md) | [TOC](#toc)


## Phoenix 360 Web Apps Routing Dispatch

Then on `phoenix360/config/config.exs` let's add the configuration for the new apps:

```elixir
# file: config/config.exs

phoenix360_http_port = System.fetch_env!(PHOENIX360_HTTP_PORT)
phoenix360_host = System.fetch_env!(PHOENIX360_HOST)
links_host = System.fetch_env!(LINKS_HOST)
notes_host = System.fetch_env!(NOTES_HOST)

config :phoenix360, Phoenix360Web.Endpoint,
  http: [
    port: phoenix360_http_port,
    dispatch: [
      {
        # @link https://ninenines.eu/docs/en/cowboy/2.7/guide/routing/
        phoenix360_host,
        [
          # @PHOENIX_360_WEB_APPS - Each included 360 web app needs to have an
          #   endpoint in the top level 360 web app. This is necessary in order
          #   for the top level 360 web app to know how to dispatch the requests.
          #   For example a request to example.com/links/* will be dispatched to
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
# file: config/dev.exs

config :phoenix360, Phoenix360Web.Endpoint,
  live_reload: [
    patterns: [
      ~r"priv/static/.*(js|css|png|jpeg|jpg|gif|svg)$",
      ~r"priv/gettext/.*(po)$",
      ~r"lib/phoenix360_web/(live|views)/.*(ex)$",
      ~r"lib/phoenix360_web/templates/.*(eex)$",

      # @PHOENIX_360_WEB_APPS - we need to give the relative path to each of the
      # included 360 web apps, otherwise live code reload will not work at the
      # top level, eg: example.com/links.
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
# file: apps/links/config/dev.exs

# @PHOENIX_360_WEB_APPS
config :links, NotesWeb.Endpoint,
  # When this web app is being used as a dependency of another web app the code
  # will not recompile on a live reload event if we not explicitly enable it:
  reloadable_apps: [:links]
```

For the `:links` 360 included web app:

```elixir
# file: apps/notes/config/dev.exs

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
# file: apps/links/config/config.exs

# @PHOENIX_360_WEB_APPS
config :links, NotesWeb.Endpoint,
  # Adds a prefix to the static assets path, eg: example.com/links/css/app.css.
  static_url: [path: "/links"]
```

For the `:notes` 360 included web app:

```elixir
# file: apps/notes/config/config.exs

# @PHOENIX_360_WEB_APPS
config :notes, NotesWeb.Endpoint,
  # Adds a prefix to the static assets path, eg: example.com/notes/css/app.css.
  static_url: [path: "/notes"]
```

Next, we need to enable live reload for each 360 web app static assets:

```elixir
# @file: apps/links/lib/links_web/endpoint.ex

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
# @file: apps/notes/lib/notes_web/endpoint.ex

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
