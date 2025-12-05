# Overview

This repo shows you how to get an SSH tunnel to forward ports from your local
machine to any machine that you can SSH into, provided that the server allows it.


# Linux - SystemD Quickstart

Linux just needs a simple systemd service that starts up the SSH tunnel on
startup.

## Remote Port Forwarding

For Remote port forwarding, edit the service template at
[ssh-remote-port-forward-DESC.service](ssh-remote-port-forward-DESC.service):

* Replace `<USER>` and `<GROUP>` with the user and group of who you want to run
  the systemd service under. Usually, `<GROUP>` is the same as `<USER>`. If
  unspecified, defaults to root.
* If the remote user is the same as `<USER>`, then remove `<REMOTE_USER>@`.
  Otherwise, specify the remote user you want to SSH in as.
* Replace `<REMOTE_SERVER>` with your remote server's (VSP's) IP address or
  domain name. This is what you are SSHing into.
* The SSH port is assumed to be `22`, but you can change that if needed.
* Replace `<REMOTE_PORT>` with the port you want to expose to the world from the
  remote server you are SSHing into.
* Replace `<LOCAL_PORT>` with a port from the local machine you are SSHing from.
  This will be accessible from `localhost:<LOCAL_PORT>` and will be connected
  via SSH tunnel to `<REMOTE_SERVER>:<REMOTE_PORT>`.

Then, save it to `/etc/systemd/system/ssh-remote-port-forward-DESC.service`.
`<DESC>` is a simple description of the service for the port you are forwarding.
Now, enable and start it:

```
sudo systemctl enable ssh-remote-port-forward-DESC.service
sudo systemctl start ssh-remote-port-forward-DESC
sudo systemctl status ssh-remote-port-forward-DESC
```

## Local Port Forwarding

Follow the steps for remote port forwarding, but use this template file:
[ssh-local-port-forward-DESC.service](ssh-local-port-forward-DESC.service).

Other differences from remote port forwarding:
* Replace `<MACHINE_VISIBLE_ON_REMOTE>` with the machine you want to access
  while SSHed into the remote server
* Replace `<MACHINE_VISIBLE_ON_REMOTE_PORT>` with the port of that machine to
  access.


## Debug

To debug, access the logs:

```
journalctl -u ssh-remote-port-forward-DESC.service
```

The ssh tunnel will be visible with this command on the remote server:

```
sudo netstat -tulpn
```

If you don't enable the service, it won’t start up automatically on boot up!

Sometimes a higher <REMOTE_PORT> number may be required because of permission
issues; sometimes you can’t bind to a lower port <= `1024` unless you are
`root`.

If an ISP blocks a specific port on a server (like port `22`), then make sshd
listen to multiple ports. In `/etc/ssh/sshd_config` add:

```
Ports 22
Ports 2222
```

That adds an additional port `2222` for SSHD to listen. Port `22` also is
necessary, since without it, port `22` stops being listened to.

Then, in `/etc/systemd/system/ssh-remote-port-forward-DESC.service`, update `ExecStart` to
use `p 2222` instead of `p 22`:

```
ExecStart=/usr/bin/ssh  <REMOTE_SERVER> -p 2222 -vvv -N -o ServerAliveInterval=30 -o ExitOnForwardFailure=yes -R *:<REMOTE_PORT>:localhost:<LOCAL_PORT>
```

Then, reload everything:

```
systemctl daemon-reload
systemctl restart ssh-remote-port-forward-DESC.service
```

--------------------------------------------------------------------------------

# Background on SSH Tunnels/Port Forwards

## Remote Port Forwarding

SSH Tunneling can be achieved with the following ssh client command:

```
ssh user1@example.com -N -R \*:80:localhost:8081
```

This command says "I want to make my local port `8081` accessible to the world
at `example.com` port `80`. Do this by SSHing into `example.com` as `user1`."

`-N`: says that we don't want a bash terminal (no commands)

`-R`: Remote port forward. The source/destination syntax is:

```
[who to let connect]:[remote port]:[who to tunnel to]:[local port to map to remote port]
```

With `example.com`'s SSH server set to have `GatewayPorts=clientspecified`, that
means that our command can set `\*` to mean anyone can connect to public port
`80`. If `GatewayPorts=yes`, then our `-R` command will _always_ set up a public
port, regardless of what you request in your ssh command.


## Local Port Forwarding

Instead of `-R`, you can do `-L`:

```
ssh user2@example.com -N -L 1234:private-machine:8080
```

`-N`: says that we don't want a bash terminal (no commands)

`-L` (Local) means "I want to access `private-machine` port `8080` from the
viewpoint of `user2` SSHed into `example.com`. Forward that back to my local
machine at port `1234`."

This is useful for accessing unexposed services on private (work) networks. For
example, let's say you have a Jenkins server on your work's private network at
`private-machine` (`192.168.2.200`) port `8080`. It's not exposed to the
internet, so you can access it only while SSHed into the private network. But
you want to access it via a web browser on your laptop, which is NOT on the
private network (you are only SSHed in - you are not on your work's VPN). By
port forwarding `private-machine:8000` to your local port `1234`, you can now
open `localhost:1234` in your browser on your laptop and SSHD will forward all
traffic to and from your laptop to the machine at `private-machine` via a secure
SSH tunnel.


## Further resources on SSH tunnels and port forwarding:

* [SSH 101 - SSH Port Forwarding](https://www.youtube.com/watch?v=JKrO5WABdoY) (Good visualization of port forwarding)
* [Self healing reverse SSH setup with systemd](https://blog.stigok.com/2018/04/22/self-healing-reverse-ssh-systemd-service.html)
* [SSH Reverse Tunnel on Linux with systemd](https://blog.kylemanna.com/linux/ssh-reverse-tunnel-on-linux-with-systemd/)
* [README-setup-tunnel-as-systemd-service.md](https://gist.github.com/drmalex07/c0f9304deea566842490)
* [How to make an SSH tunnel publicly accessible?](https://superuser.com/questions/588591/how-to-make-ssh-tunnel-open-to-public)
* [take changes in file sshd_config file without server reboot](https://askubuntu.com/questions/462968/take-changes-in-file-sshd-config-file-without-server-reboot)


# Use cases

You have a cheap Virtual Private Server (VPS) with a static public IP address.
You set up DNS records to associate your domain, example.com, with the static IP
address. You set up a proxy server with Caddy, which automatically uses Let's
Encrypt to get an HTTPS certificate. You add DNS records that point to various
subdomains (service1.example.com, service2.example.com) in your VPS.
You have a beefier private machine on your local network running various
services, and those services are bound to local ports that are not accessible
by anything else on the network.
Using SSH, you forward the local ports on your private/local beefy machine to
the remote's local ports. Finally, Caddy routes incoming requests to the right
local ports, with HTTPS traffic already unwrapped as HTTP. Since the tunnels are
through SSH, the content will still remain private.

