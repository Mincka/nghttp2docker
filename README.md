# Proxying HTTP/2 only web services with nghttpx
How-to and Docker images to make testing of HTTP/2 only web services easy.

## Disclaimer
This content does not present new techniques or tools. It is inspired from other articles published a long time ago. Its purpose is to provide a quick setup to go on with this very special situation without much trouble.

## Introduction
HTTP/2 has been available for years and its popularity is still increasing slowly. [A recent review](https://www.linkedin.com/pulse/why-do-only-3-top-1000-websites-use-http2-server-push-samir-jafferali/) showed that 685 of the Alexa's top 1000 sites supports HTTP/2, without dropping support for HTTP/1.0 & 1.1. [According to W3Techs](https://w3techs.com/technologies/details/ce-http2), as of June 2020, 46.2% of the top 10 million websites supported HTTP/2.

I did not find any site that would accept only HTTP/2 in top 50k, but people starts to report this kind of setup in special occasions like in IoT, as reported by [NCCGroup](https://www.nccgroup.com/uk/about-us/newsroom-and-events/blogs/2018/may/testing-http2-only-web-services/) in 2018.

In these cases, it could be annoying for pentesters who need to intercept the traffic or use tools that are not able to use HTTP/2, without any option to downgrade to HTTP/1.x

## Use case
Let's say you want to inspect traffic between the browser and a website called `http2only.com`. It works fine in any modern browser but as soon as you want to intercept the traffic using Burp, OWASP Zap or Fiddler, the connection cannot be established.

So it's 2020 and HTTP/2 is on Burp's roadmap but still, you need to play with `http2only.com` right now, with all of your favorite attack & fuzzing tools.

## nghttpx

[nghttpx](https://nghttp2.org/documentation/nghttpx.1.html) is a tool provided with the [nghttp2 library](https://nghttp2.org). We will use it to virtually downgrade the connection to HTTP/1.x so the requests can be intercepted and replayed by proxy tools.

`CLIENT:[HTTP1] <–> [HTTP1]:ATTACKER 
PROXY:[HTTP1] <–> [HTTP1]:nghttpx:[HTTP2] <-> [HTTP2]:SERVER`

## General advices
We want to be the most transparent possible during the traffic interception. All the options that could alter the headers or the traffic downstream or upstream should be avoided.

This configuration will allow you to browse a HTTP/2 website (with mandatory TLS) as a HTTP/1.1 one, even without TLS. I don't recommend to use a different scheme however. You will break basic features like session cookies with `secure` tag.

Moreover, I don't recommend accessing the website using the IP address of the proxy directly. If the website relies on virtual hosts, or if URLs are hardcoded somewhere, it will break at some point. Use the `hosts` configuration, as describe below. 

Finally, I prefer to use patterns to avoid unwanted traffic going through the proxy hitting the targeted server. It may happen if you use the proxy locally. Tools on your workstation could leak information to the target.

## Setup

### nghttp2
Install the package available for your platform or use the Docker images provided here.

`apt install nghttp2`

### Host redirection
We will consider that you will run the proxy locally on your workstation.

Add entries to your `hosts` [file](https://en.wikipedia.org/wiki/Hosts_(file)) to redirect the domains and subdomains to `127.0.0.1`. 

```
127.0.0.1 http2only.com
127.0.0.1 www.http2only.com
```

If you use DNS over HTTPS (DoH), make sure that DoH is not enabled or that exceptions are added in the browser or at OS level. 

### Available ports (if proxy is used locally)
Make sure that ports 80 and 443 are not already used by other processes.

### Get a TLS certificate ready
If you **don't** use the provided Docker containers, you will have to generate a TLS certificate yourself.

`
openssl req -new -newkey rsa:4096 -days 365 -nodes -x509 \
    -subj "/C=US/ST=Denial/L=Springfield/O=Dis/CN=www.example.com" \
    -keyout host.key -out host.crt``
`

### Configuration file
I recommend to use the configuration file as many arguments are required and the command line can be very messy.

The most important settings are `backend` settings. The others are already documented in the full example and in the official documentation. You may have multiple lines for `backend`, one per IP address of your targets.

You don't need to change `frontend` settings if you use Docker images because the port mapping is done when you start the container.

#### Back-end settings
Add the original IP address for each targeted domain and subdomain. Wildcard is possible if IP address is the same but be careful for the proper syntax.

In this example, `http2only.com` and `www.http2only.com` point to `52.186.121.82`. 
```
# Match https://http2only.com and https://www.http2only.com
backend=52.186.121.82,443;http2only.com:www.http2only.com;proto=h2;tls;upgrade-scheme

# Match https://example.com, https://www1.example.com, https://www2.example.com...
# *example.com alone will not match https://www1.example.com
backend=45.12.103.91,443;example.com:*.example.com;proto=h2;tls;upgrade-scheme
```

## Run

### Command line

If you use the package for your distribution, make sure to comment out the lines in the default configuration file (e.g. `/etc/nghttpx/nghttpx.conf`). You may avoid some conflicts if you use the arguments of the command line.

#### Using the configuration file
`nghttpx --conf /nghttpx/nghttpx.conf`

#### Using the arguments
A basic equivalent of the provided configuration file.

`
nghttpx -k -f'*,80;no-tls' -f'*,443' -b'52.186.121.82,443;http2only.com:www.http2only.com;proto=h2;tls;upgrade-scheme' -b'127.0.0.1,1337' --no-ocsp host.key host.crt --workers=5 --no-via --no-server-rewrite --no-location-rewrite --no-add-x-forwarded-proto --no-strip-incoming-x-forwarded-proto -L INFO
`

## Docker images
Three images are available to make it easy to use the proxy on any platform.

### Docker Hub
On [Docker Hub](https://hub.docker.com/r/jehrhart/nghttp2docker). Nothing to build, ready to use. Clone this repo (or just get `nghttpx.conf`), edit the `backend` setting, update your `hosts` file and run this. It is based on the "`package`" image (official nghttp2 package for Ubuntu 20.04).

`
docker run --rm --name nghttpx -p 80:8000 -p 443:8001 -v ${PWD}:/nghttpx -it jehrhart/nghttp2docker nghttpx --conf /nghttpx/nghttpx.conf
`

### Build
#### nghttp2 distro package
Use the current distro package on Ubuntu 20.04.

`docker build --pull --rm -t nghttp2docker:latest "package"`

#### nghttp2 stable release
Build from a release package of nghttp2 (specify the version).

`docker build --pull --rm -t nghttp2docker:latest "stable"`

#### nghttp2 Repository
Build the binaries against the latest code source of nghttp2.

`docker build --pull --rm -t nghttp2docker:latest "master"`

### Run
You can run the container and execute `nghttpx` from it, with the configuration file you provide locally or the full command line, as describe earlier.

`docker run --rm --name nghttpx -p 80:8000 -p 443:8001 -v ${PWD}:/nghttpx -it nghttp2docker nghttpx --conf /nghttpx/nghttpx.conf`

## References and alternatives
* Original idea and an even more complex setup: https://www.nccgroup.com/uk/about-us/newsroom-and-events/blogs/2018/may/testing-http2-only-web-services/
* https://tosebro.hatenablog.com/entry/2015/04/20/235043
* https://security.stackexchange.com/questions/205388/how-to-mitm-http-2-traffic
* https://www.dinosec.com/docs/WAMPT2-PST_v1.0.pdf

## License
Shield: [![CC BY 4.0][cc-by-shield]][cc-by]

This work is licensed under a [Creative Commons Attribution 4.0 International
License][cc-by].

[![CC BY 4.0][cc-by-image]][cc-by]

[cc-by]: http://creativecommons.org/licenses/by/4.0/
[cc-by-image]: https://i.creativecommons.org/l/by/4.0/88x31.png
[cc-by-shield]: https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg

