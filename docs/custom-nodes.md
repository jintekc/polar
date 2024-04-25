# Build Custom Node Images

Polar v1.0.0 supports using custom docker images for nodes in your networks. The instructions below will walk you through the process of creating custom images using the `master` branch of each implementation. There are some limitations, such as running as a non-root user to be compatible across multiple platforms. Feel free to customize these Dockerfiles to your specific needs.

> Note: The Dockerfiles for each implementation will likely change in the future. Please be mindful to make updates as needed when necessary.

- [LND](#LND)
- [c-lightning](#c-lightning)
- [Eclair](#Eclair)

## LND

1. `git clone https://github.com/lightningnetwork/lnd`
1. `cd lnd`
1. Overwrite `Dockerfile` with the following:

   ```
   FROM golang:1.19.7-alpine as builder

   # Force Go to use the cgo based DNS resolver. This is required to ensure DNS
   # queries required to connect to linked containers succeed.
   ENV GODEBUG netdns=cgo

   # Pass a tag, branch or a commit using build-arg.  This allows a docker
   # image to be built from a specified Git state.  The default image
   # will use the Git tip of master by default.
   ARG checkout="master"

   # Install dependencies and build the binaries.
   RUN apk add --no-cache --update alpine-sdk \
       git \
       make \
       gcc \
   &&  git clone https://github.com/lightningnetwork/lnd /go/src/github.com/lightningnetwork/lnd \
   &&  cd /go/src/github.com/lightningnetwork/lnd \
   &&  git checkout $checkout \
   &&  make \
   &&  make install tags="signrpc walletrpc chainrpc invoicesrpc routerrpc"

   # Start a new, final image.
   FROM alpine as final

   # Define a root volume for data persistence.
   VOLUME /root/.lnd

   # Add bash and ca-certs, for quality of life and SSL-related reasons.
   RUN apk --no-cache add \
       bash \
       su-exec \
       ca-certificates

   # Copy the binaries from the builder image.
   COPY --from=builder /go/bin/lncli /bin/
   COPY --from=builder /go/bin/lnd /bin/

   COPY docker-entrypoint.sh /entrypoint.sh

   RUN chmod a+x /entrypoint.sh
   # Expose lnd ports (p2p, rpc).
   VOLUME ["/home/lnd/.lnd"]

   EXPOSE 9735 8080 10000

   # Specify the start command and entrypoint as the lnd daemon.
   ENTRYPOINT ["/entrypoint.sh"]

   CMD ["lnd"]
   ```

1. Create a new file named `docker-entrypoint.sh` with the following contents

   ```
   #!/bin/sh
   set -e

   # containers on linux share file permissions with hosts.
   # assigning the same uid/gid from the host user
   # ensures that the files can be read/write from both sides
   if ! id lnd > /dev/null 2>&1; then
     USERID=${USERID:-1000}
     GROUPID=${GROUPID:-1000}

     echo "adding user lnd ($USERID:$GROUPID)"
     addgroup -g $GROUPID lnd
     adduser -D -u $USERID -G lnd lnd
   fi

   if [ $(echo "$1" | cut -c1) = "-" ]; then
     echo "$0: assuming arguments for lnd"

     set -- lnd "$@"
   fi

   if [ "$1" = "lnd" ] || [ "$1" = "lncli" ]; then
     echo "Running as lnd user: $@"
     exec su-exec lnd "$@"
   fi

   echo "$@"
   exec "$@"

   ```

1. `docker build -t lnd-master .` or for a fresh build run `docker build --no-cache -t lnd-master .`

### c-lightning

1. `git clone https://github.com/ElementsProject/lightning.git`
1. `cd lightning`
1. Overwrite `Dockerfile` with the following:

   ```
    #########################
    # Polar needs c-lightning compiled with the DEVELOPER=1 flag in order to decrease the
    # normal 30 second bitcoind poll interval using the argument --dev-bitcoind-poll=<seconds>.
    # When running in regtest, we want to be able to mine blocks and confirm transactions instantly.
    # Original Source: https://github.com/ElementsProject/lightning/blob/v24.02.2/Dockerfile
    #########################

    #########################
    # BEGIN ElementsProject/lightning/Dockerfile
    #########################

    # This dockerfile is meant to compile a core-lightning x64 image
    # It is using multi stage build:
    # * downloader: Download litecoin/bitcoin and qemu binaries needed for core-lightning
    # * builder: Compile core-lightning dependencies, then core-lightning itself with static linking
    # * final: Copy the binaries required at runtime
    # The resulting image uploaded to dockerhub will only contain what is needed for runtime.
    # From the root of the repository, run "docker build -t yourimage:yourtag ."
    FROM debian:bullseye-slim as downloader

    RUN set -ex \
      && apt-get update \
      && apt-get install -qq --no-install-recommends ca-certificates dirmngr wget

    WORKDIR /opt


    ARG BITCOIN_VERSION=22.0
    ARG TARBALL_ARCH=x86_64-linux-gnu
    ENV TARBALL_ARCH_FINAL=$TARBALL_ARCH
    ENV BITCOIN_TARBALL bitcoin-${BITCOIN_VERSION}-${TARBALL_ARCH_FINAL}.tar.gz
    ENV BITCOIN_URL https://bitcoincore.org/bin/bitcoin-core-$BITCOIN_VERSION/$BITCOIN_TARBALL
    ENV BITCOIN_ASC_URL https://bitcoincore.org/bin/bitcoin-core-$BITCOIN_VERSION/SHA256SUMS

    RUN mkdir /opt/bitcoin && cd /opt/bitcoin \
      #################### Polar Modification
      # We want to use the base image arch instead of the BUILDARG above so we can build the
      # multi-arch image with one command:
      # "docker buildx build --platform linux/amd64,linux/arm64 ..."
      ####################
      && TARBALL_ARCH_FINAL="$(uname -m)-linux-gnu" \
      && BITCOIN_TARBALL=bitcoin-${BITCOIN_VERSION}-${TARBALL_ARCH_FINAL}.tar.gz \
      && BITCOIN_URL=https://bitcoincore.org/bin/bitcoin-core-$BITCOIN_VERSION/$BITCOIN_TARBALL \
      && BITCOIN_ASC_URL=https://bitcoincore.org/bin/bitcoin-core-$BITCOIN_VERSION/SHA256SUMS \
      ####################
      && wget -qO $BITCOIN_TARBALL "$BITCOIN_URL" \
      && wget -qO bitcoin "$BITCOIN_ASC_URL" \
      && grep $BITCOIN_TARBALL bitcoin | tee SHA256SUMS \
      && sha256sum -c SHA256SUMS \
      && BD=bitcoin-$BITCOIN_VERSION/bin \
      && tar -xzvf $BITCOIN_TARBALL $BD/ --strip-components=1 \
      && rm $BITCOIN_TARBALL

    ENV LITECOIN_VERSION 0.16.3
    ENV LITECOIN_URL https://download.litecoin.org/litecoin-${LITECOIN_VERSION}/linux/litecoin-${LITECOIN_VERSION}-${TARBALL_ARCH_FINAL}.tar.gz

    # install litecoin binaries
    RUN mkdir /opt/litecoin && cd /opt/litecoin \
      && wget -qO litecoin.tar.gz "$LITECOIN_URL" \
      && tar -xzvf litecoin.tar.gz litecoin-$LITECOIN_VERSION/bin/litecoin-cli --strip-components=1 --exclude=*-qt \
      && rm litecoin.tar.gz

    FROM debian:bullseye-slim as builder

    ENV LIGHTNINGD_VERSION=master
    RUN apt-get update -qq && \
      apt-get install -qq -y --no-install-recommends \
      autoconf \
      automake \
      build-essential \
      ca-certificates \
      curl \
      dirmngr \
      gettext \
      git \
      gnupg \
      libpq-dev \
      libtool \
      libffi-dev \
      pkg-config \
      libssl-dev \
      protobuf-compiler \
      python3.9 \
      python3-dev \
      python3-mako \
      python3-pip \
      python3-venv \
      python3-setuptools \
      libev-dev \
      libevent-dev \
      qemu-user-static \
      wget \
      jq

    RUN wget -q https://zlib.net/fossils/zlib-1.2.13.tar.gz \
      && tar xvf zlib-1.2.13.tar.gz \
      && cd zlib-1.2.13 \
      && ./configure \
      && make \
      && make install && cd .. && \
      rm zlib-1.2.13.tar.gz && \
      rm -rf zlib-1.2.13

    RUN apt-get install -y --no-install-recommends unzip tclsh \
      && wget -q https://www.sqlite.org/2019/sqlite-src-3290000.zip \
      && unzip sqlite-src-3290000.zip \
      && cd sqlite-src-3290000 \
      && ./configure --enable-static --disable-readline --disable-threadsafe --disable-load-extension \
      && make \
      && make install && cd .. && rm sqlite-src-3290000.zip && rm -rf sqlite-src-3290000

    USER root
    ENV RUST_PROFILE=release
    ENV PATH=$PATH:/root/.cargo/bin/
    RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
    RUN rustup toolchain install stable --component rustfmt --allow-downgrade

    WORKDIR /opt/lightningd
    COPY . /tmp/lightning

    RUN git clone --recursive /tmp/lightning . && \
    git checkout $(git --work-tree=/tmp/lightning --git-dir=/tmp/lightning/.git rev-parse HEAD)

    ENV PYTHON_VERSION=3
    RUN curl -sSL https://install.python-poetry.org | python3 -

    RUN update-alternatives --install /usr/bin/python python /usr/bin/python3.9 1

    RUN pip3 install --upgrade pip setuptools wheel
    RUN pip3 wheel cryptography
    RUN pip3 install grpcio-tools

    RUN /root/.local/bin/poetry export -o requirements.txt --without-hashes --with dev
    RUN pip3 install -r requirements.txt

    RUN ./configure --prefix=/tmp/lightning_install --enable-static && \
      make && \
      /root/.local/bin/poetry run make install

    FROM debian:bullseye-slim as final

    RUN apt-get update && \
      apt-get install -y --no-install-recommends \
      tini \
      socat \
      inotify-tools \
      python3.9 \
      python3-pip \
      qemu-user-static \
      libpq5 && \
      apt-get clean && \
      rm -rf /var/lib/apt/lists/*

    ENV LIGHTNINGD_DATA=/root/.lightning
    ENV LIGHTNINGD_RPC_PORT=9835
    ENV LIGHTNINGD_PORT=9735
    ENV LIGHTNINGD_NETWORK=bitcoin

    RUN mkdir $LIGHTNINGD_DATA && \
      touch $LIGHTNINGD_DATA/config
    VOLUME [ "/root/.lightning" ]

    COPY --from=builder /tmp/lightning_install/ /usr/local/
    COPY --from=builder /usr/local/lib/python3.9/dist-packages/ /usr/local/lib/python3.9/dist-packages/
    COPY --from=downloader /opt/bitcoin/bin /usr/bin
    COPY --from=downloader /opt/litecoin/bin /usr/bin
    #################### Polar Modification
    # This line is removed as we have our own entrypoint file
    # Original line:
    # COPY tools/docker-entrypoint.sh entrypoint.sh
    ####################

    #########################
    # END ElementsProject/lightning/Dockerfile
    #########################

    COPY --from=builder /opt/lightningd/contrib/lightning-cli.bash-completion /etc/bash_completion.d/

    # install nodejs
    RUN apt-get update -y \
      && apt-get install -y curl gosu git \
      && apt-get clean \
      && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

    # install lightning-cli bash completion
    RUN curl -SLO https://raw.githubusercontent.com/scop/bash-completion/master/bash_completion \
      && mv bash_completion /usr/share/bash-completion/

    COPY docker-entrypoint.sh /entrypoint.sh
    COPY bashrc /home/clightning/.bashrc

    RUN chmod a+x /entrypoint.sh

    VOLUME ["/home/clightning"]
    VOLUME ["/opt/c-lightning-rest/certs"]

    EXPOSE 9735 9835 8080 10000

    ENTRYPOINT ["/entrypoint.sh"]

    CMD ["lightningd"]
   ```

1. Create a new file named `docker-entrypoint.sh` with the following contents

   ```
   #!/bin/sh
   set -e

   # give bitcoind a second to bootup
   sleep 1

   # containers on linux share file permissions with hosts.
   # assigning the same uid/gid from the host user
   # ensures that the files can be read/write from both sides
   if ! id clightning > /dev/null 2>&1; then
     USERID=${USERID:-1000}
     GROUPID=${GROUPID:-1000}

     echo "adding user clightning ($USERID:$GROUPID)"
     groupadd -f -g $GROUPID clightning
     useradd -r -u $USERID -g $GROUPID clightning
     # ensure correct ownership of user home dir
     mkdir -p /home/clightning
     chown clightning:clightning /home/clightning
   fi

   if [ $(echo "$1" | cut -c1) = "-" ]; then
     echo "$0: assuming arguments for lightningd"

     set -- lightningd "$@"
   fi

   # TODO: investigate hsmd error on Windows
   # https://gist.github.com/jamaljsr/404c20f99be2f77fff2d834e2449158b

   if [ "$1" = "lightningd" ] || [ "$1" = "lightning-cli" ]; then
     echo "Running as clightning user: $@"
     exec gosu clightning "$@"
   fi

   echo "$@"
   exec "$@"
   ```

1. `docker build -t core-lightning-master .`

## Eclair

1. `git clone https://github.com/ACINQ/eclair.git`
1. `cd eclair`
1. Overwrite `Dockerfile` with the following:

   ```
   FROM adoptopenjdk/openjdk11:jdk-11.0.3_7-alpine as BUILD

   # Setup maven, we don't use https://hub.docker.com/_/maven/ as it declare .m2 as volume, we loose all mvn cache
   # We can alternatively do as proposed by https://github.com/carlossg/docker-maven#packaging-a-local-repository-with-the-image
   # this was meant to make the image smaller, but we use multi-stage build so we don't care
   RUN apk add --no-cache curl tar bash

   ARG MAVEN_VERSION=3.6.3
   ARG USER_HOME_DIR="/root"
   ARG SHA=c35a1803a6e70a126e80b2b3ae33eed961f83ed74d18fcd16909b2d44d7dada3203f1ffe726c17ef8dcca2dcaa9fca676987befeadc9b9f759967a8cb77181c0
   ARG BASE_URL=https://apache.osuosl.org/maven/maven-3/${MAVEN_VERSION}/binaries

   RUN mkdir -p /usr/share/maven /usr/share/maven/ref \
     && curl -fsSL -o /tmp/apache-maven.tar.gz ${BASE_URL}/apache-maven-${MAVEN_VERSION}-bin.tar.gz \
     && echo "${SHA}  /tmp/apache-maven.tar.gz" | sha512sum -c - \
     && tar -xzf /tmp/apache-maven.tar.gz -C /usr/share/maven --strip-components=1 \
     && rm -f /tmp/apache-maven.tar.gz \
     && ln -s /usr/share/maven/bin/mvn /usr/bin/mvn

   ENV MAVEN_HOME /usr/share/maven
   ENV MAVEN_CONFIG "$USER_HOME_DIR/.m2"

   # Let's fetch eclair dependencies, so that Docker can cache them
   # This way we won't have to fetch dependencies again if only the source code changes
   # The easiest way to reliably get dependencies is to build the project with no sources
   WORKDIR /usr/src
   COPY pom.xml pom.xml
   COPY eclair-core/pom.xml eclair-core/pom.xml
   COPY eclair-node/pom.xml eclair-node/pom.xml
   COPY eclair-node-gui/pom.xml eclair-node-gui/pom.xml
   COPY eclair-node/modules/assembly.xml eclair-node/modules/assembly.xml
   COPY eclair-node-gui/modules/assembly.xml eclair-node-gui/modules/assembly.xml
   RUN mkdir -p eclair-core/src/main/scala && touch eclair-core/src/main/scala/empty.scala
   # Blank build. We only care about eclair-node, and we use install because eclair-node depends on eclair-core
   #################### Polar Modification
   ENV MAVEN_OPTS=-Xmx256m -XX:MaxPermSize=512m
   ####################
   RUN mvn install -pl eclair-node -am
   RUN mvn clean

   # Only then do we copy the sources
   COPY . .

   # And this time we can build in offline mode, specifying 'notag' instead of git commit
   RUN mvn package -pl eclair-node -am -DskipTests -Dgit.commit.id=notag -Dgit.commit.id.abbrev=notag -o
   # It might be good idea to run the tests here, so that the docker build fail if the code is bugged

   # We currently use a debian image for runtime because of some jni-related issue with sqlite
   FROM openjdk:11.0.4-jre-slim
   WORKDIR /app

   # install jq for eclair-cli
   RUN apt-get update && apt-get install -y bash jq curl unzip gosu

   # copy and install eclair-cli executable
   COPY --from=BUILD /usr/src/eclair-core/eclair-cli .
   RUN chmod +x eclair-cli && mv eclair-cli /sbin/eclair-cli

   # we only need the eclair-node.zip to run
   COPY --from=BUILD /usr/src/eclair-node/target/eclair-node-*.zip ./eclair-node.zip
   RUN unzip eclair-node.zip && mv eclair-node-* eclair-node && chmod +x eclair-node/bin/eclair-node.sh

   #################### Polar Modification
   # Original lines:
   # ENV ECLAIR_DATADIR=/data
   # ENV JAVA_OPTS=
   # RUN mkdir -p "$ECLAIR_DATADIR"
   # VOLUME [ "/data" ]
   # ENTRYPOINT JAVA_OPTS="${JAVA_OPTS}" eclair-node/bin/eclair-node.sh "-Declair.datadir=${ECLAIR_DATADIR}"
   ENV ECLAIR_DATADIR=/home/eclair/
   RUN chmod -R a+x eclair-node/*
   RUN ls -al eclair-node/bin

   COPY docker-entrypoint.sh /entrypoint.sh

   RUN chmod a+x /entrypoint.sh

   VOLUME ["/home/eclair"]

   EXPOSE 9735 8080

   ENTRYPOINT ["/entrypoint.sh"]

   CMD $JAVA_OPTS bash eclair-node/bin/eclair-node.sh -Declair.datadir=$ECLAIR_DATADIR
   ####################

   ```

1. Create a new file named `docker-entrypoint.sh` with the following contents

   ```
   #!/usr/bin/env bash
   set -e

   # give bitcoind a second to bootup
   sleep 1

   # containers on linux share file permissions with hosts.
   # assigning the same uid/gid from the host user
   # ensures that the files can be read/write from both sides
   if ! id eclair > /dev/null 2>&1; then
     USERID=${USERID:-1000}
     GROUPID=${GROUPID:-1000}

     echo "adding user eclair ($USERID:$GROUPID)"
     groupadd -f -g $GROUPID eclair
     useradd -r -u $USERID -g $GROUPID eclair
     # ensure correct ownership of user home dir
     mkdir -p /home/eclair
     chown eclair:eclair /home/eclair
   fi

   if [ "$1" = "polar-eclair" ]; then
     # convert command line args to JAVA_OPTS
     JAVA_OPTS=""
     for arg in "$@"
     do
       if [ "${arg:0:2}" = "--" ]; then
         JAVA_OPTS="$JAVA_OPTS -Declair.${arg:2}"
       fi
     done
     # trim leading/trailing whitespace
     JAVA_OPTS="$(sed -e 's/[[:space:]]*$//' <<<${JAVA_OPTS})"

     echo "Running as eclair user:"
     echo "bash eclair-node/bin/eclair-node.sh $JAVA_OPTS"
     exec gosu eclair bash eclair-node/bin/eclair-node.sh $JAVA_OPTS
   fi

   echo "Running: $@"
   exec "$@"
   ```

1. `docker build -t eclair-master .`
