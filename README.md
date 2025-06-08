# Overview

This repo shows you how to get an SSH tunnel to forward ports from your local
machine to any machine that you can SSH into, provided that the server allows it.


# Linux - SystemD Quickstart

Linux just needs a simple systemd service that starts up the SSH tunnel on
startup:

```
[Unit]
Description=Create an SSH tunnel to forward a local port to a remote server.
After=network.target

[Service]
Type=simple
User=<USER>
Group=<GROUP>
ExecStart=/usr/bin/ssh <REMOTE_SERVER> -p 22 -vvv -N -o ServerAliveInterval=30 -o ExitOnForwardFailure=yes -R *:<REMOTE_PORT>:localhost:<LOCAL_PORT>
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

* Replace `<USER>` and `<GROUP>` with the user and group of who you are SSHing
  in under. Usually, `<GROUP>` is the same as `<USER>`.
* Replace `<REMOTE_SERVER>` with your remote server's (VSP's) IP address or
  domain name. This is what you are SSHing into.
* Replace `<REMOTE_PORT>` with the port you want to expose to the world from your
  server you are SSHing into.
* Replace `<LOCAL_PORT>` with a local port from the machine you are SSHing from
  that you want to tunnel to the server and forward to `<REMOTE_PORT>`.

Save it to `/etc/systemd/system/ssh-tunnel-DESC.service`. Then start it:

```
sudo systemctl start ssh-tunnel-DESC
sudo systemctl status ssh-tunnel-DESC
```

`<DISC>` is a simple description of the service for the port you are forwarding.

To debug, access the logs:

```
journalctl -u ssh-tunnel-DESC.service
```

Don’t forget to enable it, or else it won’t start up automatically on boot up!:

```
systemctl enable ssh-tunnel-DESC.service
```

Sometimes a higher <REMOTE_PORT> number may be required because of permission
issues; sometimes you can’t bind to a lower port  <= 1024 unless you are root.

If an ISP blocks a specific port on a server (like port 22), then make sshd
listen to multiple ports. In `/etc/ssh/sshd_config` add:

```
Ports 22
Ports 2222
```

That adds an additional port 2222 for SSHD to listen. Port 22 also is necessary,
since without it, port 22 stops being listened to.

Then, in `/etc/systemd/system/ssh-tunnel-DESC.service`, update `ExecStart` to
use `p 2222` instead of `p 22`:

```
ExecStart=/usr/bin/ssh  <REMOTE_SERVER> -p 2222 -vvv -N -o ServerAliveInterval=30 -o ExitOnForwardFailure=yes -R *:<REMOTE_PORT>:localhost:<LOCAL_PORT>
```

Then, reload everything:

```
systemctl daemon-reload
systemctl restart ssh-tunnel-DESC.service
```

--------------------------------------------------------------------------------

# Background on SSH Tunnels/Port Forwards

SSH Tunneling can be achieved with the following ssh client command:

```
ssh user@example.com -N -R \*:80:localhost:32400
```

The source/destination syntax is:

```
[who to let connect]:[remote port]:[who to tunnel to/from]:[local port to map to remote port]
```


`-N`: says that we don't want a bash terminal (No commands)

`-R`: Remote. "I want to make something local accessible to the world via a
remote machine's IP address."

With example.com's SSH server set to have `GatewayPorts=clientspecified`, that
means that our command can set `\*` to mean anyone can connect to public port
80. If `GatewayPorts=yes`, then your -R command will _always_ set up a public
port, regardless of what you request in your ssh command.


Another option is to do this:

```
ssh user@example.com -N -L 1234:private-machine:8080
```

`L` (Local) means "I want to access something in the world from my local
machine through a remote machine intermediary."

This is useful if you want to get inside a private network at work and forward a
HTTP server port from that private network at port 8080 back to your own machine
at port 1234, so you can open it in your browser with `localhost:1234`.


The above command will SSH into my remote server `example.com` (on the
default port 22), then forward the remote server's port 80 to my local machine's
port 32400.

For this to work, first edit `/etc/ssh/sshd_config` file on the remote server
and set `GatewayPorts` to `clientspecified`.
Then, restart the sshd on the remote server for the change to take effect:

```
sudo service sshd restart
```

Now, all new `ssh ... -R ...` requests do `0.0.0.0:80` instead of
`127.0.0.1:80`!


The ssh tunnel on the custom port is visible with this command on the remote
server:

```
sudo netstat -tulpn
```


Further resources on SSH tunnels and port forwarding:

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

