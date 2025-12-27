---
title : Dokploy and Traefik config
date: 2024-11-06 17:45 -500
categories: [opensource, self-host]
tags: [paas,platforms,platform] 
---

## Install Dockploy


```bash
curl -sSL https://dokploy.com/install.sh | sh
```


> 💡 **Tip:** Please follow the install guide on [dokploy!](https://docs.dokploy.com/en/docs/core/get-started/installation) for your specific use case.



**Now, let’s dive into the good stuff!** The following Traefik configuration enables you to serve your apps deployed through Dokploy on either `*.example.tld` or subdomains like `*.dok.example.tld`. In fact, you can even host your main website at `example.tld` and `www.example.tld`.

I’ve also updated the configuration to use `dnsChallenge` instead of `httpChallenge`. Why the change? In my specific setup, as mentioned before, I already had two Traefik instances in place before exploring platforms like Dokploy and Coolify. Many guides and videos suggest hosting on a VPS with a dedicated domain, but I already own a domain and have a robust infrastructure. I’ll be sharing details about what’s running in my setup in a future article. For now, though, there’s no need to pay for VPS hosting when all my services are securely routed through Traefik on port 443.

## Traefik config
`/etc/dokploy/traefik/traefik.yml`

```yaml
providers:
  swarm:
    exposedByDefault: false
    watch: false
  docker:
    exposedByDefault: false
  file:
    directory: /etc/dokploy/traefik/dynamic
    watch: true
entryPoints:
  web:
    address: ':80'
  websecure:
    address: ':443'
    http:
      tls:
        certResolver: letsencrypt
api:
  insecure: true
certificatesResolvers:
  letsencrypt:
    acme:
      email: test@example.com
      storage: /etc/dokploy/traefik/dynamic/acme.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53" 
          - "1.0.0.1:53" # Optional: specify a DNS resolver, Cloudflare’s public DNS server
```
## Traefik config
`/etc/dokploy/traefik/dynamic/dokploy.yml`

```yaml
http:
  routers:
    traefik-router:
      rule: Host(`traefik.example.com`)
      entryPoints:
        - websecure
      service: api@internal
      tls:
        certResolver: letsencrypt

    dokploy-router-app-secure:
      rule: Host(`dokploy.example.com`)
      entryPoints:
        - websecure
      service: dokploy-service-app
      tls:
        certResolver: letsencrypt
        domains:
          - main: "example.com"
            sans:
             - "*.example.com"
          - main: "dok.example.com"
            sans:
              - "*.dok.example.com"
             
       
  services:
    dokploy-service-app:
      loadBalancer:
        servers:
          - url: http://dokploy:3000
        passHostHeader: true

```

[Go back to the previous post](./2024-11-05-Dokploy-traefik.md)
