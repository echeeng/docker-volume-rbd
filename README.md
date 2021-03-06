# Docker Volume Rbd Plugin

RADOS block device Docker volume plugin

## Overview

This is work based on [Nenzyz/docker-volume-rbd](https://github.com/Nenzyz/docker-volume-rbd)

In building this plugin, we will use a pre-built image *imachineml/rbd:rootfs*.
This image was built with some defaults. For e.g the default volume size (if
user did not specify on create request) is set to 10GB. If these defaults need to change,
we will have to build a different *rootfs* image and use that to build the plugin here.

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

The following will create the plugin *imachineml/rbd:1.0.0*. Any existing plugin
with the same name:tag will be replaced.

```bash

sudo rm -rf ./plugin \
    && mkdir -p ./plugin/rootfs \
    && docker create --name tmp imachineml/rbd:rootfs \
    && sudo docker export tmp | sudo tar -x --exclude=dev/ -C ./plugin/rootfs \
    && cp config.json ./plugin/ \
    && docker rm -vf tmp \
    && docker plugin rm -f imachineml/rbd:1.0.0 || true \
    && sudo docker plugin create imachineml/rbd:1.0.0 ./plugin \
    && sudo rm -rf ./plugin

```

After the build, we can see that the plugin in loaded into this docker and is
in state disabled.

```bash
$ docker plugin ls

ID                  NAME                   DESCRIPTION             ENABLED
f6fe8670aaae        imachineml/rbd:1.0.0   RBD plugin for Docker   false
```

### Step 4. Push the plugin to registry

If we want to store the plugin to a registry, we can push it and clean up from
this build machine (or leave it there).

```bash

docker plugin push imachineml/rbd:1.0.0

docker plugin rm imachineml/rbd:1.0.0

```

## Usage of the plugin

### Installation and activation of the plugin

At the every node in your cluster, install the plugin. When installing, you may
specify few parameters (refer to the *config.json* during build). Some basic ones
you should set are shown below as an example.

Note that you should have already created the Ceph pool *imachine* and client
*docker* before hand. The host should have the client's keyring file at location
*/etc/ceph/ceph.client.docker.keyring*.


```bash

docker plugin install imachineml/rbd:1.0.0 \
    --alias=imachine/rbd \
    LOG_LEVEL=3 \
    RBD_CONF_POOL="imachine" \
    RBD_CONF_CLUSTER=ceph \
    RBD_CONF_KEYRING_USER=client.docker

```

### Usage of the plugin

When starting a container, the volume will be automatically created with the
following:

```bash
docker run .... -v vollabel:/data --volume-driver=imachine/rbd ...
```

You may also create the volume beforehand like so (specifying the size 512mb):

```bash

docker volume create \
    -d imachine/rbd \
    -o size=512 \
    vollabel

```
