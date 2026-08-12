+++
title = 'Homelab'
date = 2026-08-11T09:53:39-07:00
draft = true
author = "Ceald"
authorTwitter = "" #do not include @
cover = "/kubenetes-meme-resized.png"
coverOnly = true
tags = ["new", "kubernetes","homelab", "k3s", "k8s"]
keywords = ["homelab", "kubernetes","homelab", "k3s", "k8s"]
description = "an elitist homelab guide to homelabbing"
showFullContent = false
readingTime = true
hideComments = false
summary = "malware notes"
toc = true
+++

# Background
After messing with K3S for a bit I was kind of sick of dealing with the lack of resources on a PI 4 so I brought it to my laptop.


# So We Begin!

![alt text](https://media1.tenor.com/m/ns6SqcQI7WwAAAAC/here-we-go-joker.gif)

First and most important step is to pick the Kubernetes distribution for the right job, I went with K3S for because it comes with traefik but I'd use something like RKE2 or Kubeadm instead because of the customizability next time.

Second is to pick a CNI/Network plugin for Kubernetes, depending on the distribution it might already come with one or maybe you'd want to switch it out. I personally used Cilium because of its observability and Hubble UI. Without a CNI your cluster won't be able to communicate with anything inside or outside it.

Third is to pick a storage class for persistence, I went with longhorn because of the frontend for it and flexability, longhorn is also distributed/replicated rather than just being local on a node.

Fourth it'd be nice to have a frontend for managing everything. Something like either rancher or headlamp would be best. I went with headlamp because of how lightweight it is. The rest after this is optional but if you want the most Kuber Kubernetes cluster it'd be a good idea to set these up too.

## Side Note
The next services that are going to be setup is overkill for a basic homelab but this is an elitist homelab so nothing is overkill!

## Other services
Getting metrics with grafana is almost always a must so installing the kubestack is a good idea and loki. These were my values.yaml files:
```yaml
# kube-stack-values.yaml
USER-SUPPLIED VALUES:
grafana:
  additionalDataSources: []
  persistence:
    accessModes:
      - ReadWriteOnce
    enabled: true
    size: 10Gi
    storageClassName: longhorn
    type: pvc
  resources:
    # Main Grafana container
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
persistence:
  storageClass: longhorn

prometheus:
  prometheusSpec:
    scrapeInterval: 30s

    serviceMonitorSelectorNilUsesHelmValues: false
    serviceMonitorSelector: {}
    serviceMonitorNamespaceSelector: {}

    podMonitorSelectorNilUsesHelmValues: false
    podMonitorSelector: {}
    podMonitorNamespaceSelector: {}

    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 50Gi
          storageClassName: longhorn

    resources:
      requests:
        cpu: 500m
        memory: 512Mi
      limits:
        cpu: 1000m
        memory: 2Gi
```
and loki:
```yaml
# loki-values.yaml
loki:
  commonConfig:
    replication_factor: 1

  auth_enabled: false

  isDefault: false

  schemaConfig:
    configs:
      - from: "2024-01-01"
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: loki_index_
          period: 24h

  storage:
    type: filesystem
    bucketNames:
      chunks: chunks
      ruler: ruler
      admin: admin

  limits_config:
    allow_structured_metadata: true

deploymentMode: SingleBinary

singleBinary:
  replicas: 1

gateway:
  enabled: false

read:
  replicas: 0

write:
  replicas: 0

backend:
  replicas: 0

querier:
  replicas: 0

queryFrontend:
  replicas: 0

queryScheduler:
  replicas: 0

ingester:
  replicas: 0
distributor:
  replicas: 0
compactor:
  replicas: 0
lokiCanary:
  enabled: false
chunksCache:
  enabled: false
resultsCache:
  enabled: false
test:
  enabled: false
monitoring:
  dashboards:
    enabled: false
  rules:
    enabled: false
  serviceMonitor:
    enabled: false
promtail:
  config:
    snippets:
      extraScrapeConfigs: |
        - job_name: tetragon
          static_configs:
            - targets:
                - localhost
              labels:
                job: tetragon
                __path__: /var/run/cilium/tetragon/tetragon.log
          pipeline_stages:
            - json:
                expressions:
                  process_exec: process_exec
                  process_exit: process_exit
                  process_kprobe: process_kprobe
  extraVolumeMounts:
    - mountPath: /var/run/cilium/tetragon
      name: tetragon-logs
      readOnly: true
  extraVolumes:
    - hostPath:
        path: /var/run/cilium/tetragon
      name: tetragon-logs
```
another thing that would compliment cilium well is tetragon! Here's my values.yaml file:
```yaml
# tetragon-values.yaml
export:
  stdout:
    enabledCommand: true
    enabledArgs: true
  filenames:
    - /var/log/tetragon/tetragon.json

tetragon:
  prometheus:
    metricsLabelFilter: "namespace,workload,binary" # "pod" label is disabled
    serviceMonitor:
      enabled: true
    port: 2222 # default is 2112
tetragonOperator:
  prometheus:
    serviceMonitor:
      enabled: true
    port: 3333 # default is 2113

```
you'll get metrics and logs exported so you can view them in grafana.

It's probably apparent that you won't be able to access anything because you don't have any ingresses setup in the cluster yet, that's where kyverno and cert manager will come into play.

Install kyverno and cert manager then create a cluster issuer and everything else for cert manager like:
```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned-issuer
  namespace: cert-manager
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: orbit-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: orbit-ca
  secretName: orbit-ca-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-issuer
    kind: Issuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: orbit-ca-cluster-issuer
spec:
  ca:
    secretName: orbit-ca-secret # adjust to match your actual CA secret
```
this issuer is cluster wide btw so you can get certs signed from this issuer anywhere.

For kyverno policy:
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: force-external-dns-target
spec:
  admission: true
  background: true
  emitWarning: false
  mutateExistingOnPolicyUpdate: true
  rules:
  - match:
      any:
      - resources:
          kinds:
          - Ingress
    mutate:
      patchStrategicMerge:
        metadata:
          annotations:
            cert-manager.io/cluster-issuer: orbit-ca-issuer
            external-dns.alpha.kubernetes.io/target: 127.0.0.1
      targets:
      - apiVersion: networking.k8s.io/v1
        kind: Ingress
    name: set-target-annotation
    skipBackgroundRequests: true
  validationFailureAction: Audit
```
You don't need the external dns annotation in the policy if you're not going to use external dns but if you do it'd be a good idea to also get etcd or a source for external dns to write to.

The policy will mutate all your ingresses to have the orbit-ca-issuer added so you don't need to manually add it everytime you make an ingress.

Now for your first ingress on the cluster, you'd write something like this:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
  name: prometheus-grafana-ingress
  namespace: monitoring
spec:
  ingressClassName: traefik
  rules:
  - host: dashboard.orbit.orbit
    http:
      paths:
      - backend:
          service:
            name: prometheus-grafana
            port:
              number: 80
        path: /
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - dashboard.orbit.orbit
    secretName: grafana-tls
```
If you get that ingress like: `kubectl get ingress -n monitoring -o yaml` you'll notice that it's been changed! If not then add this cluster role:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kyverno-background-controller-namespace
  labels:
    rbac.kyverno.io/aggregate-to-background-controller: "true"
rules:
  - apiGroups:
      - ""
    resources:
      - namespaces
    verbs:
      - get
      - list
      - watch
      - update
      - patch
```
Nice, not even halfway done yet! I won't post ALL my ingresses because that'd be insane.

![alt text](https://media1.tenor.com/m/wGufiBV_pI0AAAAC/hide-the-pain-harold-pain.gif)
