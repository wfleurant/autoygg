
[![Go Report Card](https://goreportcard.com/badge/github.com/cure/autoygg)](https://goreportcard.com/report/github.com/cure/autoygg)

Autoygg is a tool to automatically configure an internet egress node on an
[yggdrasil](https://yggdrasil-network.github.io/) network.

There is a server component (autoygg-server) which acts as egress gatekeeper
on an yggdrasil node with internet access (e.g. a VPS in the cloud). It has a
REST api that is consumed by the client component (autoygg-client).

The client can register with the server for internet egress access. When it
gets a successful response from the server, it will configure the local
yggdrasil node to send all internet-bound traffic out via the node that runs
autoygg-server. Because the traffic between the client and server runs over the
yggdrasil network, it is end-to-end encrypted until the point of egress to the
internet.

## Building the software

GNU Make and a Go compiler, version 1.11 or higher are required.

Build server and client software with the *make* command from the root of the
tree. The resulting binaries will be in the

```
cmd/autoygg-client/
cmd/autoygg-server/
```

directories.

It is also possible to build just the client. For example, to build for amd64 (e.g. to run on an APU2 target):

```
$ cd cmd/autoygg-client/
$ make
```

To build for arm64 (e.g. to run on a Raspberry Pi 4):

```
$ cd cmd/autoygg-client/
$ GOARCH=arm64 make
```

## Quick start

Example session:

1) Go to the machine that will run *autoygg-server*.

The `autoygg-server` program will accept parameters in a configuration file named *server.yaml*, which can be located in */etc/autoygg/* or in the directory the *autoygg-server* binary is executed from. A few parameters may be specified as command line arguments. The complete list of parameters can be consulted by running *autoygg-server --help*, e.g.:

```
$ ./autoygg-server --help

autoygg-server provides internet egress for Yggdrasil nodes running autoygg-client.

Options:
      --dumpConfig   dump the configuration that would be used by autoygg-server and exit
      --help         print usage and exit
```

Create a config file at */etc/autoygg/server.yaml*. A sample configuration file is provided in *cmd/autoygg-server/server.yaml*. A minimal configuration could be:

```
---
ListenHost: "the:yggdrasil:ip:address:of:this:machine"
AllowRegistration: true
WhitelistEnabled: false
GatewayTunnelIP: "10.42.0.1"
GatewayTunnelNetmask: "16"
```

Get the value for 'ListenHost' by running

```
$ yggdrasilctl getSelf
```

*WARNING*: in this configuration, this autoygg server will provide internet egress to any client that registers itself. To limit which clients can use the server, change WhitelistEnabled to *true* in server.yaml and create a file named */etc/autoygg/whitelist.yaml*. Add your client yggdrasil IP to that file, e.g. like this:

```
---
Whitelist:
- 200:1234:5678:9000:0000:0000:0000:0001
```

Note: the `autoygg-server` program will automatically refresh all its config files when they change. There is no need to restart it after modifying the main config file, the whitelist or blacklist.

2) run `autoygg-server`, e.g. in screen:

```
./autoygg-server
```

3) Now switch to machine that will run `autoygg-client`. Because the client will reconfigure your local networking, it needs sufficient privileges to do so, e.g. by using sudo to invoke it.

The `autoygg-client` program will accept parameters in a configuration file named *client.yaml*, which can be located in */etc/autoygg/* or in the directory the *autoygg-client* binary is executed from. Parameters may also be specified as command line arguments. The complete list of parameters can be consulted by running *autoygg-client --help*, e.g.:

```
$ ./autoygg-client --help

autoygg-client is a tool to register an Yggdrasil node with a gateway for internet egress.

Options:
      --action string              action (register/renew/release) (default "register")
      --debug                      debug output
      --defaultGatewayDev string   LAN default gateway device (default "eth0")
      --defaultGatewayIP string    LAN default gateway IP address (e.g. 192.168.1.1)
      --dumpConfig                 dump the configuration that would be used by autoygg-client and exit
      --gatewayHost string         Yggdrasil IP address of the gateway host
      --gatewayPort string         port of the gateway daemon (default "8080")
      --help                       print usage and exit
      --quiet                      suppress non-error output
```

Supply the yggdrasil IP of the node that runs autoygg-server as *--gatewayHost*. You do not need to be peered directly with that node. The *--defaultGatewayDev* and *--defaultGatewayIP* values are your current local default gateway information. It is needed by the *autoygg-client* tool because a host route to your local gateway IP needs to be installed for each of your Yggdrasil peers! The default values for the default gateway device and address are respectively eth0 and 192.168.1.1, and they may be omitted if your values match those defaults.

```
$ sudo ./autoygg-client --gateway-host the:yggdrasil:ip:where:autoygg-server:runs --wan-gw-dev eth0 --wan-gw-ip 192.168.10.1 --action register
Enabling Yggdrasil tunnel routing                                     [ ok ]
Adding Yggdrasil local subnet 0.0.0.0/0                               [ ok ]
Adding tunnel IP 10.42.42.1/16                                        [ ok ]
Adding Yggdrasil remote subnet 0.0.0.0/0                              [ ok ]
Getting Yggdrasil peers                                               [ ok ]
Adding Yggdrasil peer route for X.X.X.X via 192.168.10.1              [ ok ]
Adding default gateway pointing at 10.42.0.1                          [ ok ]
```

Tearing the session back down again will return your network configuration to its previous state:

```
sudo ./autoygg-client --gateway-host the:yggdrasil:ip:where:autoygg-server:runs --wan-gw-dev eth0 --wan-gw-ip 192.168.10.1 --action release
Removing default gateway pointing at 10.42.0.1                        [ ok ]
Getting Yggdrasil peers                                               [ ok ]
Removing Yggdrasil peer route for X.X.X.X via 192.168.13.1            [ ok ]
Removing Yggdrasil remote subnet 0.0.0.0/0                            [ ok ]
Removing tunnel IP 10.42.42.1/16                                      [ ok ]
Removing Yggdrasil local subnet 0.0.0.0/0                             [ ok ]
Disabling Yggdrasil tunnel routing                                    [ ok ]
```

## Hacking

GNU Make and a Go compiler, version 1.11 or higher are required. In addition,
[GolangCI-Lint](https://github.com/golangci/golangci-lint) is needed.

Build the software with the *make dev* command.

## Licensing

Autoygg is Free Software, released under the GNU Affero GPL v3 or later. See the LICENSE file for the text of the license.
