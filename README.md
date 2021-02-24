[![Apache 2.0](https://img.shields.io/github/license/openziti/ziti-tunnel-sdk-c)](https://github.com/openziti/ziti-tunnel-sdk-c/blob/master/LICENSE)
[![Latest Release](https://img.shields.io/github/v/release/openziti/ziti-tunnel-sdk-c)](https://github.com/openziti/ziti-tunnel-sdk-c/releases/latest)
[![Build Status](https://github.com/openziti/ziti-tunnel-sdk-c/workflows/CI%20build/badge.svg)](https://github.com/openziti/ziti-tunnel-sdk-c/actions?query=workflow%3A%22CI+build%22)

# Ziti Tunneler SDK

The Ziti Tunneler SDK provides protocol translation and other common functions
that are useful to Ziti Tunnelers.

## What's a Ziti Tunneler?

Ziti Tunnelers allow pre-existing TCP/IP applications to access or provide
services on a secure Ziti network. A Ziti Tunneler is a Ziti-native application
that communicates with local peers using TCP/IP and proxies the payload to/from
a Ziti service.

Note that embedding the Ziti SDK directly into an application is preferable to
using a Ziti Tunneler, if possible. A Ziti-native application is secured and
encrypted to the farthest edges of the communication path - all the way to the
application's internal buffers. A Ziti Tunneler cannot secure the communication
between itself and the TCP/IP application.

Ziti Tunnelers are intended for situations where going Ziti-native is expensive
or impossible to implement (e.g. a third-party applications or libraries). Ziti
Tunnelers enable standard TCP/IP applications to reap _most_ of the security and
reliability benefits offered by Ziti networks without changing a line of code.

### Running `ziti-edge-tunnel`
Download appropriate version from [Releases](https://github.com/openziti/ziti-tunnel-sdk-c/releases/latest)

#### Linux
Enrollment

```
$ ziti-edge-tunnel enroll -j <enrollment JWT file> -i <output identity file>
```

Run tunnel with default options (DNS configured with systemd-resolved)

```
$ ziti-edge-tunnel run -i <identity file>
```

Sample service file
``` 
[Unit]
Description=Ziti Edge Tunnel
After=network-online.target

[Service]
Type=simple
ExecStart=/opt/ziti/bin/ziti-edge-tunnel run -i /opt/ziti/etc/id.json
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

## What is the Ziti Tunneler SDK?

The Ziti Tunneler SDK provides functionality that is common to Ziti Tunnelers across
supported operating systems:

- Converse with TCP/IP peers
- Map TCP/IP connections to Ziti sessions
- Respond to DNS queries for Ziti service hostnames

A Ziti Tunneler application that uses the Ziti Tunneler SDK only needs to
implement platform-specific functionality, such as creating a virtual network
interface, and providing a user interface.

A set of callback
functions that interact with the specific _ziti-sdk_ that the application
uses (e.g. `ziti-sdk-c`, `ziti-sdk-go`).

The Ziti Tunneler SDK includes an implementation of the required callback
functions for `ziti-sdk-c`. Here's how a minimal tunneler application written
in C could use the Ziti Tunneler SDK:

```c
int main(int argc, char *argv[]) {
    uv_loop_t *nf_loop = uv_default_loop();
    netif_driver tun = tun_open(NULL, 0); /* open a tun device, and */

    if (tun == NULL) {
        fprintf(stderr, "failed to open network interface: %s\n", tun_error);
        return 1;
    }

    tunneler_sdk_options tunneler_opts = {
            .netif_driver = tun,
            .ziti_dial = ziti_sdk_c_dial,
            .ziti_close = ziti_sdk_c_close,
            .ziti_write = ziti_sdk_c_write
    };
    tunneler_context tnlr_ctx = NF_tunneler_init(&tunneler_opts, nf_loop);

    nf_options opts = {
            .init_cb = on_nf_init,
            .config = argv[1],
            .service_cb = on_service,
            .ctx = tnlr_ctx, /* this is passed to the service_cb */
            .refresh_interval = 10,
            .config_types = cfg_types,
    };

    if (NF_init_opts(&opts, nf_loop, NULL) != 0) {
        fprintf(stderr, "failed to initialize ziti\n");
        return 1;
    }

    if (uv_run(nf_loop, UV_RUN_DEFAULT) != 0) {
        fprintf(stderr, "failed to run event loop\n");
        exit(1);
    }

    free(tnlr_ctx);
    return 0;
}
```

Once the Ziti Tunneler SDK is initialized with a network device and ziti-sdk
callbacks, a tunneler application only needs to indiciate which service(s)
that should be  

