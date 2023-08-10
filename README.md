## Overview

For a full Traefik Enterprise cluster we need at least 1 Controller, 2 Ingress Proxies and 1 Plugin Registry.

| Servername | Function                | IP Address    |
| ---------- | ----------------------- | ------------- |
| traefik01  | Controller              | 192.168.66.20 |
| traefik02  | Proxy                   | 192.168.66.21 |
| traefik03  | Proxy                   | 192.168.66.23 |
| traefik04  | Proxy / Plugin Registry | 192.168.66.24 |
| desktop    | Management Station      | 192.168.66.22 | 

## Generic

### Download and install binaries
Beware that I am using the ARM64 binaries here, more binaries can be found [here](https://doc.traefik.io/traefik-enterprise/installing/on-premise/#generating-a-pre-shared-token).
These steps are performed on all four Traefik servers.

```shell
wget https://s3.amazonaws.com/traefikee/binaries/v2.10.4/traefikee_v2.10.4_linux_arm64.tar.gz
wget https://s3.amazonaws.com/traefikee/binaries/v2.10.4/teectl_v2.10.4_linux_arm64.tar.gz
tar -zxvf traefikee_v2.10.4_linux_arm64.tar.gz
tar -zxvf teectl_v2.10.4_linux_arm64.tar.gz
rm -fr docs/
sudo mv traefikee /usr/local/bin/
sudo mv teectl /usr/local/bin/
rm *.gz
```

### Generate `traefikee` user and group

```shell
groupadd -g 1500 traefikee
useradd -g traefikee --no-user-group --home-dir="/opt/traefikee" --shell="/usr/sbin/nologin" --system --uid="1500" traefikee
```

### Create `traefikee` directory

```shell
mkdir -p /opt/traefikee
```
### Generate pre-shared token (`$PLUGIN_TOKEN`)

This is used for both the Controller and the Plugin Registry

```shell
openssl rand -base64 32
```

## Controller (`traefik01`)

```shell
tee /opt/traefikee/controller.env << EOF
CONTROLLER_BIND_ADDRESS="192.168.66.20:4242"
TRAEFIKEE_LICENSE="<license-key>"
PLUGIN_URL="https://192.168.66.24"
PLUGIN_TOKEN="<plugin-token>"
EOF

chown -R traefikee:traefikee /opt/traefikee/

tee /etc/systemd/system/traefikee-controller.service << EOF
[Unit]
Description=Traefik Enterprise Controller
After=network-online.target
Wants=network-online.target systemd-networkd-wait-online.service

[Service]
EnvironmentFile=-/opt/traefikee/controller.env
Restart=on-abnormal
User=traefikee
Group=traefikee
ExecStart=/usr/local/bin/traefikee controller --advertise=\${CONTROLLER_BIND_ADDRESS} --license=\${TRAEFIKEE_LICENSE} --plugin.url=\${PLUGIN_URL} --plugin.token=\${PLUGIN_TOKEN} --api.socket=/opt/traefikee/run/teectl.sock --socket=/opt/traefikee/run/cluster.sock --statedir=/opt/traefikee/data --jointoken.file.path=/opt/traefikee/tokens --api.autocerts
PrivateTmp=true
PrivateDevices=false
ProtectHome=true
ProtectSystem=full
ReadWritePaths=/opt/traefikee
PermissionsStartOnly=true
ExecStartPre=/usr/bin/mkdir -p /opt/traefikee/run /opt/traefikee/data /opt/traefikee/tokens
ExecStartPre=-chown -R traefikee:traefikee /opt/traefikee
ExecStartPre=-chmod -R 700 /opt/traefikee

NoNewPrivileges=true
LimitNOFILE=16384

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now traefikee-controller.service
```

## Ingress Proxies (`traefik02`, `traefik03`, `traefik04`)

Retrieve the **proxy** token from `traefik01` from `/opt/traefikee/tokens/proxy`.

```shell
tee /opt/traefikee/proxies.env << EOF
CONTROLLER_PEERS="192.168.66.20:4242"
PROXY_NODE_TOKEN="<proxy-token>"
EOF

chown -R traefikee:traefikee /opt/traefikee/

tee /etc/systemd/system/traefikee-proxy.service << EOF
[Unit]
Description=Traefik Enterprise Proxy
After=network-online.target
Wants=network-online.target systemd-networkd-wait-online.service

[Service]
EnvironmentFile=-/opt/traefikee/proxies.env
Restart=on-abnormal
User=traefikee
Group=traefikee
ExecStart=/usr/local/bin/traefikee proxy --jointoken.value=\${PROXY_NODE_TOKEN} --discovery.static.peers=\${CONTROLLER_PEERS} --statedir=/opt/traefikee/data
PrivateTmp=true
PrivateDevices=false
ProtectHome=true
ProtectSystem=full
ReadWritePaths=/opt/traefikee
ExecStartPre=mkdir -p /opt/traefikee/data /var/log/traefikee
ExecStartPre=-chown -R traefikee:traefikee /var/log/traefikee
LimitNOFILE=16384

; The following additional security directives only work with systemd v229 or later.
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now traefikee-proxy.service
```

## Plugin Registry (`traefik04`)

```shell
tee /opt/traefikee/registry.env << EOF
CONTROLLER_PEERS="192.168.66.20:4242"
PROXY_NODE_TOKEN="<proxy-token>"
PLUGIN_TOKEN="<plugin-token>"
EOF
chown -R traefikee:traefikee /opt/traefikee/

tee /etc/systemd/system/traefikee-plugin-registry.service << EOF
[Unit]
Description=Traefik Enterprise Plugin Registry
After=network-online.target
Wants=network-online.target systemd-networkd-wait-online.service

[Service]
EnvironmentFile=-/opt/traefikee/registry.env
Restart=on-abnormal
User=traefikee
Group=traefikee
ExecStart=/usr/local/bin/traefikee plugin-registry --jointoken.value=\${PROXY_NODE_TOKEN} --discovery.static.peers=\${CONTROLLER_PEERS} --token=\${PLUGIN_TOKEN} --plugindir=/opt/traefikee/plugins --statedir=/opt/traefikee/data --name="%H"
PrivateTmp=true
PrivateDevices=false
ProtectHome=true
ProtectSystem=full
ReadWritePaths=/opt/traefikee
ExecStartPre=mkdir -p /opt/traefikee/data
LimitNOFILE=16384

; The following additional security directives only work with systemd v229 or later.
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now traefikee-plugin-registry.service
```

## Remote Management

### Controller (`traefik01`)

```shell
traefikee generate credentials --onpremise.hosts="192.168.66.20" --cluster=default --socket="/opt/traefikee/run/teectl.sock" > config.yaml
```

### Management station (`desktop`)

```shell
teectl cluster import --file="config.yaml"
teectl get nodes
```

#### Create static configuration

```shell
tee traefikee_config.yml << EOF
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true
  websecure:
    address: ":443"
    http:
      middlewares:
        - hsts@traefikee
      tls: {}
  ping:
    address: ":8082"

metrics:
  prometheus: {}

api:
  dashboard: true
  insecure: true

log:
  format: json
  filePath: "/var/log/traefikee/traefikee.log"
  level: DEBUG

accessLog:
  format: json
  filePath: "/var/log/traefikee/access.log"

ping:
  entryPoint: ping

providers:
  file:
    filename: dynamic_conf.yml
EOF
teectl apply --file=traefikee_config.yml
```

#### Create dynamic configuration

```shell
tee dynamic_conf.yml << EOF
http:
  middlewares:
    hsts:
      headers:
        stsSeconds: 63072000
        stsIncludeSubdomains: true
        stsPreload: true
  serversTransports:
    skiptls:
      insecureSkipVerify: true

tls:
  options:
    default:
      sniStrict: true
      minVersion: VersionTLS12
EOF
teectl apply --file=dynamic_conf.yml
```

Now you should be able to navigate to either `http://traefik02:8080`, `http://traefik03:8080` or `http://traefik04:8080` and see the UI.
