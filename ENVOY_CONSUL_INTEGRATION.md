# Dynamic Service Discovery with Consul & Envoy

This document outlines how to configure Envoy Proxy to dynamically resolve backend services using the Consul Agent `.consul` DNS.

## 1. Consul Agent Setup on the Envoy Server
The Envoy proxy server itself needs to be able to resolve `*.service.consul`. 
The easiest way is to install the `consul-agent` role on the Envoy server as well, but without registering any local services (i.e. `consul_services: []`). 
This ensures the Envoy server has a local Consul agent listening on `127.0.0.1:8600` for DNS queries.

## 2. Configuring Envoy for STRICT_DNS
When you configure Envoy upstream clusters (like Keycloak, Orchestrator, Minio), change the discovery `type` to `STRICT_DNS` and point the `address` to the dynamically generated `.service.consul` hostname.

### Example `envoy.yaml` Cluster Config:
```yaml
clusters:
  - name: keycloak_cluster
    connect_timeout: 3s
    type: STRICT_DNS
    dns_lookup_family: V4_ONLY
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: keycloak_cluster
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: keycloak.service.consul
                    port_value: 8080
```

## 3. Configuring DNS Resolution for Envoy
Envoy needs to know how to resolve `keycloak.service.consul`. 

**Option A: Custom DNS Resolver inside envoy.yaml (Recommended)**
You can configure the cluster to explicitly query the local Consul DNS agent on port `8600`:

```yaml
clusters:
  - name: keycloak_cluster
    connect_timeout: 3s
    type: STRICT_DNS
    typed_dns_resolver_config:
      name: envoy.network.dns_resolver.cares
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.network.dns_resolver.cares.v3.CaresDnsResolverConfig
        resolvers:
          - socket_address:
              address: "127.0.0.1"
              port_value: 8600
    # ... rest of your cluster definition ...
```

**Option B: System Resolver via `systemd-resolved` / `resolv.conf`**
Alternatively, configure the Envoy host operating system to forward `.consul` domains to `127.0.0.1:8600`.
In your Envoy proxy Ansible role, add a task that configures `/etc/systemd/resolved.conf` or `dnsmasq` to forward requests to the local Consul agent.
