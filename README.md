## consumer

### metrics
each namespace that has a prometheus/thanos deployment will need to add the following job to their prometheus's in order to take advantage of the service level instrumentation. The namespace filters should be that of the current `kubernetes-services`, `kubernetes-ingresses` and `kubernetes-pods` job filters.
```yaml
    - job_name: 'linkerd-proxy'
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ["telecom"]
      relabel_configs:
      - source_labels:
        - __meta_kubernetes_pod_container_name
        - __meta_kubernetes_pod_container_port_name
        - __meta_kubernetes_pod_label_linkerd_io_control_plane_ns
        action: keep
        regex: ^linkerd-proxy;linkerd-metrics;kube-system$
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: pod
      # special case k8s' "job" label, to not interfere with prometheus' "job"
      # label
      # __meta_kubernetes_pod_label_linkerd_io_proxy_job=foo =>
      # k8s_job=foo
      - source_labels: [__meta_kubernetes_pod_label_linkerd_io_proxy_job]
        action: replace
        target_label: k8s_job
      # drop __meta_kubernetes_pod_label_linkerd_io_proxy_job
      - action: labeldrop
        regex: __meta_kubernetes_pod_label_linkerd_io_proxy_job
      # __meta_kubernetes_pod_label_linkerd_io_proxy_deployment=foo =>
      # deployment=foo
      - action: labelmap
        regex: __meta_kubernetes_pod_label_linkerd_io_proxy_(.+)
      # drop all labels that we just made copies of in the previous labelmap
      - action: labeldrop
        regex: __meta_kubernetes_pod_label_linkerd_io_proxy_(.+)
      # __meta_kubernetes_pod_label_linkerd_io_foo=bar =>
      # foo=bar
      - action: labelmap
        regex: __meta_kubernetes_pod_label_linkerd_io_(.+)
```

### npols
each namespace using network policies will need to add the following policy in order to allow ingress from the control plane
```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-control-plane-ingress
spec:
  podSelector:
    matchExpressions:
        - key: linkerd.io/proxy-component
          operator: Exists
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              linkerd.io/control-plane-ns: kube-system
        - podSelector:
            matchLabels:
              linkerd.io/control-plane-component: controller
```
this assumes you allow all egress

### services
for a service to consume the service mesh you will need to inject it and its dependencies with the proxy sidecar. after installing the linkerd tool chain this can be achieved by running the following 
```
linkerd inject --linkerd-namespace kube-system --linkerd-cni-enabled --tls=optional grpc-test-server.yaml
```
 where `grpc-test-server.yaml` is the manifest of the service you wish to inject. If you're not interested in tls upgrading drop the `--tls=optional` argument. The above will output the replacement manifest to stdout, apply and commit this as per your normal workflow.

## host

### additional resources

a network policy to allow traffic to ingress into the control plane
```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-meshed-ingress
spec:
  podSelector:
    matchLabels:
      linkerd.io/control-plane-component: controller
  ingress:
    - from:
        - namespaceSelector: {}
          podSelector:
            matchExpressions:
              - key: linkerd.io/proxy-deployment
                operator: Exists
```

a network policy selector to allow traffic to ingress into the namespace hosting the thanos/prometheus query interface
```yaml
    - namespaceSelector:
        matchLabels:
          name: kube-system
```

a scrape job for the linkerd control plane components
```yaml
    - job_name: 'linkerd-controller'
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ['kube-system']
      relabel_configs:
      - source_labels:
        - __meta_kubernetes_pod_label_linkerd_io_control_plane_component
        - __meta_kubernetes_pod_container_port_name
        action: keep
        regex: (.*);admin-http$
      - source_labels: [__meta_kubernetes_pod_container_name]
        action: replace
        target_label: component
```

a scape job for the control plane proxy components
```yaml
    - job_name: 'linkerd-proxy'
       kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels:
        - __meta_kubernetes_pod_container_name
        - __meta_kubernetes_pod_container_port_name
        - __meta_kubernetes_pod_label_linkerd_io_control_plane_ns
        action: keep
        regex: ^linkerd-proxy;linkerd-metrics;kube-system$
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: pod
      # special case k8s' "job" label, to not interfere with prometheus' "job"
      # label
      # __meta_kubernetes_pod_label_linkerd_io_proxy_job=foo =>
      # k8s_job=foo
      - source_labels: [__meta_kubernetes_pod_label_linkerd_io_proxy_job]
        action: replace
        target_label: k8s_job
      # drop __meta_kubernetes_pod_label_linkerd_io_proxy_job
      - action: labeldrop
        regex: __meta_kubernetes_pod_label_linkerd_io_proxy_job
      # __meta_kubernetes_pod_label_linkerd_io_proxy_deployment=foo =>
      # deployment=foo
      - action: labelmap
        regex: __meta_kubernetes_pod_label_linkerd_io_proxy_(.+)
      # drop all labels that we just made copies of in the previous labelmap
      - action: labeldrop
        regex: __meta_kubernetes_pod_label_linkerd_io_proxy_(.+)
      # __meta_kubernetes_pod_label_linkerd_io_foo=bar =>
      # foo=bar
      - action: labelmap
        regex: __meta_kubernetes_pod_label_linkerd_io_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: keep
        regex: (kube-system|sys-.*)
```

a scrape job for consumer proxy components
```yaml
    - job_name: 'linkerd-proxy'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels:
        - __meta_kubernetes_pod_container_name
        - __meta_kubernetes_pod_container_port_name
        - __meta_kubernetes_pod_label_linkerd_io_control_plane_ns
        action: keep
        regex: ^linkerd-proxy;linkerd-metrics;kube-system$
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: pod
      # special case k8s' "job" label, to not interfere with prometheus' "job"
      # label
      # __meta_kubernetes_pod_label_linkerd_io_proxy_job=foo =>
      # k8s_job=foo
      - source_labels: [__meta_kubernetes_pod_label_linkerd_io_proxy_job]
        action: replace
        target_label: k8s_job
      # drop __meta_kubernetes_pod_label_linkerd_io_proxy_job
      - action: labeldrop
        regex: __meta_kubernetes_pod_label_linkerd_io_proxy_job
      # __meta_kubernetes_pod_label_linkerd_io_proxy_deployment=foo =>
      # deployment=foo
      - action: labelmap
        regex: __meta_kubernetes_pod_label_linkerd_io_proxy_(.+)
      # drop all labels that we just made copies of in the previous labelmap
      - action: labeldrop
        regex: __meta_kubernetes_pod_label_linkerd_io_proxy_(.+)
      # __meta_kubernetes_pod_label_linkerd_io_foo=bar =>
      # foo=bar
      - action: labelmap
        regex: __meta_kubernetes_pod_label_linkerd_io_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: drop
        regex: (kube-system|sys-.*)
```
