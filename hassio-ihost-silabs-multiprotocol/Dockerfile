ARG BUILD_FROM
ARG BUILD_ARCH
FROM --platform=linux/amd64 debian:trixie AS cross-builder-base

ENV \
    LANG="C.UTF-8" \
    DEBIAN_FRONTEND="noninteractive" \
    CURL_CA_BUNDLE="/etc/ssl/certs/ca-certificates.crt"

WORKDIR /usr/src

# Allow to reuse downloaded packages (these are only staged build images)
# hadolint ignore=DL3009
RUN \
    set -x \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
       bash \
       curl \
       ca-certificates \
       build-essential \
       git

FROM --platform=linux/amd64 cross-builder-base AS cross-builder-armv7

COPY debian-armv7.cmake /usr/src/debian.cmake

ENV DEBIAN_ARCH=armhf
ENV DEBIAN_CROSS_PREFIX=arm-linux-gnueabihf
ENV SLC_ARCH=linux_arch_32

RUN \
    set -x \
    && dpkg --add-architecture armhf \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
       crossbuild-essential-armhf

FROM --platform=linux/amd64 cross-builder-base AS cross-builder-aarch64

COPY debian-arm64.cmake /usr/src/debian.cmake

ENV DEBIAN_ARCH=arm64
ENV DEBIAN_CROSS_PREFIX=aarch64-linux-gnu
ENV SLC_ARCH=zigbee_arm64

RUN \
    set -x \
    && dpkg --add-architecture arm64 \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
       crossbuild-essential-arm64

FROM --platform=linux/amd64 cross-builder-base AS cross-builder-amd64

COPY debian-amd64.cmake /usr/src/debian.cmake

ENV DEBIAN_ARCH=amd64 \
        DEBIAN_CROSS_PREFIX=x86_64-linux-gnu \
        SLC_ARCH=zigbee_x86_64

RUN \
    set -x \
    && dpkg --add-architecture amd64 \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
       crossbuild-essential-amd64


FROM --platform=linux/amd64 cross-builder-${BUILD_ARCH} AS cpcd-builder

ARG CPCD_VERSION
ENV CPCD_DIR=/usr/src/cpc-daemon
RUN \
    set -x \
    && apt-get install -y --no-install-recommends \
       cmake \
       "libmbedtls-dev:${DEBIAN_ARCH}" \
       "libmbedtls21:${DEBIAN_ARCH}" \
    && git clone --depth 1 -b "${CPCD_VERSION}" \
       https://github.com/SiliconLabs/cpc-daemon.git \
    && mkdir $CPCD_DIR/build && cd $CPCD_DIR/build \
    && cmake ../ \
       -DCMAKE_TOOLCHAIN_FILE=../debian.cmake \
       -DENABLE_ENCRYPTION=FALSE \
    && make \
    && make install

FROM --platform=linux/amd64 cross-builder-${BUILD_ARCH} AS zigbeed-builder

ARG GECKO_SDK_VERSION

RUN \
    set -x \
    && apt-get install -y --no-install-recommends \
       python3 \
       python3-jinja2 \
       python3-pip \
       openjdk-21-jre \
       git-lfs \
       unzip \
    && curl -O https://www.silabs.com/documents/login/software/slc_cli_linux.zip \
    && unzip slc_cli_linux.zip \
    && cd slc_cli/ && chmod +x slc
 
ENV PATH="/usr/src/slc_cli/:$PATH"

# Copy local gecko_sdk instead of cloning (saves time/bandwidth)
COPY gecko_sdk /usr/src/gecko_sdk

# zigbeed links against libcpc.so
COPY --from=cpcd-builder /usr/local/ /usr/${DEBIAN_CROSS_PREFIX}/

RUN \
    set -x \
    && cd gecko_sdk \
    && GECKO_SDK=$(pwd) \
    && slc signature trust --sdk=${GECKO_SDK} \
    && cd protocol/zigbee \
    && sed -i '/zigbee_recommended_linux_arch/d' app/projects/zigbeed/zigbeed.slcp \
    && slc generate \
       --sdk=${GECKO_SDK} \
       --with="${SLC_ARCH}" \
       --project-file=$(pwd)/app/projects/zigbeed/zigbeed.slcp \
       --export-destination=$(pwd)/app/projects/zigbeed/output \
       --copy-proj-sources \
    && cd app/projects/zigbeed/output \
    && make -f zigbeed.Makefile \
        AR="${DEBIAN_CROSS_PREFIX}-ar" \
        CC="${DEBIAN_CROSS_PREFIX}-gcc" \
        LD="${DEBIAN_CROSS_PREFIX}-gcc" \
        CXX="${DEBIAN_CROSS_PREFIX}-g++" \
        C_FLAGS="-std=gnu99 -DEMBER_MULTICAST_TABLE_SIZE=16" \
        debug

# OTBR builder stage - based on mainline otbr approach
FROM --platform=linux/amd64 cross-builder-${BUILD_ARCH} AS otbr-builder

ARG OTBR_VERSION

ENV BORDER_ROUTING=1
ENV BACKBONE_ROUTER=1
ENV PLATFORM=debian
ENV RELEASE=1
ENV WEB_GUI=1
ENV REST_API=1
ENV DOCKER=1

COPY otbr-patches/0001-Avoid-writing-to-system-console.patch /usr/src/
COPY otbr-patches/0001-channel-monitor-disable-by-default.patch /usr/src/
COPY otbr-patches/0001-rest-support-deleting-the-dataset.patch /usr/src/
COPY otbr-patches/0002-spinel-Clear-source-match-tables-before-restoring.patch /usr/src/

WORKDIR /usr/src
RUN \
    set -x \
    && apt-get update \
    # The OTBR build script tries to run `sudo`
    && apt-get install -y --no-install-recommends sudo git \
    && git clone --depth 1 -b main https://github.com/openthread/ot-br-posix.git /usr/src/ot-br-posix \
    && cd /usr/src/ot-br-posix \
    && git fetch origin ${OTBR_VERSION} \
    && git checkout ${OTBR_VERSION} \
    && git submodule update --init \
    # Bootstrap downloads a few more repos
    && ./script/bootstrap \
    # Remove libsystemd-dev to avoid unnecessary linkage
    && apt-get purge -y libsystemd-dev

WORKDIR /usr/src/ot-br-posix
RUN \
    set -x \
    && patch -p1 < /usr/src/0001-Avoid-writing-to-system-console.patch \
    && patch -p1 < /usr/src/0001-rest-support-deleting-the-dataset.patch \
    && ( \
        cd third_party/openthread/repo \
        && patch -p1 < /usr/src/0001-channel-monitor-disable-by-default.patch \
        && patch -p1 < /usr/src/0002-spinel-Clear-source-match-tables-before-restoring.patch \
    ) \
    && ./script/cmake-build \
        -DBUILD_TESTING=OFF \
        -DCMAKE_INSTALL_PREFIX=/usr \
        -DOTBR_FEATURE_FLAGS=ON \
        -DOTBR_DNSSD_DISCOVERY_PROXY=ON \
        -DOTBR_SRP_ADVERTISING_PROXY=ON \
        -DOTBR_MDNS=mDNSResponder \
        -DOTBR_VERSION= \
        -DOT_PACKAGE_VERSION= \
        -DOTBR_DBUS=OFF \
        -DOT_POSIX_RCP_BUS_UART=ON \
        -DOT_LINK_RAW=1 \
        -DOTBR_VENDOR_NAME="Home Assistant" \
        -DOTBR_PRODUCT_NAME="Silicon Labs Multiprotocol" \
        -DOTBR_WEB=ON \
        -DOTBR_BORDER_ROUTING=ON \
        -DOTBR_REST=ON \
        -DOTBR_BACKBONE_ROUTER=ON \
        -DOTBR_TREL=ON \
        -DOTBR_NAT64=ON \
        -DOT_POSIX_NAT64_CIDR="192.168.255.0/24" \
        -DOTBR_DNS_UPSTREAM_QUERY=ON \
        -DOT_CHANNEL_MONITOR=ON \
        -DOT_COAP=OFF \
        -DOT_COAPS=OFF \
        -DOT_DNS_CLIENT_OVER_TCP=OFF \
        -DOT_THREAD_VERSION=1.4 \
        -DOT_RCP_RESTORATION_MAX_COUNT=2 \
    && cd /usr/src/ot-br-posix/build/otbr \
    && ninja install

FROM $BUILD_FROM

ARG UNIVERSAL_SILABS_FLASHER_VERSION

RUN \
    set -x \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
       python3 \
       python3-pip \
       netcat-openbsd \
       socat \
       iproute2 \
       iputils-ping \
       ipset \
       iptables \
       libreadline8 \
       libncurses6 \
       libmbedtls21 \
    && pip install --break-system-packages --no-cache-dir \
       universal-silabs-flasher==${UNIVERSAL_SILABS_FLASHER_VERSION} \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /usr/src/*

# Setup routing table for OpenThread
RUN \
    mkdir -p /etc/iproute2 \
    && echo "88 openthread" >> /etc/iproute2/rt_tables

COPY --from=otbr-builder /usr /usr
COPY --from=cpcd-builder /usr/local/ /usr/local/
COPY --from=zigbeed-builder \
     /usr/src/gecko_sdk/protocol/zigbee/app/projects/zigbeed/output/build/debug/zigbeed /usr/local/bin

COPY rootfs /

ENV \
    S6_STAGE2_HOOK=/etc/s6-overlay/scripts/enable-check.sh

HEALTHCHECK --interval=10s --start-period=120s CMD [ "$(s6-svstat -u /run/service/zigbeed)" = "true" ]

# use s6-overlay as init system
WORKDIR /
LABEL org.opencontainers.image.source="https://github.com/bepvte/hassio-ihost-addon"
ENTRYPOINT ["/init"]
