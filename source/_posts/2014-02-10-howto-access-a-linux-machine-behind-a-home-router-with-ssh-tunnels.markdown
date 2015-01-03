---
layout: post
title: "Howto access a Linux machine behind a home router with SSH tunnels"
date: 2014-02-10 08:41:00 -0800
comments: true
categories: linux ssh tutorials
---
Recently, I had the need to set up a Linux machine so that I could put it behind a home router with
no special configuration, and still access the machine from anywhere. Machines behind home routers
are normally difficult to access from the outside world for two reasons: dynamic IP addresses and
network address translation.

Dynamic IPs mean you don't know, without being behind the same home
router, what the IP address to connect to is at any given time. Network Address Translation makes it
impossible to connect to a machine behind a home router unless the router is specifically configured
to allow it.

However, nothing about this arrangement disallows the machine behind the home router from making
outbound connections, and thanks to the flexibility of SSH tunnels, we can use this to create a
simple, self configuring and healing connection allowing us to SSH into the machine behind the home
router.

Here's the different pieces of hardware that play into the story:

HostA: This is a Linux machine with a known and publicly accessible IP address. Its best if the IP
address is static. In my case it's a VPS on Linode, but any Linux machine accessible via SSH can
work. No root access or special configurations are required.

Router: This is the home router. For our purposes it's just a standard Linksys router with no
special configuration, and we do not have access to its control panel.

HostB: This is the Linux machine you will put behind the home router. Again, full root access is not
required, only a user account. It's assumed that we will lose physical access to this machine once
its placed behind the home router, but can configure it as we wish before then.

HostC: another Linux machine of any type and network location, where we have direct physical access.

Given these machines, the requirement is simple: allow us to SSH from HostC to HostB.

## SSH Tunnel Primer

One of the most powerful abilities of SSH is to set up port forwarding (much like might be done on a
home router), but ensure the forwarded traffic is routed through a secure connection created by SSH.

One of the classic uses for SSH tunnels is in fact exactly what we need to get around the
restrictions put in place by the router. If we execute the following command on HostB

    ssh -R 10022:localhost:22 HostA

then we can connect to HostB from HostA this way:

    ssh -p 10022 localhost

How does this work? The first command tells SSH to set up a secure port forwarding from port 22,
where SSH is listening, on HostB, to port 10022 on HostA. The router does not disallow HostB
from initiating outbound connections, so this command does not fail. Users other than root can set
up sockets on ports above 1024 without issue, so this also doesn't require any special permissions
on HostA.

However, currently, this command would have to be run manually on HostB, and it simply
drops us into an SSH session to HostA. As soon as that session was closed, or as soon as there was a
network issue causing the SSH session to terminate, our tunnel to HostB terminates as well. A little
more work is required to create a foolproof setup.

## Setting up a hands-free SSH tunnel

To facilitate setting up persistent port forwarding, SSH has two handy options: -N, and -f.
Combining these two with our command above will send SSH into the background after initializing our
port forwarding. With this, a simple cron job can ensure that we have a connection to HostB even if
the network temporarily goes down or HostB is rebooted. However, it will create a new SSH connection
every time, which is at best inefficient, and at worst can consume all the resources on one of our
machines.

How then, can we check if there is already a connection from HostB to HostA, and only initialize a
 connection when needed? Let's take a look at another SSH port forwarding command:

    ssh -L 19922:HostA:22 HostA

If run from HostB, this will set up a port forwarding from port 19922 on HostB, to port 22 on HostA,
essentially the reverse of the previous forwarding. We can already ssh to port 22 on HostA directly
from HostB, but now, we can also SSH to HostA from HostB with the following command:

    ssh localhost -p 19922

Since port 19922 is only active when our ssh tunnel is in place, this allows us to check for the
tunnel's existence: if our tunnel is down, ssh will fail with a connection error.

One final trick SSH gives us is the ability to set up multiple port forwardings in one command. With
that, our complete solution can be written in a simple bash script:

{% codeblock lang:bash %}
createTunnel() {
    /usr/bin/ssh -f -N -R 10022:localhost:22 -L19922:HostA:22 HostA
    if [[ $? -eq 0 ]]; then
        echo Tunnel to HostA created successfully
    else
        echo An error occurred creating a tunnel to HostA. Return code: $?
    fi
}
/usr/bin/ssh -p 19922 localhost ls > /dev/null
if [[ $? -ne 0 ]]; then
    echo Creating new tunnel connection to HostA
    createTunnel
fi
{% endcodeblock %}

The meat of the script is in the createTunnel function, which creates the tunnel as described above.
The canary tunnel is checked on line 9, and if the connection fails, a new tunnel is created.

I've been running this code myself for several months and haven't had any problems, but would love
to hear from anyone else using something similar. Happy coding!
