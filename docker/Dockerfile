FROM golang:1.24 AS compiler
WORKDIR /go

RUN apt-get update && apt-get install -y --no-install-recommends \
      git make bash build-essential \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

RUN git clone --depth=1 https://github.com/amnezia-vpn/amneziawg-tools.git \
    && git clone --depth=1 https://github.com/amnezia-vpn/amneziawg-go.git

RUN cd /go/amneziawg-tools/src \
    && make

RUN cd /go/amneziawg-go && \
    go get -u ./... && \
    go mod tidy && \
    make && \
    chmod +x /go/amneziawg-go/amneziawg-go /go/amneziawg-tools/src/wg /go/amneziawg-tools/src/wg-quick/linux.bash
RUN echo "DONE AmneziaWG"

### INTERMEDIATE STAGE
FROM scratch AS bins
COPY --from=compiler /go/amneziawg-go/amneziawg-go /amneziawg-go
COPY --from=compiler /go/amneziawg-tools/src/wg /awg
COPY --from=compiler /go/amneziawg-tools/src/wg-quick/linux.bash /awg-quick

# FINAL STAGE
FROM alpine:latest
LABEL maintainer="dselen@nerthus.nl"

COPY --from=bins /amneziawg-go /usr/bin/amneziawg-go
COPY --from=bins /awg /usr/bin/awg
COPY --from=bins /awg-quick /usr/bin/awg-quick

# Declaring environment variables, change Peernet to an address you like, standard is a 24 bit subnet.
ARG wg_net="10.0.0.1" \
  wg_port="51820"

# Following ENV variables are changable on container runtime because /entrypoint.sh handles that. See compose.yaml for more info.
ENV TZ="Europe/Amsterdam" \
  global_dns="9.9.9.9" \
  wgd_port="10086" \
  public_ip=""

# Doing package management operations, such as upgrading
RUN apk update \
  && apk add --no-cache bash git tzdata \
    iptables ip6tables openrc curl wireguard-tools \
    sudo py3-psutil py3-bcrypt \
  && apk upgrade

# Using WGDASH -- like wg_net functionally as a ARG command. But it is needed in entrypoint.sh so it needs to be exported as environment variable.
ENV WGDASH=/opt/wgdashboard

# Removing the Linux Image package to preserve space on the image, for this reason also deleting apt lists, to be able to install packages: run apt update.

# Doing WireGuard Dashboard installation measures. Modify the git clone command to get the preferred version, with a specific branch for example.
RUN mkdir /data \
  && mkdir /configs \
  && mkdir -p ${WGDASH}/src \
  && mkdir -p /etc/amnezia/amneziawg
COPY ./src ${WGDASH}/src

# Generate basic WireGuard interface. Echoing the WireGuard interface config for readability, adjust if you want it for efficiency.
# Also setting the pipefail option, verbose: https://github.com/hadolint/hadolint/wiki/DL4006.
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
RUN out_adapt=$(ip -o -4 route show to default | awk '{print $NF}') \
  && echo -e "[Interface]\n\
Address = ${wg_net}/24\n\
PrivateKey =\n\
PostUp = iptables -t nat -I POSTROUTING 1 -s ${wg_net}/24 -o ${out_adapt} -j MASQUERADE\n\
PostUp = iptables -I FORWARD -i wg0 -o wg0 -j DROP\n\
PreDown = iptables -t nat -D POSTROUTING -s ${wg_net}/24 -o ${out_adapt} -j MASQUERADE\n\
PreDown = iptables -D FORWARD -i wg0 -o wg0 -j DROP\n\
ListenPort = ${wg_port}\n\
SaveConfig = true\n\
DNS = ${global_dns}" > /configs/wg0.conf.template \
  && chmod 600 /configs/wg0.conf.template

# Defining a way for Docker to check the health of the container. In this case: checking the gunicorn process.
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD sh -c 'pgrep gunicorn > /dev/null && pgrep tail > /dev/null' || exit 1

# Copy the basic entrypoint.sh script.
COPY ./docker/entrypoint.sh /entrypoint.sh

# Exposing the default WireGuard Dashboard port for web access.
EXPOSE 10086
WORKDIR $WGDASH

ENTRYPOINT ["/bin/bash", "/entrypoint.sh"]