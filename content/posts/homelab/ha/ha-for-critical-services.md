+++
title = "High Availability for critical services in Home Lab"
summary = "Importance of high availability for critical services in Home Lab"
date = 2026-04-10T00:00:00Z   
tags = ["home lab", "critical services", "high availability", "tailscale", "unifi", "teleport"]
ShowReadingTime = true
draft = false
+++

In a home lab environment, there are certain critical services that we rely on for our day-to-day activities. These services could include things like a DNS server, monitoring tools, self hosted applications, or a remote access solution. Ensuring high availability for these critical services is essential to maintain a seamless and uninterrupted experience.

In my home lab, I have used Proxmox as my hypervisor and I have set up a few critical services that I rely on. These include Technitium DNS for domain name resolution and Tailscale for secure remote access.

Recently, I traveled out of town and tried to access Grafana via my Tailnet. I noticed one of my VMs on which I ran Grafana was running out of space. I logged into my Proxmox server and clicked on the VM to stop it.

At that moment I realized that I had not set up high availability for this critical service and I was no longer able to access any of my services remotely. 

We always think of high availability when we configure services in our work environment, but we often forget to set it up for our home lab. 

We should always keep in mind what impact would a service outage have on our home lab experience and set up high availability for critical services accordingly.

Fortunately, I use Unifi for my home lab and I had also enabled Teleport for remote access to my home lab. This allowed me to log into my Unifi controller and access the Proxmox UI. From there, I restarted the VM and re-enabled my Tailnet access.

This experience highlighted the importance of setting up high availability for critical services in a home lab environment. 

Here are a few lessons I took away from this experience:

- **Set up redundancy for your primary remote access tool.** The first action I took was to set up high availability for Tailscale by adding a second node that exposes subnets. This way, if one node goes down, the other can still serve traffic.

- **Always have a backup remote access method.** What if your primary tool like Tailscale goes down entirely? In my case, having Teleport enabled saved me — without it, I would have been locked out of my home lab until I got back home.

- **Don't forget DNS.** If you rely on a self-hosted DNS server like Technitium, make sure it also has redundancy. A DNS failure can silently break everything else.

- **Test your setup regularly.** Simulate a failure to confirm that both your primary and backup access methods work as expected. A simple way to do this is to turn off the WiFi on your phone and try accessing your home lab via Tailscale or Teleport.

## My Setup

My current setup is to have two Tailscale nodes running in LXC containers on Proxmox. Each of these nodes are on different Proxmox hosts and they both have subnet routing enabled. This way if one of the nodes goes down, I can still access my home lab via the other node. If for some reason both of these nodes go down, I can still access my home lab via Teleport.

## Conclusion

A home lab outage is rarely catastrophic, but it can be frustrating — especially when you are away from home and cannot physically access your hardware. The key takeaway is simple: don't wait for a failure to think about high availability. Audit your critical services, identify the ones that would cause the most disruption if they went down, and put a redundancy plan in place. If you have a similar setup or a different approach to HA in your home lab, I'd love to hear about it in the comments.
