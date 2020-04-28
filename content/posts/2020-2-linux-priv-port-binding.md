+++
title = "Privileged port binding in Linux"
date = "2020-02-28"
tags = ["linux", "frustration"]
draft = true
+++

This one time at work, back in February this year, I spent 3, maybe 4 days trying to implement a clean
solution to [privileged port binding](https://www.w3.org/Daemon/User/Installation/PrivilegedPorts.html) in linux. This is something that initially took 30 minutes to solution
to grow to only 3-4 days of tinkering, hair-pulling, etc... The issue was that I was rebuilding non-containerized
applications on GCP, replacing the old HAProxy instances, with cloud load balancers. In doing so the
[HAProxy](https://www.haproxy.org)-based front end that bound to 80/443 and proxied to a non-priv port in Go was going away. This Go-app
that didn't support dropping root privs, etc... and could not bind to 80/443 itself, at least not without one of the
following solutions:

- Setting the capability `cap_net_bind_service` to `+ep` for the binary (via path) - this is a pretty clean solution
- Port traffic redirection via the kernel: `iptables -A PREROUTING -t nat -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 1080` (if the Go-based app, in this case, had a non-TLS binding of `1080`)  - this seems like a hack and it isn't obvious as the app itself will bind to `1080`
- Running the application as root - bad, or even, starting the app as root and dropping privs to a more restricted user

The first option, setting the capability `cap_net_bind_service`, was clean, in my opinion, and worked ... easily! Unfortunately,
the deploy process for this application meant that the symlink and binaries were disjoint after a deploy and Puppet run.
We couldn't guarantee when and how we'd ensure things were good.

Looking further at things I found [authbind](https://en.wikipedia.org/wiki/Authbind). The concept of authbind seemed simple...

- Touch files at `/etc/authbind` in the `byport|byaddr|byuid` sub-directories with child nodes that are ports, or, in the case of `byport` they're ports.
- Full example for this use-case `/etc/authbind/byport/80` and `/etc/authbind/byport/443` where those files are owned by the user/group and have read permissions.
- Creating the example: `touch /etc/authbind/byport/80 && chown go-app /etc/authbind/byport/80 && chmod 500 /etc/authbind/byport/80`

The final, connecting piece of this, is that you start your app wrapped in a call to authbind. So, if your app is an executable:

- at `/opt/specialthing` you'd start it with `authbind /opt/specialthing`
- java app, ...with `authbind java -jar x.jar`

etcetera...

With go, however, I was struggling and finding that it didn't seem to be working. The go-app wasn't able to bind where it should be, even in case where the same calling user to something like `nc` would work.
