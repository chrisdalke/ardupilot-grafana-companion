FROM buildpack-deps:bookworm-curl AS telegraf-builder

ARG DEBIAN_FRONTEND=noninteractive
RUN echo 'APT::Install-Suggests "0";' >> /etc/apt/apt.conf.d/00-docker && \
  echo 'APT::Install-Recommends "0";' >> /etc/apt/apt.conf.d/00-docker && \
  echo 'APT::Get::Assume-Yes "true";' >> /etc/apt/apt.conf.d/00-docker

SHELL ["/bin/bash", "-c"]
ENTRYPOINT ["/bin/bash", "-c"]
ENV TERM=xterm-color

# Setup build requirements for telegraf
RUN DEBIAN_FRONTEND=noninteractive apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
    iputils-ping \
    snmp \
    procps \
    lm-sensors \
    libcap2-bin \
    git \
    make && \
    rm -rf /var/lib/apt/lists/*

RUN ARCH= && dpkgArch="$(dpkg --print-architecture)" && \
    case "${dpkgArch##*-}" in \
        amd64) ARCH='amd64';; \
        arm64) ARCH='arm64';; \
        armhf) ARCH='armhf';; \
        armel) ARCH='armel';; \
        *)     echo "Unsupported architecture: ${dpkgArch}"; exit 1;; \
    esac && \
    wget -qO- https://go.dev/dl/go1.23.3.linux-${ARCH}.tar.gz | tar -C /usr/local -xzf - && \
    ln -s /usr/local/go/bin/go /usr/bin/go && \
    ln -s /usr/local/go/bin/gofmt /usr/bin/gofmt

WORKDIR /home

# Clone telegraf and install scripts.
RUN git clone https://github.com/influxdata/telegraf.git telegraf && \
    chmod +x ./telegraf/scripts/deb/pre-install.sh && \
    chmod +x ./telegraf/scripts/deb/post-install.sh && \
    ./telegraf/scripts/deb/pre-install.sh && \
    ./telegraf/scripts/deb/post-install.sh

# Build latest release of telegraf.
RUN cd telegraf && \
    go build && \
    make && \
    rm -rf /usr/bin/telegraf && \
    ln -s /home/telegraf/telegraf /usr/bin/telegraf

# Create final image from Ubuntu 22
FROM ubuntu:22.04

# Create directory structure
RUN mkdir -p /home
WORKDIR /home

# Install dependencies
RUN DEBIAN_FRONTEND=noninteractive apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \    
    python3 \
    python3-pip \
    iputils-ping \
    wget \
    gnupg \
    curl \
    snmp \
    dnsutils \
    nano \
    procps \
    lm-sensors \
    libcap2-bin && \
    rm -rf /var/lib/apt/lists/*

# Download the base telegraf installation
# Install telegraf to setup the environment, but delete the base binary.
RUN set -ex && \
    mkdir ~/.gnupg; \
    echo "disable-ipv6" >> ~/.gnupg/dirmngr.conf; \
    for key in \
        9D539D90D3328DC7D6C8D3B9D8FF8E1F7DF8B07E ; \
    do \
        gpg --keyserver hkp://keyserver.ubuntu.com --recv-keys "$key" ; \
    done

ENV TELEGRAF_VERSION=1.33.3
RUN ARCH= && dpkgArch="$(dpkg --print-architecture)" && \
    case "${dpkgArch##*-}" in \
      amd64) ARCH='amd64';; \
      arm64) ARCH='arm64';; \
      armhf) ARCH='armhf';; \
      armel) ARCH='armel';; \
      *)     echo "Unsupported architecture: ${dpkgArch}"; exit 1;; \
    esac && \
    wget --no-verbose https://dl.influxdata.com/telegraf/releases/telegraf_${TELEGRAF_VERSION}-1_${ARCH}.deb.asc && \
    wget --no-verbose https://dl.influxdata.com/telegraf/releases/telegraf_${TELEGRAF_VERSION}-1_${ARCH}.deb && \
    gpg --batch --verify telegraf_${TELEGRAF_VERSION}-1_${ARCH}.deb.asc telegraf_${TELEGRAF_VERSION}-1_${ARCH}.deb && \
    dpkg -i telegraf_${TELEGRAF_VERSION}-1_${ARCH}.deb && \
    rm -f telegraf_${TELEGRAF_VERSION}-1_${ARCH}.deb* && \
    rm -rf /usr/bin/telegraf

# Replace base telegraf installation with custom build
COPY --from=telegraf-builder /home/telegraf/telegraf /usr/bin/telegraf
COPY ./config/telegraf.conf /etc/telegraf/telegraf.conf

CMD ["/usr/bin/telegraf", "--config", "/etc/telegraf/telegraf.conf"]