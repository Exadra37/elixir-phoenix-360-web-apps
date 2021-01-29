# RELEASES WITH DOCKER

Let's take the Dockerfile example from the [Elixir Deployment Guide](https://hexdocs.pm/phoenix/releases.html#containers) and use it as the base to build your releases.

## Improve the Dockerfile

Let's replace the `nobody` user with a dedicated and unprivileged user to run the release:

```dockerfile
FROM hexpm/elixir:1.11.2-erlang-23.1.2-alpine-3.12.1 as build

# install build dependencies
RUN apk add --no-cache build-base npm git python3

# prepare build dir
WORKDIR /app

# install hex + rebar
RUN mix local.hex --force && \
    mix local.rebar --force

# set build ENV
ENV MIX_ENV=prod

# install mix dependencies
COPY mix.exs mix.lock ./
RUN mix deps.get --only $MIX_ENV
RUN mkdir config
# Dependencies sometimes use compile-time configuration. Copying
# these compile-time config files before we compile dependencies
# ensures that any relevant config changes will trigger the dependencies
# to be re-compiled.
COPY config/config.exs config/$MIX_ENV.exs config/
RUN mix deps.compile

# build assets
COPY assets/package.json assets/package-lock.json ./assets/
# install all npm dependencies from scratch
RUN npm --prefix ./assets ci --progress=false --no-audit --loglevel=error

COPY priv priv

# Note: if your project uses a tool like https://purgecss.com/,
# which customizes asset compilation based on what it finds in
# your Elixir templates, you will need to move the asset compilation step
# down so that `lib` is available.
COPY assets assets
# use webpack to compile npm dependencies - https://www.npmjs.com/package/webpack-deploy
RUN npm run --prefix ./assets deploy
RUN mix phx.digest

# compile and build the release
COPY lib lib
RUN mix compile
# changes to config/runtime.exs don't require recompiling the code
COPY config/runtime.exs config/
# uncomment COPY if rel/ exists
# COPY rel rel
RUN mix release

# Start a new build stage so that the final image will only contain
# the compiled release and other runtime necessities
FROM alpine:3.12.1 AS app
RUN apk add --no-cache openssl ncurses-libs

ENV USER="phoenix"
ENV HOME=/home/"${USER}"
ENV APP_DIR="${HOME}/app"

# Creates a unprivileged user to run the app
RUN \
  addgroup \
   -g 1000 \
   -S "${USER}" && \
  adduser \
   -s /bin/sh \
   -u 1000 \
   -G "${USER}" \
   -h "${HOME}" \
   -D "${USER}" && \

  su "${USER}" sh -c "mkdir ${APP_DIR}"

# Everything from this line onwards will run in the context of the unprivileged user.
USER "${USER}"

WORKDIR "${APP_DIR}"

COPY --from=build --chown="${USER}":"${USER}" /app/_build/prod/rel/my_app ./

ENTRYPOINT ["bin/my_app"]

# Docker Usage:
#  * build: sudo docker build -t phoenix/my_app .
#  * shell: sudo docker run --rm -it --entrypoint "" -p 80:4000 -p 443:4040 phoenix/my_app sh
#  * run:   sudo docker run --rm -it -p 80:4000 -p 443:4040 --env-file .env --name my_app phoenix/my_app
#  * exec:  sudo docker exec -it my_app sh
#  * logs:  sudo docker logs --follow --tail 10 my_app
#
# Extract the production release to your host machine with:
#
# ```
# mkdir archive
# sudo docker run --rm -it --entrypoint "" -v "$PWD/archive:/home/phoenix/archive"  phoenix/my_app sh -c "tar zcf /home/phoenix/archive/app.tar.gz ."
# ls -al archive
# ````
CMD ["start"]
```

## Build the Release

From the root of your Phoenix project run:

```
sudo docker build --tag phoenix/my_app .
```

## Extract the Release from the Container to the Host

Create a folder in the host machine to map into the docker container:

```
mkdir archive
```

Build the tar ball archive inside the docker container in the shared folder between the host and the docker container:

```
sudo docker run \
  --rm \
  -it \
  --entrypoint "" \
  --volume "$PWD/archive:/home/phoenix/archive" \
  phoenix/my_app \
  sh -c "tar zcf /home/phoenix/archive/app.tar.gz ."
```

Check that it exists in the host:

```
ls -al archive
````

## Release Usage with Docker

### The Environment Variables

Remember that you need to create a `.env` file with all the environment variables that are needed by your Phoenix app.

### Run the Release Within Docker

```
sudo docker run \
  --rm \
  --name my_app \
  --env-file .env \
  --publish 80:4000 \
  --publish 443:4040 \
  --restart unless-stopped \
  phoenix/my_app
```

### Tail the Logs while the Container is Running

Will start tailing the logs in real-time from the last 10 lines.

```
sudo docker logs --follow --tail 10 my_app
```

### Get a Shell Inside a Running Container

```
sudo docker exec -it my_app sh
```

### Get a Shell in a new Container

```
sudo docker run \
  --rm \
  -it \
  --env-file .env \
  --entrypoint "" \
  --publish 80:4000 \
  --publish 443:4040 \
  phoenix/my_app sh
```
