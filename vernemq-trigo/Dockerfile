# ---------- builder: build VerneMQ release ----------
FROM erlang:27 AS vernemq-builder
ARG VERNEMQ_TAG=2.1.1

RUN apt-get update && apt-get install -y --no-install-recommends \
    git ca-certificates make gcc g++ libc6-dev libssl-dev \
    libsnappy-dev zlib1g-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /src
# RUN git clone https://github.com/vernemq/vernemq.git .
RUN git clone https://github.com/trigo-at/vernemq.git .
# RUN git checkout ${VERNEMQ_TAG}

RUN make rel

RUN mkdir -p /vernemq && \
    cp -a _build/default/rel/vernemq/* /vernemq/


# ---------- builder: build vmq_webadmin frontend (dist/) ----------
FROM node:20-bookworm-slim AS webui-builder

RUN corepack enable

WORKDIR /webui
COPY vmq_webadmin/ .
RUN pnpm install
RUN pnpm build


# ---------- runtime (based on official Dockerfile) ----------
FROM debian:bookworm-slim

RUN apt-get update && \
    apt-get -y install bash procps openssl iproute2 curl jq libsnappy-dev net-tools nano && \
    rm -rf /var/lib/apt/lists/* && \
    addgroup --gid 10000 vernemq && \
    adduser --uid 10000 --system --ingroup vernemq --home /vernemq --disabled-password vernemq

WORKDIR /vernemq

# Defaults
ENV DOCKER_VERNEMQ_KUBERNETES_LABEL_SELECTOR="app=vernemq" \
    DOCKER_VERNEMQ_LOG__CONSOLE=console \
    PATH="/vernemq/bin:$PATH" \
    VERNEMQ_VERSION="2.1.1"
COPY --chown=10000:10000 bin/vernemq.sh /usr/sbin/start_vernemq
COPY --chown=10000:10000 bin/join_cluster.sh /usr/sbin/join_cluster

# Copy the build output instead of downloading the binary release
COPY --from=vernemq-builder --chown=10000:10000 /vernemq/ /vernemq/

# Overwrite vm.args with our version (must come after the builder copy)
COPY --chown=10000:10000 files/vm.args /vernemq/etc/vm.args

# Copy web UI dist files into vmq_web_ui priv/www (versioned dir varies)
COPY --from=webui-builder /webui/dist/ /tmp/webui-dist/
RUN set -eux; \
    target_dir="$(ls -d /vernemq/lib/vmq_web_ui-*/priv/www 2>/dev/null | head -n 1)"; \
    if [ -z "${target_dir}" ]; then \
      echo "WARNING: vmq_web_ui priv/www dir not found under /vernemq/lib, skipping web UI copy"; \
    else \
      cp -a /tmp/webui-dist/* "${target_dir}/"; \
    fi; \
    rm -rf /tmp/webui-dist

RUN mkdir -p /vernemq/data/generated.configs /vernemq/log /vernemq/etc && \
    chown -R 10000:10000 /vernemq && \
    ln -s /vernemq/etc /etc/vernemq && \
    ln -s /vernemq/data /var/lib/vernemq && \
    ln -s /vernemq/log /var/log/vernemq

# Ports
# 1883  MQTT
# 8883  MQTT/SSL
# 8080  MQTT WebSockets
# 44053 VerneMQ Message Distribution
# 4369  EPMD - Erlang Port Mapper Daemon
# 8888  Health, API, Prometheus Metrics
# 9100 9101 9102 9103 9104 9105 9106 9107 9108 9109  Specific Distributed Erlang Port Range

EXPOSE 1883 8883 8080 44053 4369 8888 \
       9100 9101 9102 9103 9104 9105 9106 9107 9108 9109


VOLUME ["/vernemq/log", "/vernemq/data", "/vernemq/etc"]

HEALTHCHECK CMD vernemq ping | grep -q pong

USER vernemq

CMD ["start_vernemq"]
