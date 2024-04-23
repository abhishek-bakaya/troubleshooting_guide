# Install Quay Registry in IPv6/dual-stack setup

## Make Podman dual-stack

By default, podman is configured with IPv4 network only. For IPv6 or dual stack setups, we have to add IPv6 subnet in its network config.


1. List current network.
```
    podman network ls
```
output:
```
    NETWORK ID       NAME        DRIVER
    2f259bdh68aa     podman      bridge
```


2. Inspect the podman network config.
```
    podman network inspect podman
```

The output is a JSON with following default entries:
```
        ...

        "subnets": [
            {
                "subnet": 10.88.0.0/16",
                "gateway": 10.88.0.1"
            }
        ],
        "ipv6_enabled": false,
        
        ...
```


3. Create a dual-stack network temp.
```
    podman network create temp --ipv6
```

4. Inspect current podman bridge.
```
    nmcli con show
```
output:
```
    NAME          UUID    TYPE      DEVICE
    cni-podman0   ...     bridge    cni-podman0
```

5. Edit temp configuration and change the network name and bridge as above and restart podman.
```
    vi /etc/cni/net.d/tmp.conflist
    
    ...
    "name": "podman",
    "plugins": [
        {
            "type": "bridge",
            "bridge": "cni-podman0",
            ...
        }
        ...
    ]
    ...


    systemctl restart podman
```

6. Inspect new podman network.
```
    podman network inspect podman
```
An IPv6 network and a gateway would have been added in the config.



## Install Quay Registry

1. Run the installation command.
```
    mirror-registry install --quayHostname installer.cluster.example.com --quayRoot /root/mirror-registry/data --quayStorage /root/mirror-registry/data/quay-storage --pgStorage /root/registry/data/pg-data --initUser admin --initPassword "password" --sslCert /root/mirror-registry/ssl/ssl.cert --sslKey /root/mirror-registry/ssl/ssl.key
```

2. Wait for `/root/mirror-registry/data/quay-config/config.yaml` to generate. Add `FEATURE_LISTEN_IP_VERSION` parameter and set it to `IPv6` or `dual-stack` as per setup.
```
    vi config.yaml
    ...
    FEATURE_ANONYMOUS_ACCESS: true
    FEATURE_LISTEN_IP_VERSION: IPv6
    FEATURE_APP_REGISTRY: false
    ...
```

3. Restart podman and quay-pod.
```
    systemctl restart podman
    systemctl restart quay-pod.service
```
