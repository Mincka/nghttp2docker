FROM ubuntu:20.04
# Properties
LABEL description="nghttpx server for HTTP/1.x traffic interception of HTTP/2.0 only web services"
# Install system packages
RUN apt-get update && apt-get upgrade -y
# Install dependencies to build ngttp2
RUN DEBIAN_FRONTEND="noninteractive" apt-get install -y nghttp2
# Create a dummy TLS certificate for TLS front-end support
RUN openssl req -new -newkey rsa:4096 -days 365 -nodes -x509 \
    -subj "/C=US/ST=Denial/L=Springfield/O=Dis/CN=www.example.com" \
    -keyout host.key -out host.crt
# Expose two ports to support TLS and non-TLS front-end at the same time
EXPOSE 8000
EXPOSE 8001