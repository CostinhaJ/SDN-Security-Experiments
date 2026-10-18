FROM ubuntu:jammy AS coap-builder

ARG WOLFSSL_VERSION=v5.9.4-stable
ARG LIBCOAP_VERSION=v4.3.5b

RUN apt update && DEBIAN_FRONTEND=noninteractive apt install -y \
    git \
    build-essential \
    pkg-config \
    autoconf \
    automake \
    libtool \
    && rm -rf /var/lib/apt/lists/*

# --- wolfSSL com DTLS 1.3 (estático, para o binário não depender de libs) ---
WORKDIR /opt/wolfssl
RUN git clone --depth 1 -b ${WOLFSSL_VERSION} https://github.com/wolfSSL/wolfssl.git . \
    && ./autogen.sh \
    && ./configure --prefix=/usr/local \
        --enable-all --enable-dtls13 --enable-dtlscid \
        --enable-static --disable-shared \
        --disable-examples --disable-crypttests \
        CFLAGS="-DBUILD_TLS_PSK_WITH_AES_128_CCM -DHAVE_RPK -DDTLS_CID_MAX_SIZE=8" \
    && make -j"$(nproc)" \
    && make install

# --- libcoap com backend wolfSSL ---
WORKDIR /opt/libcoap
RUN git clone --depth 1 -b ${LIBCOAP_VERSION} https://github.com/obgm/libcoap.git .

COPY coap-client-mod.c examples/coap-client.c

RUN ./autogen.sh \
    && PKG_CONFIG_PATH=/usr/local/lib/pkgconfig ./configure \
        --enable-examples --enable-dtls --with-wolfssl \
        --disable-shared --disable-documentation --disable-tests \
    && make


# =========================================================================
# Estágio 2: nó Edge (MQTT + TLS pós-quântico + CoAP)
# =========================================================================
FROM ubuntu:jammy

RUN apt update && apt install -y --no-install-recommends \
    iputils-ping \
    sntp \
    python3 \
    python3-pip \
    wget \
    git \
    cmake \
    ninja-build \
    build-essential \
    libssl-dev \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

RUN pip3 install --no-cache-dir \
    paho-mqtt \
    joblib \
    numpy \
    scikit-learn \
    xgboost

ARG LIBOQS_TAG=0.13.0
ARG OQSPROVIDER_TAG=0.9.0

# --- liboqs (biblioteca dos algoritmos pós-quânticos) ---
RUN git clone --depth 1 --branch ${LIBOQS_TAG} https://github.com/open-quantum-safe/liboqs.git /tmp/liboqs \
    && cmake -S /tmp/liboqs -B /tmp/liboqs/build -GNinja \
    && cmake --build /tmp/liboqs/build --parallel \
    && cmake --install /tmp/liboqs/build \
    && rm -rf /tmp/liboqs

# --- oqs-provider (plugin que liga o liboqs ao OpenSSL 3) ---
RUN git clone --depth 1 --branch ${OQSPROVIDER_TAG} https://github.com/open-quantum-safe/oqs-provider.git /tmp/oqs-provider \
    && cmake -S /tmp/oqs-provider -B /tmp/oqs-provider/build -GNinja \
    && cmake --build /tmp/oqs-provider/build --parallel \
    && cmake --install /tmp/oqs-provider/build \
    && rm -rf /tmp/oqs-provider

# --- dizer ao OpenSSL para carregar o provider ---
COPY openssl-oqs.cnf /etc/ssl/openssl-oqs.cnf
ENV OPENSSL_CONF=/etc/ssl/openssl-oqs.cnf

COPY appliances_energy/energydata_complete.csv.xz /energydata_complete.csv.xz
COPY --from=iotsim/certificates:latest /artifacts/pki/iot-sim-ca.crt /iot-sim-ca.crt

# --- CoAP: binário compilado no estágio 1 + chave PSK ---
# O client.py procura 'coap-client' no PATH e em /opt, e lê a PSK de /opt/psk.txt
COPY --from=coap-builder /opt/libcoap/examples/coap-client-wolfssl /opt/coap-client
COPY --from=iotsim/certificates:latest /artifacts/psk/psk.txt /opt/psk.txt

COPY models/decision_model.joblib /models/decision_model.joblib
COPY models/normalization.joblib /models/normalization.joblib

COPY tls_groups.py /tls_groups.py
COPY client.py /client.py
RUN chmod +x /client.py

ENTRYPOINT ["/client.py"]