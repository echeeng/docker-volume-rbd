# Docker Volume Rbd Plugin

RADOS block device Docker volume plugin

## Overview

This is work based on [Nenzyz/docker-volume-rbd](https://github.com/Nenzyz/docker-volume-rbd)

## Steps to build the plugin

At the build machine ...

### Step 1. Pull the RootFs image

```bash
docker pull imachineml/rbd:rootfs
```

### Step 2. Create the config.json

```bash
vim config.json
```

Content:

```json

{
  "description": "RBD plugin for Docker",
  "documentation": "https://github.com/echeeng/docker-volume-rbd",
  "entrypoint": [
    "/docker-volume-rbd"
  ],
  "env": [
    {
      "name": "PLUGIN_VERSION",
      "Description": "Current version of RBD plugin for Docker Plugin",
      "settable": [
        "value"
      ],
      "value": "1.0.0"
    },
    {
      "name": "LOG_LEVEL",
      "Description": "[0:ErrorLevel; 1:WarnLevel; 2:InfoLevel; 3:DebugLevel] defaults to 0",
      "settable": [
        "value"
      ],
      "value": "0"
    },
    {
      "name": "RBD_CONF_DEVICE_MAP_ROOT",
      "settable": [
        "value"
      ]
    },
    {
      "name": "RBD_CONF_POOL",
      "settable": [
        "value"
      ]
    },
    {
      "name": "RBD_CONF_CLUSTER",
      "settable": [
        "value"
      ]
    },
    {
      "name": "RBD_CONF_KEYRING_USER",
      "settable": [
        "value"
      ]
    },
    {
      "name": "MOUNT_OPTIONS",
      "Description": "Options to pass to the mount command",
      "settable": [
        "value"
      ],
      "value": "--options=noatime"
    }
  ],
  "interface": {
    "socket": "rbd.sock",
    "types": [
      "docker.volumedriver/1.0"
    ]
  },
  "linux": {
    "AllowAllDevices": true,
    "capabilities": [
      "CAP_SYS_ADMIN",
      "CAP_SYS_MODULE"
    ],
    "devices": null
  },
  "mounts": [
    {
      "source": "/lib/modules",
      "destination": "/lib/modules",
      "type": "bind",
      "options": [
        "rbind"
      ]
    },
    {
      "source": "/dev",
      "destination": "/dev",
      "type": "bind",
      "options": [
        "shared",
        "rbind"
      ]
    },
    {
      "source": "/etc/ceph",
      "destination": "/etc/ceph",
      "type": "bind",
      "options": [
        "rbind"
      ]
    },
    {
      "source": "/sys",
      "destination": "/sys",
      "type": "bind",
      "options": [
        "rbind"
      ]
    }
  ],
  "network": {
    "type": "host"
  },
  "propagatedmount": "/mnt/volumes"
}

```

### Step 3. Build the plugin

```bash

sudo rm -rf ./plugin \
    && mkdir -p ./plugin/rootfs \
    && docker create --name tmp imachineml/rbd:rootfs \
    && sudo docker export tmp | sudo tar -x --exclude=dev/ -C ./plugin/rootfs \
    && cp config.json ./plugin/ \
    && docker rm -vf tmp && \
    && docker plugin rm -f imachineml/rbd:1.0.0 || true \
    && sudo docker plugin create imachineml/rbd:1.0.0 ./plugin \
    && sudo rm -rf ./plugin

```

