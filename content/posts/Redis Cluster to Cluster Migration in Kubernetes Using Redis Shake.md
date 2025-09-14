+++
date = '2025-09-14T13:45:29+05:30'
draft = false
title = 'Redis/Valkey Cluster to Cluster Migration in Kubernetes Using Redis Shake'
+++

## Overview :

Migrating data between Redis clusters in Kubernetes can be challenging, especially with high availability and minimal downtime requirements. This guide walks you through performing a **Redis cluster-to-cluster migration** using [**Redis Shake**](https://github.com/tair-opensource/RedisShake) — a powerful data migration and synchronization tool by `tair-opensource`.

Redis Shake supports both **full data migration** and **real-time incremental synchronization**, making it ideal for production migrations or ongoing cross-cluster replication scenarios. It works with both Redis and Redis-compatible databases like **Valkey**.

## Deployment Options for Redis Shake in Kubernetes

To run Redis Shake in Kubernetes, you can follow one of the two deployment approaches described below:

### Approach 1: Ephemeral BusyBox Pod with Manual Redis Shake Setup

This method is quick and lightweight, suitable for ad-hoc or one-off migrations.

***Steps:***
1. Create a BusyBox pod with a long-running sleep command to keep it alive:
```bash
kubectl run redis-shake — image=busybox — command — sleep 3600
```
2. Exec into the pod:
```bash
kubectl exec -it redis-shake — sh
```
3. Download Redis Shake:
```bash
wget https://github.com/tair-opensource/RedisShake/releases/download/v4.4.0/redis-shake-v4.4.0-linux-amd64.tar.gz
```
4. Extract the tarball:
```bash
tar -zxvf redis-shake-v4.4.0-linux-amd64.tar.gz
```


### Approach 2 (Recommended): Custom Docker Image with Redis Shake Pre-Installed

This approach is more reusable and production-friendly.

***Dockerfile:***

```bash
**Stage 1: Download Redis Shake**
FROM golang:1.20 as builder
WORKDIR /app
RUN apt-get update && apt-get install -y wget
RUN wget https://github.com/tair-opensource/RedisShake/releases/download/v4.4.0/redis-shake-v4.4.0-linux-amd64.tar.gz && \
    tar -xvzf redis-shake-v4.4.0-linux-amd64.tar.gz

**Stage 2: Build a minimal BusyBox image with the binary**
FROM busybox:latest
WORKDIR /app
COPY --from=builder /app/redis-shake /usr/local/bin/redis-shake
ENTRYPOINT ["redis-shake"]
```

## TLS Setup for Secure Redis Clusters

If your Redis clusters are configured with TLS, additional setup is required to mount certificates and secrets securely.

### If you’re following Approach 1:
- Mount the TLS certificates for both source and destination Redis clusters as volumes.
- Pass the password secret (if applicable) as an environment variable in the pod.

### If you’re following Approach 2:
- Create a Kubernetes job using the custom Docker image you built.
- Mount the TLS certificates as volumes.
- Provide the password (if required) as an environment variable.

## Create the shake.toml configuration file
With Redis Shake binary available and the necessary TLS certificates mounted inside the container, the next step is to create shake.toml file.

This file defines how Redis Shake connects to both the **source** and **target** Redis clusters. It includes essential parameters like:
- Cluster mode
- Authentication details
- TLS configurations
Default [shake.toml](https://github.com/tair-opensource/RedisShake/blob/v4/shake.toml) Configuration

Let’s create a ConfigMap for `shake.toml` so we can mount it into the pod.

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-shake-config
  namespace: default
data:
  shake.toml: |
    # Redis-Shake Configuration for Valkey Migration/Sync

    # Source Redis/Valkey configuration
    [sync_reader]
    cluster = true                            # Using cluster instead of standalone
    address = "redis-cluster-0.redis-cluster-headless:6379" 
    password = "${REDIS_PASSWORD}"
    tls = true
    tls_config = {ca = "/etc/redis/certs/ca.crt", cert = "/etc/redis/certs/tls.crt", key = "/etc/redis/certs/tls.key"}

    # Target Redis/Valkey configuration  
    [redis_writer]
    cluster = true                            # Using cluster instead of standalone                            
    address = "valkey-cluster-0.valkey-cluster-headless:6379"
    password = "${VALKEY_PASSWORD}"               # Using env variable for password                          
    tls = true                            # Enable TLS
# tls certificates are not required for this dest cluster, because `tls-auth-clients` is not set to true for this valkey cluster
# tls_config = {ca = "/etc/valkey/certs/ca.crt", cert = "/etc/valkey/certs/tls.crt", key = "/etc/valkey/certs/tls.key"}
```

## Sample Job Manifest :
```bash
apiVersion: batch/v1
kind: Job
metadata:
  name: redis-shake-migration
  namespace: default
spec:
  ttlSecondsAfterFinished: 100
  template:
    metadata:
      labels:
        app: redis-shake
    spec:
      containers:
      - name: redis-shake
        image: redis-shake-custom-image
        imagePullPolicy: Always
        command: ["redis-shake", "/etc/redis-shake/shake.toml"]
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        env:
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: valkey-cluster-password
              key: password
        volumeMounts:
        - name: redis-certs
          mountPath: /etc/redis/certs
          readOnly: true
        - name: valkey-certs
          mountPath: /etc/valkey/certs
          readOnly: true
        - name: shake-config
          mountPath: /etc/redis-shake
          readOnly: true
      restartPolicy: Never
      volumes:
      - name: redis-certs
        secret:
          secretName: redis-cluster
      - name: valkey-certs
        secret:
          secretName: valkey-cluster
      - name: shake-config
        configMap:
          name: redis-shake-config
```
**Redis Shake will :**
- Perform an initial full data sync from the source Redis cluster to the target.
- Enter incremental sync mode, where it continuously streams new writes, updates, and deletes from the source to the target in real-time.

Once you’re confident that the target cluster is fully caught up, you can safely redirect traffic from source Redis to the target.

**Note:** You can also migrate from or to a single-node Redis instance by setting the cluster = false option in the shake.toml file for the source or target, as needed.