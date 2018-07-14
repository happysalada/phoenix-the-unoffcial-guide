FROM elixir:1.6.6-alpine

ARG APP_NAME=union
ARG MIX_ENV=prod
ARG APP_VERSION=0.0.0
ARG PHOENIX_SUBDIR=.
ENV MIX_ENV ${MIX_ENV}
ENV APP_VERSION ${APP_VERSION}
ENV REPLACE_OS_VARS true

WORKDIR /opt/app
# use yarn instead of npm to reduce from 180s to 60s
# add git since one dependency pulls from git
RUN apk update \
  && apk --no-cache --update add nodejs yarn git build-base \
  && mix local.rebar --force \
  && mix local.hex --force
COPY . .
RUN mix do deps.get, deps.compile, compile

RUN cd ${PHOENIX_SUBDIR}/assets \
  && yarn install \
  && yarn deploy \
  && cd .. \
  && mix phx.digest
RUN mix release --env=${MIX_ENV} --verbose \
  && mv _build/prod/rel/${APP_NAME} /opt/release \
  && mv /opt/release/bin/${APP_NAME} /opt/release/bin/start_server

# minimal runtime image
FROM alpine:3.8
# bash is required by distillery
RUN apk update && apk --no-cache --update add bash openssl-dev
ENV REPLACE_OS_VARS true
WORKDIR /opt/app
COPY --from=0 /opt/release .
CMD ["/opt/app/bin/start_server", "foreground"]
