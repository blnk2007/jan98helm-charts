![version: 0.1.2](https://img.shields.io/badge/version-0.1.2-informational?style=flat-square) ![type: application](https://img.shields.io/badge/type-application-informational?style=flat-square) ![app version: v1.6.1](https://img.shields.io/badge/app%20version-v1.6.1-informational?style=flat-square) [![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/satisfactory)](https://artifacthub.io/packages/search?repo=satisfactory)

# satisfactory
Helm chart for deploying a Satisfactory server in Kubernetes. This chart can be installed without modification to deploy a Satisfactory server using the defaults specified in the [wolveix/satisfactory-server](https://hub.docker.com/r/wolveix/satisfactory-server) image.

# Installation

```sh
helm repo add 98jan https://98jan.github.io/helm-charts/
helm install satisfactory 98jan/satisfactory
```

## Setup

Recent updates consume 4GB - 6GB RAM, but [the official wiki](https://satisfactory.wiki.gg/wiki/Dedicated_servers#Requirements) recommends allocating 12GB - 16GB RAM.

## Environment Variables

| Parameter               |  Default  | Function                                            |
|-------------------------|:---------:|-----------------------------------------------------|
| `AUTOPAUSE`             |  `true`   | pause game when no player is connected              |
| `AUTOSAVEINTERVAL`      |   `300`   | autosave interval in seconds                        |
| `AUTOSAVENUM`           |    `5`    | number of rotating autosave files                   |
| `AUTOSAVEONDISCONNECT`  |  `true`   | autosave when last player disconnects               |
| `CRASHREPORT`           |  `true`   | automatic crash reporting                           |
| `DEBUG`                 |  `false`  | for debugging the server                            |
| `DISABLESEASONALEVENTS` |  `false`  | disable the FICSMAS event (you miserable bastard)   |
| `MAXOBJECTS`            | `2162688` | set the object limit for your server                |
| `MAXPLAYERS`            |    `4`    | set the player limit for your server                |
| `MAXTICKRATE`           |   `30`    | set the maximum sim tick rate for your server       |
| `NETWORKQUALITY`        |    `3`    | set the network quality/bandwidth for your server   |
| `PGID`                  |  `1000`   | set the group ID of the user the server will run as |
| `PUID`                  |  `1000`   | set the user ID of the user the server will run as  |
| `ROOTLESS`              |  `false`  | run the container as a non-root user                |
| `SERVERGAMEPORT`        |  `7777`   | set the game's port                                 |
| `SERVERIP`              | `0.0.0.0` | set the game's ip (usually not needed)              |
| `SKIPUPDATE`            |  `false`  | avoid updating the game on container start/restart  |
| `STEAMBETA`             |  `false`  | set experimental game version                       |
| `TIMEOUT`               |   `30`    | set client timeout (in seconds)                     |

## Experimental Branch

If you want to run a server for the Experimental version of the game, set the `STEAMBETA` environment variable to `true`.

## How to Improve the Multiplayer Experience

The [Satisfactory Wiki](https://satisfactory.wiki.gg/wiki/Multiplayer#Engine.ini) recommends a few config tweaks to really get the best out of multiplayer. These changes are already applied to the server, but they need to be applied to your local config too:

-   Press `WIN + R`
-   Enter `%localappdata%/FactoryGame/Saved/Config/WindowsNoEditor`
-   Copy the config data from the wiki into the respective files
-   Right-click each of the 3 config files (Engine.ini, Game.ini, Scalability.ini)
-   Go to Properties > tick Read-only under the attributes

## Restrict network accessibility with Network Policies
As I am using cilium as my Kubernetes CNI, I will provide here an example of NetworkPolicies to restrict satisfactory's network communication:

The domain "telemetry.deparc.net" is disabled by default and traffic is blocked, so far no impact on the server could be seen.

ToDo: add NetworkPolicies as optional

```YAML
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: default-deny
  namespace: satisfactory
spec:
  description: "Block all the traffic (except DNS) by default"
  ingress:
    - {}
  egress:
    - toEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: '53'
              protocol: UDP
          rules:
            dns:
              - matchPattern: '*'
  endpointSelector:
    matchExpressions:
      - key: io.kubernetes.pod.namespace
        operator: NotIn
        values:
          - kube-system
---
# allow ingress from world to 7777
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-ingress-from-world
  namespace: satisfactory
spec:
  endpointSelector:
    matchLabels:
      io.cilium.k8s.policy.serviceaccount: default
  ingress:
    - fromEntities:
        - "world"
      toPorts:
        - ports:
            - port: "7777"
              protocol: TCP
            - port: "7777"
              protocol: UDP
---
# allow egress to world from 7777
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-egress-to-world
  namespace: satisfactory
spec:
  endpointSelector:
    matchLabels:
      io.cilium.k8s.policy.serviceaccount: default
  egress:
    - toEntities:
      - "world"
      toPorts:
        - ports:
            - port: "7777"
              protocol: TCP
            - port: "7777"
              protocol: UDP
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-egress-to-steam
  namespace: satisfactory
spec:
  endpointSelector:
    matchLabels:
      io.cilium.k8s.policy.serviceaccount: default
  egress:
    - toFQDNs:
        - matchName: media.steampowered.com
        - matchName: test.steampowered.com
        - matchName: clientconfig.akamai.steamstatic.com
      toPorts:
        - ports:
            - port: "80"
    - toFQDNs:
        - matchName: cdn.steamstatic.com
        - matchName: api.steampowered.com
        - matchName: steampipe.akamaized.net
        - matchName: api.ipify.org
        - matchPattern: "*.steamcontent.com"
      toPorts:
        - ports:
            - port: "443"
    # https://help.steampowered.com/en/faqs/view/2EA8-4D75-DA21-31EB#:~:text=The%20following%20are%20ports%20you,UDP%20remote%20port%2027015%2D27050
    - toFQDNs:
        - matchPattern: "*.steamserver.net"
      toPorts:
          - ports:
              - port: "443"
              - port: "27015"
                protocol: TCP
                endPort: 27050
              - port: "27015"
                protocol: UDP
                endPort: 27050
```

## Ingress Configuration

This chart supports ingress configuration with the following options in `values.yaml`:

- `ingress.enabled`: Enable or disable ingress resource creation.
- `ingress.className`: Specify the ingress class (e.g., nginx, traefik).
- `ingress.annotations`: Custom annotations for the ingress resource.
- `ingress.hosts`: List of hosts with paths and path types.
- `ingress.tls`: TLS configuration with secret names and hosts.

Example ingress configuration in `values.yaml`:

```yaml
ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []
```

## Extra Ports Configuration

This chart supports extra ports on the service and deployment with the following options:

- `service.extraPortsEnabled`: Boolean flag to enable or disable extra ports.
- `service.extraPorts`: List of extra ports to expose.

Example extra ports configuration in `values.yaml`:

```yaml
service:
  extraPortsEnabled: false
  extraPorts:
    - name: extra-tcp
      port: 8888
      targetPort: 8888
      protocol: TCP
```

Set `extraPortsEnabled` to `true` to enable the extra ports defined in `extraPorts`.

---

This completes the documentation update to reflect the recent changes. I will now commit and push this updated README along with any other changed files.
