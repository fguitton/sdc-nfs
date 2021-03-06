# sdc-nfs

`sdc-nfs` implements a user-level [NFS v3](http://tools.ietf.org/html/rfc1813)
server in [node.js](http://nodejs.org/).

- [Overview](#overview)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [Usage](#usage)
- [Limitations](#limitations)
- [OS Specific Considerations](#os-specific-considerations)
    - [Linux](#linux)
    - [SmartOS](#smartos)


## Overview

The server is a user-level process that runs locally and services NFS requests.
Its primary purpose is to enable NFS service from within a SmartOS zone, which
is otherwise unable to act as an NFS server.

The server includes a built-in portmapper, but it will also interoperate
transparently with the zone's portmapper (usually `rpcbind`) if one is running.
The server also includes a built-in `mountd` and `nfsd`. There is no `lockd`
provided by the server.

By default, the server only listens on the localhost address and only serves
files locally. However, for production use it should be configured to serve
files to external hosts.

The server **requires** at least v0.10.x of [node.js](http://nodejs.org/).

## Getting Started

Clone the repo then run `npm install` within the clone to build all of the
dependencies. The 'Configuration' section of this readme describes how to
configure the server before you run it. The 'Usage' section of this readme
describes how to start the server and how to perform an NFS mount.

## Configuration

There are a variety of configuration options. An example configuration file,
showing all possible configuration options, is provided in `etc/example.json`.
Each section of the configuration file is optional. The configuration file is
specified to the server via the `-f` option:

    node server.js -f etc/myconfig.json

Although most of the sections in `etc/example.json` should be
self-explanatory, here is some additional information.

  * The `mount` section's `address` field can be used to specify an address
    other than localhost for the server to listen on. Using '0.0.0.0' tells the
    server to listen on all addresses. Both the `mountd` and `nfsd` within the
    server will listen on the given address. It is a good idea to limit foreign
    host access if listening on the external network. The `hosts_allow` or
    `hosts_deny` sections can be used to restrict access to the given IP
    addresses. The `exports` section can be used to restrict access to the
    specified portions of the local filesystem.

  * The `nfs` section is normally not needed, but it can be used to set the
    `uid` and `gid` values for 'nobody'. This is useful if NFS clients are
    running a different OS, which uses different values for 'nobody', as
    compared to the server. The `fd-cache` section can be used to configure the
    server's file descriptor cache.

## Usage

When running the server for the first time, you probably want to run it by
hand to confirm that the configuration is correct and things are working as
expected. Once you know things are working correctly, you may want to set up
a service so that the server runs automatically.

The server must be started as root since it needs access to the portmapper's
privileged port. The `sudo` or `pfexec` commands are typically used to run
a command as root, depending on which OS you're using.

In an lx zone the server can be run with no config file like:

    sudo node server.js

In a native zone the server can be run like:

    pfexec node server.js

To pass in a config file, use the -f option:

    pfexec node server.js -f etc/myconfig.json

All output logging is done via [bunyan](https://github.com/trentm/node-bunyan).
Once started, the server will output an occasional log message, but the `-d` or
`-v` options can be used to change the bunyan logging level to either 'debug'
or 'trace'. Logging at either of these levels is not recommended, except during
debugging, since there will be many log entries for each NFS operation. You may
want to redirect the output from the server into a file:

    pfexec node server.js -d -f etc/myconfig.json >log 2>&1

To mount a directory, use the standard NFS client `mount` command with a path
on the server. For example:

    pfexec mount 127.0.0.1:/data /mnt

Once you have confirmed that the server works as expected, you can set up a
service in your zone so that the server runs automatically when the zone boots.
This is discussed in the SmartOS section below.

## Limitations

The NFS mknod(2) operation is not supported.

AUTH_SYS is the only style of authorization supported.

UID/GID mapping between the server and clients is outside the scope of this
implementation.

## OS Specific Considerations

This section discusses any issues that are specific to using or running the
server in a specific type of zone.


### lx-branded zones

Some distributions (e.g. Ubuntu or Centos) may not come pre-installed with
the `/sbin/mount.nfs` command which is needed to perform a mount, while others
(e.g. Fedora) may be ready to go. On Ubuntu, install the `nfs-common` package.

    apt-get install nfs-common

On Centos, install the `nfs-utils` package.

There is no lock manager included in the server, so you must disable locking
when you mount. e.g.

    mount -o nolock 127.0.0.1:/foo.bar/public /home/foo/mnt

When mounting from inside an lx-branded zone you may need to explicitly
specify that you want to use the NFSv3 protocol. e.g.

    mount -o nolock,vers=3 127.0.0.1:/data /home/foo/mnt

### SmartOS

In order to mount from the host, the zone's 'rpcbind' must be running.  The
server's built-in portmapper cannot be used. If the svc is not already enabled,
enable it.

    svcadm enable network/rpc/bind

If you intend to serve external hosts, you must also ensure that the bind
service is configured to allow access. To check this:

    svccfg -s bind listprop config/local_only

If this is set to true, you need to change it to false.

    svccfg -s bind setprop config/local_only=false
    svcadm refresh bind

The `svc/smf/sdc-nfs.xml` file provides an example configuration for
smf(5). If necessary, edit the file and provide the correct paths to 'node',
'server.js' and your configuration file.

Run the following to load and start the service:

    svccfg -v import svc/smf/sdc-nfs.xml
