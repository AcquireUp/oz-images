FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

# OS deps
RUN apt-get update \
  && apt-get upgrade -y \
  && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    git \
    openssh-client \
    gnupg \
    build-essential \
    autoconf \
    bison \
    pkg-config \
    python3 \
    libssl-dev \
    libyaml-dev \
    libreadline-dev \
    zlib1g-dev \
    libncurses5-dev \
    libffi-dev \
    libgdbm-dev \
    libgmp-dev \
    libsqlite3-dev \
    libpq-dev \
    imagemagick \
    libmagic-dev \
    redis-server \
    postgresql \
    postgresql-contrib \
    postgis \
    postgresql-16-postgis-3 \
    sudo \
    unzip \
  && rm -rf /var/lib/apt/lists/*

# Make postgres tooling available on PATH for scripts.
RUN set -eux; \
  pg_bin="$(ls -d /usr/lib/postgresql/*/bin | head -n 1)"; \
  ln -sf "$pg_bin/initdb" /usr/local/bin/initdb; \
  ln -sf "$pg_bin/pg_ctl" /usr/local/bin/pg_ctl; \
  ln -sf "$pg_bin/pg_isready" /usr/local/bin/pg_isready; \
  ln -sf "$pg_bin/psql" /usr/local/bin/psql

# Node.js 22.x
RUN curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
  && apt-get update \
  && apt-get install -y --no-install-recommends nodejs \
  && rm -rf /var/lib/apt/lists/* \
  && corepack enable

# rbenv + ruby-build
ENV RBENV_ROOT=/opt/rbenv
ENV PATH="$RBENV_ROOT/bin:$RBENV_ROOT/shims:$PATH"

RUN git clone --depth 1 https://github.com/rbenv/rbenv.git "$RBENV_ROOT" \
  && git clone --depth 1 https://github.com/rbenv/ruby-build.git "$RBENV_ROOT/plugins/ruby-build"

SHELL ["/bin/bash", "-lc"]

# Install the exact Ruby patch versions required by the repos.
RUN rbenv install 3.4.5 \
  && rbenv install 3.4.8 \
  && rbenv global 3.4.8 \
  && gem update --system \
  && gem install bundler \
  && rbenv rehash

# Default non-root user
RUN set -eux; \
  useradd -m -s /bin/bash oz; \
  mkdir -p /work; \
  chown -R oz:oz /work /home/oz "$RBENV_ROOT"; \
  echo "oz ALL=(ALL) NOPASSWD:ALL" >/etc/sudoers.d/oz; \
  chmod 0440 /etc/sudoers.d/oz

# Ensure gems/bundler are writable for the non-root default user.
ENV GEM_HOME=/home/oz/.gem
ENV BUNDLE_PATH=/home/oz/.bundle
ENV PATH="$GEM_HOME/bin:$PATH"

WORKDIR /work
USER oz
