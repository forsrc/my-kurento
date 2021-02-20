<a href="https://www.kurento.org/">
    <img src="https://secure.gravatar.com/avatar/21a2a12c56b2a91c8918d5779f1778bf?s=120">
</a>



# Kurento Media Server (Experimental branches)

[![FIWARE Chapter](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/media-streams.svg)](https://www.fiware.org/developers/catalogue/)
[![License badge](https://img.shields.io/badge/license-Apache2-orange.svg)](http://www.apache.org/licenses/LICENSE-2.0)
[![Build Status](https://ci.openvidu.io/jenkins/buildStatus/icon?job=Development/kurento_media_server_merged_xenial)]()
[![Docker badge](https://img.shields.io/docker/pulls/fiware/orion.svg)](https://hub.docker.com/r/kurento/kurento-media-server)
[![Support badge]( https://img.shields.io/badge/tag-Kurento-orange.svg?logo=stackoverflow)](http://stackoverflow.com/questions/tagged/kurento)
<br/>
[![Documentation badge](https://readthedocs.org/projects/fiware-orion/badge/?version=latest)](https://doc-kurento.readthedocs.io/)
[![FIWARE member status](https://nexus.lab.fiware.org/static/badges/statuses/kurento.svg)](https://www.fiware.org/developers/catalogue/)



## About this image

Image source: [kurento-media-server/Dockerfile](https://github.com/Kurento/kurento-docker/blob/master/kurento-media-server/Dockerfile).

This Docker image can be used to run Kurento Media Server (*KMS*) on any **x86** platform. It cannot be used on other architectures, such as ARM. These are the exact contents of the image:

* A local `apt-get` installation of KMS, as described in the [Installation Guide](https://doc-kurento.readthedocs.io/en/latest/user/installation.html#installation-local).
* Debug symbols installed, as described in the [Developer Guide](https://doc-kurento.readthedocs.io/en/latest/dev/dev_guide.html#dev-dbg). This allows getting useful stack traces in case the process crashes. If this happens, please [report a bug](https://github.com/Kurento/bugtracker/issues).
* All **default settings** from the local installation, as found in `/etc/kurento/`.



## Working with this image

The [Kurento Project](https://www.kurento.org/) provides this Docker image as a nice *all-in-one* package for introductory purposes. It comes with default settings, which is enough to let you try the [Kurento Tutorials].

For *real-world* application development, developers are encouraged to [base FROM](https://docs.docker.com/engine/reference/builder/#from) this Docker image and build their own, with any customizations that they need or want. That's the nice thing about how Docker containers operate! You can build your own images based on the previous work of others.

Running a Docker container **won't modify your host system** and **won't create new files** or anything like that, at least by default. This is part of how Docker containers work, and is important to keep in mind for certain cases. For example, when using the [RecorderEndpoint](https://doc-kurento.readthedocs.io/en/latest/_static/client-javadoc/org/kurento/client/RecorderEndpoint.html) (which creates new files in the local filesystem) some users might be wondering where are the recordings being stored; the answer is *inside the container*.

If you need to insert or extract files from a Docker container, there is a variety of methods: You could use a [bind mount](https://docs.docker.com/storage/bind-mounts/), a [volume](https://docs.docker.com/storage/volumes/), [export](https://docs.docker.com/engine/reference/commandline/container_export/) some files from an already existing container, change your [ENTRYPOINT](https://docs.docker.com/engine/reference/run/#entrypoint-default-command-to-execute-at-runtime) to generate the files at startup, or customize this Docker image to introduce any desired changes.



## Running Kurento Media Server

Running a Docker container is a very customizable operation, so you'll want to read the [Docker run reference](https://docs.docker.com/engine/reference/run/) and find out the command options that are needed for your project.

This is a good starting point, which runs the latest Kurento Media Server image with default options:

```
$ docker pull kurento/kurento-media-server-exp:<BranchName>

$ docker run -d --name kms --network host \
    kurento/kurento-media-server-exp:<BranchName>
```

By default, KMS listens on the port **8888**. Clients wanting to control the media server using the [Kurento Protocol](https://doc-kurento.readthedocs.io/en/latest/features/kurento_protocol.html) should open a WebSocket connection to that port, either directly or by means of one of the provided [Kurento Client] SDKs.

Once the container is running, you can get its log output with the [docker logs](https://docs.docker.com/engine/reference/commandline/logs/) command:

```
$ docker logs --follow kms >"kms-$(date '+%Y%m%dT%H%M%S').log" 2>&1
```

To check whether KMS is up and listening for connections, use the following command:

```
$ curl \
    --include \
    --header "Connection: Upgrade" \
    --header "Upgrade: websocket" \
    --header "Host: 127.0.0.1:8888" \
    --header "Origin: 127.0.0.1" \
    http://127.0.0.1:8888/kurento
```

You should get a response similar to this one:

```
HTTP/1.1 500 Internal Server Error
Server: WebSocket++/0.7.0
```

Ignore the "*Server Error*" message: this is expected, and it actually proves that KMS is up and listening for connections.

The [health checker script](https://github.com/Kurento/kurento-docker/blob/master/kurento-media-server/healthchecker.sh) inside this Docker image does something very similar in order to check if the container is healthy.



### Why host networking?

Notice how our suggested `docker run` command uses `--network host`? Using [Host Networking](https://docs.docker.com/network/host/) is recommended for software like proxies and media servers, because otherwise publishing large ranges of container ports would consume a lot of memory. You can read more about this issue in our [Troubleshooting Guide](https://doc-kurento.readthedocs.io/en/latest/user/troubleshooting.html#troubleshooting-docker-network-host).

The Host Networking driver **only works on Linux hosts**, so if you are using Docker for Mac or Windows then you'll need to understand that the Docker network gateway acts as a NAT between your host and your container. To use KMS without STUN (e.g. if you are just testing some of the [Kurento Tutorials]) you'll need to publish all required ports where KMS will listen for incoming data.

For example, to have KMS listening on the UDP port range **[5000, 5050]** (thus allowing incoming data on those ports), plus the TCP port **8888** for the [Kurento Client] connection:

```sh
$ docker run --rm \
    -p 8888:8888/tcp \
    -p 5000-5050:5000-5050/udp \
    -e KMS_MIN_PORT=5000 \
    -e KMS_MAX_PORT=5050 \
    kurento/kurento-media-server:latest
```



## Configuration

For convenience reasons, this Docker image accepts some environment variables that can be used to create containers with the most commonly used settings for Kurento Media Server.

If this is not flexible enough, you can always use a [bind-mount](https://docs.docker.com/storage/bind-mounts/) or [volume](https://docs.docker.com/storage/volumes/) with a different set of configuration files in `/etc/kurento/`. Of course, you can always also base upon this Docker image and extend it to add your own configuration, modifications, or custom plugins, as desired.



### Debug logging: GST_DEBUG

KMS uses the environment variable `GST_DEBUG` to define the debug level of all underlying modules. Check the docs section about [Debug Logging](https://doc-kurento.readthedocs.io/en/stable/features/logging.html) for more information about this and other environment variables.

Set this variable to change the verbosity level of the log messages generated by KMS:

```
$ docker run [...] \
    -e GST_DEBUG="Kurento*:5" \
    kurento/kurento-media-server-exp:<BranchName>
```



### STUN/TURN server: KMS_STUN_IP, KMS_STUN_PORT, KMS_TURN_URL

Read [When are STUN and TURN needed?](https://doc-kurento.readthedocs.io/en/stable/user/faq.html#faq-stun-needed) to learn about when you might need to use these, and [STUN/TURN server install](https://doc-kurento.readthedocs.io/en/stable/user/installation.html#installation-stun-turn) for guidance on how to install your own STUN/TURN server.

The details of a STUN/TURN server can be provided to KMS via the variables `KMS_STUN_IP`, `KMS_STUN_PORT`, and `KMS_TURN_URL`.

Equivalent to KMS options `stunServerAddress`, `stunServerPort`, and `turnURL`, in the settings file `/etc/kurento/modules/kurento/WebRtcEndpoint.conf.ini`.



### Network interface selection: KMS_NETWORK_INTERFACES

To specify the network interface name(s) that KMS should use to communicate from within Docker, you can do so with the variable `KMS_NETWORK_INTERFACES`.

Equivalent to KMS option `networkInterfaces` in the settings file `/etc/kurento/modules/kurento/WebRtcEndpoint.conf.ini`.

Client API docs: [Java](https://doc-kurento.readthedocs.io/en/stable/_static/client-javadoc/org/kurento/client/WebRtcEndpoint.html#setNetworkInterfaces-java.lang.String-), [Javascript](https://doc-kurento.readthedocs.io/en/stable/_static/client-jsdoc/module-elements.WebRtcEndpoint.html#setNetworkInterfaces).



### External IP address: KMS_EXTERNAL_IPV4, KMS_EXTERNAL_IPV6

Only for advanced users who know what they are doing.

You can use the variables `KMS_EXTERNAL_IPV4` or `KMS_EXTERNAL_IPV6` to force mangling all of the Kurento IPv4 or IPv6 ICE candidates to contain the given address. This can speed up WebRTC connection establishment in scenarios where the external or public IP is already well known, also having the benefit that STUN won't be needed *for the media server*.

If the special value `auto` is used, then the container will auto-discover its own public IP address by performing a DNS query to some of the well established providers (OpenDNS, Google, Cloudflare). In cases where these services are not reachable, the external IP parameters are left unset.

Equivalent to KMS options `externalIPv4` and `externalIPv6` in the settings file `/etc/kurento/modules/kurento/WebRtcEndpoint.conf.ini`.

Client API docs: [Java](https://doc-kurento.readthedocs.io/en/stable/_static/client-javadoc/org/kurento/client/WebRtcEndpoint.html#setExternalIPv4-java.lang.String-), [Javascript](https://doc-kurento.readthedocs.io/en/stable/_static/client-jsdoc/module-elements.WebRtcEndpoint.html#setExternalIPv4).



#### RTP port bindings: KMS_MIN_PORT, KMS_MAX_PORT

You can configure the minimum and maximum ports that Kurento Media Server will open (bind to) in order to receive RTP packets from remote peers. This affects the operation of both RtpEndpoint and WebRtcEndpoint. The variables to set are `KMS_MIN_PORT` and `KMS_MAX_PORT`.

Equivalent to KMS options `minPort` and `maxPort`, in the settings file `/etc/kurento/modules/kurento/BaseRtpEndpoint.conf.ini`.



### Maximum Transmission Unit (MTU): KMS_MTU

Only for advanced users who know what they are doing.

It is possible to specify a network MTU different than the default of 1200 Bytes, by using the variable `KMS_MTU`.

Equivalent to KMS option `mtu` in the settings file `/etc/kurento/modules/kurento/BaseRtpEndpoint.conf.ini`.

Client API docs: [Java](https://doc-kurento.readthedocs.io/en/stable/_static/client-javadoc/org/kurento/client/BaseRtpEndpoint.html#setMtu-int-), [Javascript](https://doc-kurento.readthedocs.io/en/stable/_static/client-jsdoc/module-core_abstracts.BaseRtpEndpoint.html#setMtu).



# About Kurento

Kurento is an open source software project providing a platform suitable for creating modular applications with advanced real-time communication capabilities. For knowing more about Kurento, please visit the Kurento project website: https://www.kurento.org/.

Kurento is part of [FIWARE]. For further information on the relationship of FIWARE and Kurento check the [Kurento FIWARE Catalog Entry].

Kurento has been rated within [FIWARE] as follows:

-   **Version Tested:**
    ![ ](https://img.shields.io/badge/dynamic/json.svg?label=Version&url=https://fiware.github.io/catalogue/json/kurento.json&query=$.version&colorB=blue)
-   **Documentation:**
    ![ ](https://img.shields.io/badge/dynamic/json.svg?label=Completeness&url=https://fiware.github.io/catalogue/json/kurento.json&query=$.docCompleteness&colorB=blue)
    ![ ](https://img.shields.io/badge/dynamic/json.svg?label=Usability&url=https://fiware.github.io/catalogue/json/kurento.json&query=$.docSoundness&colorB=blue)
-   **Responsiveness:**
    ![ ](https://img.shields.io/badge/dynamic/json.svg?label=Time%20to%20Respond&url=https://fiware.github.io/catalogue/json/kurento.json&query=$.timeToCharge&colorB=blue)
    ![ ](https://img.shields.io/badge/dynamic/json.svg?label=Time%20to%20Fix&url=https://fiware.github.io/catalogue/json/kurento.json&query=$.timeToFix&colorB=blue)
-   **FIWARE Testing:**
    ![ ](https://img.shields.io/badge/dynamic/json.svg?label=Tests%20Passed&url=https://fiware.github.io/catalogue/json/kurento.json&query=$.failureRate&colorB=blue)
    ![ ](https://img.shields.io/badge/dynamic/json.svg?label=Scalability&url=https://fiware.github.io/catalogue/json/kurento.json&query=$.scalability&colorB=blue)
    ![ ](https://img.shields.io/badge/dynamic/json.svg?label=Performance&url=https://fiware.github.io/catalogue/json/kurento.json&query=$.performance&colorB=blue)
    ![ ](https://img.shields.io/badge/dynamic/json.svg?label=Stability&url=https://fiware.github.io/catalogue/json/kurento.json&query=$.stability&colorB=blue)


Kurento is also part of the [NUBOMEDIA] research initiative.

[FIWARE]: http://www.fiware.org
[Kurento FIWARE Catalog Entry]: http://catalogue.fiware.org/enablers/stream-oriented-kurento
[NUBOMEDIA]: http://www.nubomedia.eu



## Documentation

The Kurento project provides detailed [documentation] including tutorials, installation and development guides. The [Open API specification], also known as *Kurento Protocol*, is available on [apiary.io].

[documentation]: https://www.kurento.org/documentation
[Open API specification]: https://doc-kurento.readthedocs.io/en/stable/features/kurento_api.html
[apiary.io]: http://docs.streamoriented.apiary.io/



## Useful Links

Usage:

* [Installation Guide](https://doc-kurento.readthedocs.io/en/stable/user/installation.html)
* [Docker Deployment Guide](https://hub.docker.com/r/kurento/kurento-media-server)
* [Contribution Guide](https://doc-kurento.readthedocs.io/en/stable/project/contributing.html)
* [Developer Guide](https://doc-kurento.readthedocs.io/en/stable/dev/dev_guide.html)

Issues:

* [Bug Tracker](https://github.com/Kurento/bugtracker/issues)
* [Support](https://doc-kurento.readthedocs.io/en/stable/user/support.html)

News:

* [Kurento Blog](https://www.kurento.org/blog)
* [Google Groups](https://groups.google.com/forum/#!forum/kurento)
* [Kurento RoadMap](https://github.com/Kurento/kurento-media-server/blob/master/ROADMAP.md)

Training:

* [Kurento Tutorials]



## Source code

All source code belonging to the Kurento project can be found under the [Kurento GitHub organization](https://github.com/Kurento) page.



## Testing

Kurento has a full set of different tests mainly focused in the integrated and system tests, more specifically e2e tests that anyone can run to assess different parts of Kurento, namely functional, stability, tutorials, and API.

In order to assess properly Kurento from a final user perspective, a rich suite of E2E tests has been designed and implemented. To that aim, the Kurento Testing Framework (KTF) has been created. KTF is a part of the Kurento project aimed to carry out end-to-end (E2E) tests for Kurento. KTF has been implemented on the top of two well-known open-source testing frameworks: JUnit and Selenium.

If you want to know more about the Kurento Testing Framework and how to run all the available tests for Kurento you will find more information here: [Kurento Testing](https://doc-kurento.readthedocs.io/en/stable/dev/testing.html).



## Licensing and distribution

```
Copyright 2019 Kurento

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```



<!--- Links --->

[Kurento Client]: https://doc-kurento.readthedocs.io/en/stable/features/kurento_client.html
[Kurento Tutorials]: https://doc-kurento.readthedocs.io/en/stable/user/tutorials.html
