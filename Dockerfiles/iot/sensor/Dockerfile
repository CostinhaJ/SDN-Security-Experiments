FROM ubuntu:jammy AS builder

ARG WOLFSSL_VERSION=v5.9.4-stable
ARG LIBCOAP_VERSION=v4.3.5b

RUN apt update && DEBIAN_FRONTEND=noninteractive apt install -y \
    git \
    build-essential \
    autoconf \
    automake \
    pkg-config \
    libtool \
    xz-utils \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /opt/wolfssl
RUN git clone --depth 1 -b ${WOLFSSL_VERSION} https://github.com/wolfSSL/wolfssl.git . &&\
    ./autogen.sh &&\
    ./configure --prefix=/usr/local \
        --enable-all --enable-dtls13 --enable-dtlscid \
        --enable-static --disable-shared \
        --disable-examples --disable-crypttests \
        CFLAGS="-DBUILD_TLS_PSK_WITH_AES_128_CCM -DHAVE_RPK -DDTLS_CID_MAX_SIZE=8" &&\
    make -j"$(nproc)" &&\
    make install

# --- libcoap with the wolfSSL backend --------------------------------------
WORKDIR /opt/libcoap
RUN git clone --depth 1 -b ${LIBCOAP_VERSION} https://github.com/obgm/libcoap.git .
 
COPY coap-server-mod.c examples/coap-server.c

RUN ./autogen.sh &&\
    ./configure --enable-examples --enable-dtls --with-wolfssl \
        --disable-shared --disable-documentation --disable-tests &&\
    make -j"$(nproc)"

#COPY combined_cycle_power_plant/Fold1_pp.csv.xz Fold1_pp.csv.xz
#RUN xz -v -d Fold1_pp.csv.xz &&\
#    sed -i '1d' Fold1_pp.csv

COPY appliances_energy/energydata_complete.csv.xz energydata_complete.csv.xz
RUN xz -v -d energydata_complete.csv.xz &&\
    sed -i '1d' energydata_complete.csv

FROM ubuntu:jammy

RUN apt update && apt install -y --no-install-recommends \
    wget \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /opt/libcoap/examples/coap-server-wolfssl /opt/coap-server
#COPY --from=builder /opt/libcoap/Fold1_pp.csv /Fold1_pp.csv
COPY --from=builder /opt/libcoap/energydata_complete.csv /energydata_complete.csv

ENTRYPOINT [ "/opt/coap-server", "-v", "7", "-p", "5683" ]
