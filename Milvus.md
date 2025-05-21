# Install Milvus on Alpine Linux VM as a Rootless Podman Container

This instruction guide will walk you through setting up Milvus on an Alpine Linux virtual machine using rootless Podman containers, with system-managed dependencies started via OpenRC. Replace `<USER>` with your intended username where specified.

## 1. Prepare the Alpine Linux System

### 1.1 Enable the Community Repository

Edit the repositories file to enable the `community` repository:

```bash
sudo vi /etc/apk/repositories
```

Uncomment the community line if it is commented out. Save and exit.

### 1.2 Update Package Lists

Update the package index and upgrade current packages:

```bash
sudo apk update && sudo apk upgrade
```

## 2. Install Required Packages

Install QEMU Guest Agent, Podman, and util-linux:

```bash
sudo apk add qemu-guest-agent podman util-linux
```

## 3. Enable and Start QEMU Guest Agent

Add QEMU Guest Agent to system startup and start it immediately:

```bash
sudo rc-update add qemu-guest-agent
sudo rc-service qemu-guest-agent start
```

## 4. Prepare Podman for Rootless Operation

### 4.1 Enable cgroups

Set up cgroups support for containers:

```bash
sudo rc-update add cgroups
sudo rc-service cgroups start
```

### 4.2 Configure User for Rootless Containers

Replace `<USER>` with the intended local user:

```bash
sudo modprobe tun
sudo echo tun >>/etc/modules
sudo echo <USER>:100000:65536 >/etc/subuid
sudo echo <USER>:100000:65536 >/etc/subgid
```

## 5. Create OpenRC Service for Podman Preparation

Create an OpenRC script to handle necessary mount propagation for Podman containers:

```bash
sudo vi /etc/init.d/podman-preparation
```

Paste the following content:

```bash
#!/sbin/openrc-run

name="podman-preparation"
description="Preparations for podman"
command_background="no"

depend() {
    need net
    after firewall
    after cgroups
}

start() {
    ebegin "Starting ${name}"
    mount --make-rshared /
    findmnt -o PROPAGATION /
    eend $?
}

stop() {
    ebegin "Stopping ${name}"
    mount --make-rprivate /
    findmnt -o PROPAGATION /
    eend $?
}
```

Make the script executable and enable it at boot:

```bash
sudo chmod +x /etc/init.d/podman-preparation
sudo rc-update add podman-preparation
sudo rc-service podman-preparation start
```

It is recommended to reboot after this step to ensure all settings take effect.

## 6. Create and Configure the Milvus Pod

Run subsequent commands as your non-root `<USER>`.

### 6.1 Create Podman Pod

Create a Podman pod to hold all Milvus-related containers:

```bash
podman pod create --name milvus-pod -p 2379:2379 -p 9000:9000 -p 9001:9001 -p 19530:19530
```

Summary of Use:

- 2379 is used internally by Milvus to communicate with Etcd (distributed metadata storage) via Client API.
- 9000 is required by Milvus to interact with Minio for object storage; also available for S3-compatible tools.
- 9001 port allows you to access the Minio web dashboard from your browser.
- 19530 is required by applications/clients to interface with Milvus (gRPC protocol).

## 7. Set Up Milvus Dependencies

### 7.1 Etcd

Create Data Directory:

```bash
mkdir -p /srv/etcd
```

Create OpenRC Service for Etcd:

```bash
sudo vi /etc/init.d/etcd
```

Paste the following, replacing `<USER>` as appropriate:

```bash
#!/sbin/openrc-run

name="etcd"
user="<USER>"
description="Etcd Container"
extra_started_commands="status logs"
command_background="no"

depend() {
    need net
    after firewall
    after podman-preparation
}

start() {
    ebegin "Starting ${name} as ${user}"
    export HOME=$(getent passwd ${user} | cut -d: -f6)

    if su -s /bin/sh ${user} -c "podman container exists ${name}"; then
        su -s /bin/sh ${user} -c "podman start ${name}"
    else
        su -s /bin/sh ${user} -c "podman run --name ${name} \
            --restart=on-failure \
            --pod milvus-pod \
            -d \
            -e ETCD_AUTO_COMPACTION_MODE=revision \
            -e ETCD_AUTO_COMPACTION_RETENTION=1000 \
            -e ETCD_QUOTA_BACKEND_BYTES=4294967296 \
            -v /srv/etcd:/etcd:Z \
            quay.io/coreos/etcd:v3.5.18 \
            etcd -advertise-client-urls=http://etcd:2379 \
                 -listen-client-urls http://0.0.0.0:2379 \
                 --data-dir /etcd"
    fi

    eend $?
}

stop() {
    ebegin "Stopping ${name}"
    export HOME=$(getent passwd ${user} | cut -d: -f6)
    su -s /bin/sh ${user} -c "podman stop ${name}"
    eend $?
}

status() {
    export HOME=$(getent passwd ${user} | cut -d: -f6)
    su -s /bin/sh ${user} -c "podman ps -f name=${name}"
}

logs() {
    export HOME=$(getent passwd ${user} | cut -d: -f6)
    su -s /bin/sh ${user} -c "podman logs ${name}"
}
```

Set executable and enable service:

```bash
sudo chmod +x /etc/init.d/etcd
sudo rc-update add etcd
sudo rc-service etcd start
```

### 7.2 Minio

Create Data Directory:

```bash
mkdir -p /srv/minio
```

Create OpenRC Service for Minio:

```bash
sudo vi /etc/init.d/minio
```

Paste the following, replacing `<USER>` as appropriate:

```bash
#!/sbin/openrc-run

name="minio"
user="<USER>"
description="Minio Container"
extra_started_commands="status logs"
command_background="no"

depend() {
    need net
    after firewall
    after podman-preparation
}

start() {
    ebegin "Starting ${name} as ${user}"
    export HOME=$(getent passwd ${user} | cut -d: -f6)

    if su -s /bin/sh ${user} -c "podman container exists ${name}"; then
        su -s /bin/sh ${user} -c "podman start ${name}"
    else
        su -s /bin/sh ${user} -c "podman run --name ${name} \
            --restart=on-failure \
            --pod milvus-pod \
            -d \
            -e MINIO_ACCESS_KEY=minioadmin \
            -e MINIO_SECRET_KEY=minioadmin \
            -v /srv/minio:/minio_data:Z \
            docker.io/minio/minio:RELEASE.2023-03-20T20-16-18Z \
            minio server /minio_data --console-address ':9001'"
    fi

    eend $?
}

stop() {
    ebegin "Stopping ${name}"
    export HOME=$(getent passwd ${user} | cut -d: -f6)
    su -s /bin/sh ${user} -c "podman stop ${name}"
    eend $?
}

status() {
    export HOME=$(getent passwd ${user} | cut -d: -f6)
    su -s /bin/sh ${user} -c "podman ps -f name=${name}"
}

logs() {
    export HOME=$(getent passwd ${user} | cut -d: -f6)
    su -s /bin/sh ${user} -c "podman logs ${name}"
}
```

Set executable and enable service:

```bash
sudo chmod +x /etc/init.d/minio
sudo rc-update add minio
sudo rc-service minio start
```

### 7.3 Milvus

Create Data Directory:

```bash
mkdir -p /srv/milvus
```

Create OpenRC Service for Milvus:

```bash
sudo vi /etc/init.d/milvus
```

Paste the following, replacing `<USER>` as appropriate:

```bash
#!/sbin/openrc-run

name="milvus"
user="<USER>"
description="Milvus Container"
extra_started_commands="status logs"
command_background="no"

depend() {
    need net
    after firewall
    after podman-preparation
    after etcd
    after minio
}

start() {
    ebegin "Starting ${name} as ${user}"
    export HOME=$(getent passwd ${user} | cut -d: -f6)

    if su -s /bin/sh ${user} -c "podman container exists ${name}"; then
        su -s /bin/sh ${user} -c "podman start ${name}"
    else
        su -s /bin/sh ${user} -c "podman run --name ${name} \
            --restart=on-failure \
            --pod milvus-pod \
            -d \
            -e LOG_LEVEL=error \
            -e MINIO_REGION=us-east-1 \
            -e ETCD_ENDPOINTS=etcd:2379 \
            -e MINIO_ADDRESS=minio:9000 \
            -v /srv/milvus:/var/lib/milvus:Z \
            docker.io/milvusdb/milvus:v2.5.12 \
            milvus run standalone"
    fi

    eend $?
}

stop() {
    ebegin "Stopping ${name}"
    export HOME=$(getent passwd ${user} | cut -d: -f6)
    su -s /bin/sh ${user} -c "podman stop ${name}"
    eend $?
}

status() {
    export HOME=$(getent passwd ${user} | cut -d: -f6)
    su -s /bin/sh ${user} -c "podman ps -f name=${name}"
}

logs() {
    export HOME=$(getent passwd ${user} | cut -d: -f6)
    su -s /bin/sh ${user} -c "podman logs ${name}"
}
```

Set executable and enable service:

```bash
sudo chmod +x /etc/init.d/milvus
sudo rc-update add milvus
sudo rc-service milvus start
```

## Order of Services

Recommended service startup order:

```bash
sudo rc-service podman-preparation start
sudo rc-service etcd start
sudo rc-service minio start
sudo rc-service milvus start
```

Or, simply reboot to have OpenRC launch them in the required order.

## Notes

All Podman management should be done as the configured non-root `<USER>`.
