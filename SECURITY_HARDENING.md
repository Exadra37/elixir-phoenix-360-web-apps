# SECURITY HARDNEING

Let's see how the Phoenix 360 Web Apps methodology can be used in practice by implementing the [White Paper](/WHITE_PAPER.md#an-usage-example) recommendations for security hardening.

## TOC

* [Project context](#project-context)
* [Hosts setup](#hosts-setup)
* [Create a project](#create-a-phoenix-360-web-apps-project)
* [Secrets](#phoenix-360-web-apps-secrets)

[Home](/README.md)


## Project Context

This guide assumes that you are following it in a Linux host, therefore if you are on Windows or MAC don't forget to start your Linux Elixir VM or an Elixir docker container and run this guide inside it.

[Home](/README.md) | [TOC](#toc)


## Hosts Setup

In order to follow the Phoenix 360 Web App Security Hardening example you will need to add an entry to your `/etc/hosts` file:

```
sudo sh -c 'echo "127.0.0.1 app.local" >> /etc/hosts'
```

You can check that they were added with:

```
$ cat /etc/hosts | tail -1
127.0.0.1 app.local
```

[Home](/README.md) | [TOC](#toc)


## Create a Phoenix 360 Web App Project

Now you will create the Phoenix 360 web app, install and fetch its dependencies, thus don't forget to reply **Y** to `Fetch and install dependencies? [Yn] Y` when you run `mix phx.new ...`.

Create the Phoenix 360 web app project with:

```
mix phx.new phoenix360 --no-ecto --live
cd phoenix360
```

Change `localhost` to `app.local` in `config/config.exs`:

```
url: [host: "app.local"],
```

or just do it with:

```
find ./config -type f -name '*.exs' -exec sed -i 's|"localhost"|"app.local"|g' {} +
```

Start the server:

```
iex -S mix phx.server
```

Test it works by visiting http://app.local:4000.

[Home](/README.md) | [TOC](#toc)

## Environment Variables

The default Phoenix configuration is mostly driven by hardcoded values, making it
less flexible and/or confusing to use, and creates some security concerns when
this harcoded values are secret or salts that will be compile into the release binary.

Next you will change the configuration to be driven by environment variables, but
before you do that it's necessary to first set them up:

```
echo "PHOENIX360_SECRET_KEY_BASE=$(mix phx.gen.secret 64)" >> .env
echo "PHOENIX360_LIVE_VIEW_SIGNING_SALT=$(mix phx.gen.secret 32)" >> .env
echo "PHOENIX360_SESSION_SIGNING_SALT=$(mix phx.gen.secret 32)" >> .env
echo "PHOENIX360_SESSION_ENCRYPTION_SALT=$(mix phx.gen.secret 32)" >> .env
echo "PHOENIX360_SERVER_HOSTNAME=app.local" >> .env
echo "PHOENIX360_SERVER_HTTP_PORT=4000" >> .env
echo "PHOENIX360_SERVER_HTTPS_PORT=4001" >> .env
echo "PHOENIX360_PUBLIC_DOMAIN=app.local" >> .env
echo "PHOENIX360_PUBLIC_DOMAIN_HTTP_PORT=4000" >> .env
echo "PHOENIX360_PUBLIC_DOMAIN_HTTPS_PORT=4001" >> .env
echo "LETSENCRYPT_ACME_HTTP_PORT=4002" >> .env
echo "LETSENCRYPT_ACME_EMAILS=__your__@mail.com" >> .env
```

> **IMPORTANT:** You will store the environment variables in the `.env` file, because in production you want to keep the same secrets across deployments, otherwise all connected sessions will be dropped, websockets will need to reconnect, and users will need to re-authenticate.

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
...
```

Next, add the `.env` file to `.gitignore`:

```
echo ".env" >> .gitignore
```

[Home](/README.md) | [TOC](#toc)


## Config Cleanup and Security Improvements

Elixir as now the `runtime.exs` configuration file that is invoked each time the application is started, therefore making it the ideal place for all configuration not strictly required at compile time, thus allowing to configure the application as needed in the target environment where it will run.

The files `prod.exs` and `prod.secret.exs` will be deleted and everything on them will be moved into `runtime.exs`, where configuration values and secrets will be fetched directly from the environment.

Remove the `prod` configuration files with:

```
rm -rf config/prod.exs config/prod.secret.exs
```

> **IMPORTANT:** Using a `prod.secret.exs` at compile time for secrets is not the best approach in terms of security, because the release binaries will contain the secrets, therefore if the secrets are leaked in the CI/CD process or in any other way, then the release binary that goes into production is compromised. Also, any binary can be reverse-engineered to extract those secrets. Secrets are best to only be retrieved from the production environment where the release will run.

The file `config.exs` will also be depleted from all runtime configuration, that will be moved into `runtime.exs`, thus edit it and replace all it's content by this one:

```elixir
# file: config/config.exs

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
#
# Excludes `:prod` because it's configuration has moved to `runtime.exs` and
# it's now driven by environment variables.
if Mix.env() != :prod do
  import_config "#{Mix.env()}.exs"
end
```

Now, the `runtime.exs` configuration file needs to be created with:

```elixir
# file: config/runtime.exs

import Config

# Use here the hostname for the server itself, that is not necessarily the one
# you type in the browser.
# Behind a proxy or in a docker container the host is for example `localhost`
# and port `4000`, but when the server is facing directly the Internet will be
# like `example.com` and port `443`.
host = System.fetch_env!("PHOENIX360_HOST")
host_port = System.fetch_env!("PHOENIX360_HOST_PORT")

# Use here the same host and port you type in the browser. So, in development it
# can be `localhost` and port `4000`, but in production, behind a proxy, on a
# docker container or directly facing the Internet, it will be like
# `example.com` and port `443`.
public_host = System.fetch_env!("PHOENIX360_PUBLIC_HOST")
public_host_port = System.fetch_env!("PHOENIX360_PUBLIC_HOST_PORT")

# Phoenix by default compiles secrets and salt values into the release, and this
# becomes a security concern, because this values can be leaked during the CI/CD
# pipeline. Also, anyone getting their hands in the release binary can reverse
# engineer it to extract this sensitive values, therefore you may put a release
# into production without knowing that is already compromised or it can be
# compromised after you deploy it.
config :phoenix360, Phoenix360Web.Endpoint,
  secret_key_base: System.fetch_env!("PHOENIX360_SECRET_KEY_BASE"),
  render_errors: [view: Phoenix360Web.ErrorView, accepts: ~w(html json), layout: false],
  pubsub_server: Phoenix360.PubSub,
  live_view: [signing_salt: System.fetch_env!("PHOENIX360_LIVE_VIEW_SIGNING_SALT")],
  cache_static_manifest: "priv/static/cache_manifest.json",
  check_origin: true,
  server: true,
  url: [host: host, port: host_port],
  http: [
    port: public_host_port,
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

[Home](/README.md) | [TOC](#toc)


## Endpoint Session Configuration Security

Let's remove compile time secrets and salts, because they are a security risk
for production releases.

On the top of the Endpoint module replace:

```elixir
# file: phoenix360/lib/phoenix360_web/endpoint.ex

@session_options [
    store: :cookie,
    key: "_phoenix360_key",
    signing_salt: "eoMRUSNz"
  ]
```

with:

```elixir
# file: phoenix360/lib/phoenix360_web/endpoint.ex

@session_options {__MODULE__, :_session_options, []}
```

And at the end of the Endpoint file replace this:

```elixir
plug Plug.MethodOverride
plug Plug.Head
plug Plug.Session, @session_options # <--- TO BE REPLACED
plug Phoenix360Web.Router
```

with this:

```elixir
plug Plug.MethodOverride
plug Plug.Head
plug :_plug_session # <--- REPLACED
plug Phoenix360Web.Router

# Passing to `plug` a function means that it will be invoke only at runtime,
# therefore avoiding that the release is built with the session signing and
# encryption salt values, that creates a security concern.
defp _plug_session(conn, _opts) do
  opts = _session_options() |> Plug.Session.init()
  Plug.Session.call(conn, opts)
end

defp _session_options() do
  Application.fetch_env!(:phoenix360, Phoenix360Web.Endpoint)[:session_options]
end
```

Now, that the Phoenix 360 web app retrieves their secrets and salts at runtime we need to update the `runtime.exs` configuration to support it:

```elixir
config :phoenix360, Phoenix360Web.Endpoint,
  # To be retrieved at runtime by `Phoenix360Web.Endpoint.RuntimeSession.options/0`.
  session_options:
    [
      store: :cookie,
      key: "_phoenix360_key",
      signing_salt: System.fetch_env!("PHOENIX360_SESSION_SIGNING_SALT"),
      encryption_salt: System.fetch_env!("PHOENIX360_SESSION_ENCRYPTION_SALT"),
    ]
```

If you are paying attention, you may have noticed the addition of the `encryption_salt` key, that tells Phoenix to encrypt all session cookies, thus making impossible for anyone to read the contents of the cookie. You may say that it will hurt performance, but the impact is negligible. You are always best to be as much secure as you can by default, eg: better safe then sorry, as the say says.

[Home](/README.md) | [TOC](#toc)


## Secure Browse Headers

### Content Security Policy

In the `router.ex` at the `:browser` pipeline, replace this:

```elixir
pipeline :browser do
  # omitted code...

  plug :put_secure_browser_headers
end
```

with this:

```elixir
# Necessary to allow webpack to work during development
@unsafe_eval (if Mix.env() == :prod do "" else "script-src 'self' 'unsafe-eval';" end)

pipeline :browser do
  # omitted code...

  plug :put_secure_browser_headers, %{"content-security-policy" => "default-src 'self'; #{@unsafe_eval}"}
end
```

This is a basic configuration to protect against the most XSS attacks. Please adapt it to your use case. Some useful links:

* https://report-uri.com/home/tools
* https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP
* https://hexdocs.pm/phoenix/Phoenix.Controller.html#put_secure_browser_headers/2

[Home](/README.md) | [TOC](#toc)


## Security Static Analysis

Now let's add [nccgroup/sobelow](https://github.com/nccgroup/sobelow) to the
`:dev` dependencies in order to be able to keep the Phoenix 360 Web App as much
secure as possible:

```elixir
def deps do
  [
    # Security-focused static analysis for the Phoenix Framework
    {:sobelow, "~> 0.8", only: :dev}
  ]
end
```

Get the new deps:

```
mix deps.get
```

Now, try to run it with:

```
mix sobelow
```

If you followed all the steps it should not give you any alert.

[Home](/README.md) | [TOC](#toc)


## Run in Development mode

Start the `:phoenix360` server:

```
iex -S mix phx.server
```

Try it again at http://app.local:4000.

[Home](/README.md) | [TOC](#toc)


## Run in Production mode

Start the `:phoenix360` server:

```
ENV=prod iex -S mix phx.server
```

Try it again at http://app.local:4000.

[Home](/README.md) | [TOC](#toc)


## Build and Run a Release

Build the release with:

```
ENV=prod mix release
```

Run it with:

```
_build/prod/rel/phoenix360/bin/phoenix360 start
```

Try it again at http://app.local:4000.

[Home](/README.md) | [TOC](#toc)
