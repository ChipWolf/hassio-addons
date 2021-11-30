ARG BUILD_FROM=hassioaddons/debian-base:3.0.0
# hadolint ignore=DL3006
FROM ${BUILD_FROM} as base
ARG BUILD_ARCH

ENV BUILD_DEPS \
    autoconf \
    automake \
    autotools-dev \
    build-essential \
    git \
    libantlr3c-dev \
    libasound2-dev \
    libavahi-client-dev \
    libavcodec-dev \
    libavfilter-dev \
    libavformat-dev \
    libavutil-dev \
    libconfig-dev \
    libconfuse-dev \
    libcurl4-gnutls-dev \
    libevent-dev \
    libgcrypt20-dev \
    libgnutls28-dev \
    libjson-c-dev \
    libmxml-dev \
    libplist-dev \
    libplist-dev \
    libpopt-dev \
    libprotobuf-c-dev \
    libpulse-dev \
    libsodium-dev \
    libsoxr-dev \
    libsqlite3-dev \
    libssl-dev \
    libswscale-dev \
    libtool \
    libunistring-dev \
    libwebsockets-dev \
    openjdk-11-jre-headless \
    zlib1g-dev

ENV RUN_DEPS \
    libantlr3c-3.4-0 \
    libasound2 \
    libavahi-client3 \
    libavcodec58 \
    libavfilter7 \
    libavformat58 \
    libavutil56 \
    libconfuse2 \
    libcurl3-gnutls \
    libevent-2.1-6 \
    libevent-pthreads-2.1-6 \
    libgcrypt20 \
    libgnutls30 \
    libjson-c3 \
    libmxml1 \
    libplist3 \
    libprotobuf-c1 \
    libpulse0 \
    libsodium23 \
    libsqlite3-0 \
    libswscale5 \
    libunistring2 \
    libwebsockets8 \
    zlib1g \
    avahi-daemon

RUN apt-get update -y && apt-get install -y --no-install-recommends \
    curl gpg gpg-agent dirmngr apt-transport-https gawk gettext gperf \
    && rm -rf /var/lib/apt/lists/*

FROM base AS builder
ARG BUILD_ARCH
RUN mkdir -p /usr/share/man/man1
RUN apt-get update -y && apt-get install -y --no-install-recommends \
    ${BUILD_DEPS} \
    && rm -rf /var/lib/apt/lists/*

RUN curl -L -o /tmp/antlr-3.4-complete.jar http://www.antlr3.org/download/antlr-3.4-complete.jar \
    && echo '#!/bin/bash' > /usr/local/bin/antlr3 \
    && echo 'exec java -cp /tmp/antlr-3.4-complete.jar org.antlr.Tool "$@"' >> /usr/local/bin/antlr3 \
    && chmod 775 /usr/local/bin/antlr3

WORKDIR /tmp/owntone
RUN git clone --depth 1 https://github.com/owntone/owntone-server.git . \
    && autoreconf -fi \
    && ./configure \
        --enable-itunes \
        --enable-chromecast \
        --with-pulseaudio \
        --without-libspotify \
        --with-libwebsockets \
    && make \
    && make install DESTDIR=/tmp/owntone-install

WORKDIR /tmp/shairport
RUN git clone --depth 1 https://github.com/mikebrady/shairport-sync.git . \
    && autoreconf -i -f \
    && ./configure \
        --with-pa \
        --with-pipe \
        --with-avahi \
        --with-ssl=openssl \
        --with-soxr \
        --with-metadata \
    && make \
    && make install DESTDIR=/tmp/shairport-install

FROM base AS runner
ARG BUILD_ARCH
COPY --from=builder /tmp/owntone-install /
COPY --from=builder /tmp/shairport-install /
RUN apt-get update -y && apt-get install -y --no-install-recommends \
    ${RUN_DEPS} nginx \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /usr/local/etc
RUN sed -i -e 's/\(uid.*=.*\)/uid = "root"/g' owntone.conf \
    && sed -i s#"ipv6 = yes"#"ipv6 = no"#g owntone.conf \
    && sed -i s#/srv/music#/share/owntone/music#g owntone.conf \
    && sed -i s#/usr/local/var/cache/owntone/songs3.db#/share/owntone/cache/songs3.db#g owntone.conf \
    && sed -i s#/usr/local/var/cache/owntone/cache.db#/share/owntone/cache/cache.db#g owntone.conf \
    && sed -i s#/usr/local/var/log/owntone.log#/dev/stdout#g owntone.conf \
    && sed -i "/websocket_port\ =/ s/# *//" owntone.conf \
    && sed -i "/trusted_networks\ =/ s/# *//" owntone.conf \
    && sed -i "/pipe_autostart\ =/ s/# *//" owntone.conf \
    && sed -i "/db_path\ =/ s/# *//" owntone.conf \
    && sed -i "/cache_path\ =/ s/# *//" owntone.conf \
    && sed -i "/airplay_shared/ s/# *//" owntone.conf \
    && sed -i "/control_port\ =/ s/#/ /" owntone.conf \
    && sed -i "/timing_port\ =/ s/#/ /" owntone.conf \
    && sed -i "/timing_port/{N;s/\n#/\n/}" owntone.conf \
    && sed -i "s/\(control_port =\).*/\1 3690/" owntone.conf \
    && sed -i "s/\(timing_port =\).*/\1 3691/" owntone.conf \
    && sed -i "/type\ =/ s/#/ /" owntone.conf \
    && sed -i 's/\(type =\).*/\1 "pulseaudio"/' owntone.conf
RUN mkdir -p /var/log/nginx && ln -sf /dev/stdout /var/log/nginx/access.log && ln -sf /dev/stderr /var/log/nginx/error.log

# Copy root filesystem
COPY rootfs /

# Build arguments
ARG BUILD_DATE
ARG BUILD_REF
ARG BUILD_VERSION

# Labels
LABEL \
    io.hass.name="Forked-daapd Server" \
    io.hass.description="The owntone server program" \
    io.hass.arch="${BUILD_ARCH}" \
    io.hass.type="addon" \
    io.hass.version=${BUILD_VERSION} \
    maintainer="Ulrar" \
    org.label-schema.description="The owntone server program" \
    org.label-schema.build-date=${BUILD_DATE} \
    org.label-schema.name="Forked-daapd Server" \
    org.label-schema.schema-version="1.0" \
    org.label-schema.url="" \
    org.label-schema.usage="" \
    org.label-schema.vcs-ref=${BUILD_REF} \
    org.label-schema.vcs-url="https://github.com/Ulrar/hassio-addons/tree/master/owntone" \
    org.label-schema.vendor="Ulrar addons"
