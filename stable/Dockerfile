FROM ubuntu:20.04
# Properties
LABEL description="nghttpx server for HTTP/1.x traffic interception of HTTP/2.0 only web services"
ARG ngttp2_version="1.41.0"
# Install system packages
RUN apt-get update && apt-get upgrade -y
# Install dependencies to build ngttp2
RUN DEBIAN_FRONTEND="noninteractive" apt-get install -y apt-utils wget g++ make binutils autoconf automake autotools-dev libtool pkg-config zlib1g-dev libcunit1-dev libssl-dev libxml2-dev libev-dev libevent-dev libjansson-dev libc-ares-dev libjemalloc-dev libsystemd-dev cython python3-dev python-setuptools
# Get the ngttp2 realease and unpack it
RUN wget https://github.com/nghttp2/nghttp2/releases/download/v$ngttp2_version/nghttp2-$ngttp2_version.tar.bz2
RUN tar xfv nghttp2-$ngttp2_version.tar.bz2
# Build the libairies and the binaries
WORKDIR /nghttp2-$ngttp2_version
RUN ./configure
RUN make
# Install the binaries
RUN make install
# Update the links to the shared libaries
RUN ldconfig -v
# Create a dummy TLS certificate for TLS front-end support
RUN openssl req -new -newkey rsa:4096 -days 365 -nodes -x509 \
    -subj "/C=US/ST=Denial/L=Springfield/O=Dis/CN=www.example.com" \
    -keyout host.key -out host.crt
# Expose two ports to support TLS and non-TLS front-end at the same time
EXPOSE 8000
EXPOSE 8001