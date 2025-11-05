## 55 advanced Kubernetes interview questions

Hey folks, my name is Immanuel and I am a certified Kubernetes administrator. Here, I have collected some questions about Kubernetes that you can use to prepare for interviews or the CKA exam.

So, you think you know Kubernetes? You've deployed a few apps, scaled a few pods, and now you're ready for the big leagues. Let's see how you handle some questions that go beyond the basics. Let's go)

---

### 1. Explain the roles of the Kubernetes control plane components and how they interact to keep the cluster in sync

![Kubernetes components](https://i.postimg.cc/HspkDj4y/image.png)

**Answer:**

* **kube-apiserver** is the gatekeeper for all API calls; it validates and processes requests, persists objects to etcd, and serves as the hub for control-plane communication.
* **etcd** is the distributed key-value store holding the cluster’s desired state.
* **kube-scheduler** monitors unscheduled Pods and binds them to appropriate Nodes based on resource requirements and policies.
* **kube-controller-manager** runs controllers (e.g., ReplicaSet, Endpoint) that reconcile actual state with the desired state.
* **cloud-controller-manager** handles cloud-specific controllers (e.g., load-balancer provisioning) when running in a cloud environment.
  They form a feedback loop: the API server writes to etcd; controllers and the scheduler watch etcd, act, and write back via the API server.

### 2. How do you perform a rolling update without downtime?

**Answer:**

This is where Deployments really shine. A rolling update allows you to update your application with zero downtime by incrementally replacing old Pods with new ones. You can control the process with `maxSurge` and `maxUnavailable`.

- **`maxSurge`**: The maximum number of Pods that can be created above the desired number of Pods.
- **`maxUnavailable`**: The maximum number of Pods that can be unavailable during the update.

**Command:**

To trigger a rolling update, you can use `kubectl set image`:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1 --record
```

To monitor the rollout status:

```bash
kubectl rollout status deployment/nginx-deployment
```

**YAML Example (Rolling Update Strategy):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: nginx
  template:
    # ... rest of the template
```

**And some additional info:** the `--record` flag in commands like `kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1 --record` **has been deprecated** and is planned to be removed in future versions of kubectl.

Currently, if you run the command without `--record`, the change cause is not recorded automatically. There is no direct built-in alternative flag that replicates `--record` functionality exactly. Instead, the recommended approach is to manually annotate the resource with the change cause using `kubectl annotate`, for example:

```bash
kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="Updated nginx image to 1.16.1"
```

This manual annotation serves as the alternative way to record the reason for changes in deployments and other resources.

### 3. How would you design a highly available etcd cluster for Kubernetes, and secure it?

![ETCD](https://i.postimg.cc/SNKY22pn/image.png)
**Answer:**

* Deploy odd-numbered etcd members (3 or 5) across failure domains to avoid split-brain.
* Use mutual TLS for client-to-server and peer-to-peer communication, generating certificates per member.
* Enable etcd encryption at rest for Kubernetes secrets (see Q24).
* Regularly back up etcd snapshots and use `etcdctl snapshot save`/restore.

### 4. What's the difference between a Service and an Ingress?

Service:

![Service](https://i.imgur.com/h63dlNA.png)



Ingress:

![Ingress](https://i.imgur.com/MWc7mrW.png)



**Answer:**

Think of a **Service** as the internal load balancer for your Pods. It provides a stable IP address and DNS name for a set of Pods, so you don't have to worry about individual Pod IPs changing. There are different types of Services (`ClusterIP`, `NodePort`, `LoadBalancer`), but they all handle internal traffic.

An **Ingress**, on the other hand, is an API object that manages external access to the services in a cluster, typically HTTP. It acts as a reverse proxy and can provide load balancing, SSL termination, and name-based virtual hosting. You need an Ingress Controller (like NGINX or Traefik) for the Ingress resource to work.

**In short:** Service is for internal traffic, Ingress is for external traffic.

**YAML Example (Ingress):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          service:
            name: test
            port:
              number: 80
```



### 5. Describe the CRI, CNI, and CSI interfaces and their roles in Kubernetes

![CRI, CNI, and CSI](https://i.postimg.cc/T2zQz7tc/image.png)

**Answer:**

* **Container Runtime Interface (CRI):** Pluggable interface between kubelet and container runtimes (Docker, containerd, CRI-O).
* **Container Network Interface (CNI):** Defines how Pods get networking (IP assignment, routes) via plugins (Calico, Flannel, Cilium).
* **Container Storage Interface (CSI):** Standard for volume plugins, enabling dynamic provisioning, snapshotting, and at-rest encryption for block and file storage.



### 6. How would you troubleshoot a Pod that is stuck in a `CrashLoopBackOff` state?

![](https://i.postimg.cc/3w6YYQ4R/image.png)

**Answer:**

Ah, the dreaded `CrashLoopBackOff`. This means your Pod is starting, crashing, and then Kubernetes is trying to restart it, only for it to crash again. Here's the troubleshooting checklist:

1. **Describe the Pod:** This will give you information about the Pod's state, events, and any error messages.

   ```bash
   kubectl describe pod <pod-name>
   ```

3. **Check the image:** Make sure the container image is correct and exists. A typo in the image name is a common culprit.

4. **Check resource limits:** If your application is exceeding its memory or CPU limits, it might be getting killed by the kubelet.

5. **Check liveness and readiness probes:** If your probes are misconfigured, they might be killing your Pod prematurely.



### 7. What are Taints and Tolerations?

![Taints and tolerations](https://i.imgur.com/5HNILxt.png)

**Answer:**

**Taints** are applied to nodes, and they repel Pods that don't have a matching **Toleration**. This is a way to control which Pods can be scheduled on which nodes.

For example, you might want to dedicate a set of nodes to a specific application or prevent certain Pods from running on a particular node.

**Command (Taint a node):**

```bash
kubectl taint nodes node1 key=value:NoSchedule
```

**YAML Example (Toleration in a Pod):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: nginx
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

### 8. Explain how to deploy a custom scheduler and integrate it with the main scheduling framework or extenders

**Answer:**

* Build your scheduler binary (e.g., using client-go) that implements `Scheduler` interface.
* Deploy it as a Deployment in its own namespace.
* In Pod spec, set `spec.schedulerName: custom-scheduler`.
* (Optional) Use the Scheduling Framework’s plugin model by writing and registering a plugin via `--plugin-config` to kube-scheduler.
* Extenders can be configured in the kube-scheduler config to call external HTTP endpoints for additional predicate/prioritization logic.

### 9. What's the role of the etcd in a Kubernetes cluster?

![ETCD](https://i.imgur.com/6FFxhfx.png)

**Answer:**

`etcd` is the brain of your Kubernetes cluster. It's a consistent and highly-available key-value store used as Kubernetes' backing store for all cluster data. Everything you see in your cluster—Pods, Services, Deployments, Secrets—is stored in `etcd`.

If `etcd` goes down, your cluster becomes read-only. You can't create or update any resources. That's why it's so critical to have a reliable and backed-up `etcd` cluster.

### 10. How can you debug CRI issues using `crictl`?

**Answer:**

```bash
# List pods/containers managed by CRI:
crictl pods
crictl ps -a

# Pull an image manually:
crictl pull nginx:latest

# Inspect container logs:
crictl logs <container-id>

# Exec into a paused container:
crictl exec -it <container-id> sh

# Check runtime version:
crictl version
```

`crictl` talks directly to the CRI socket, bypassing kubelet.

### 11. How do you manage secrets in Kubernetes?

**Answer:**

Kubernetes **Secrets** are objects that let you store and manage sensitive information, such as passwords, OAuth tokens, and ssh keys. They are stored as base64 encoded strings, which is *not* encryption. Anyone with access to `etcd` can read them.

For more secure secret management, you should consider using a tool like **HashiCorp Vault**, **AWS Secrets Manager**, or **Azure Key Vault**, and then use a Kubernetes operator to inject those secrets into your Pods.

**Command (Create a secret):**

```bash
kubectl create secret generic my-secret --from-literal=password='s3cr3t'
```

**YAML Example (Using a secret in a Pod):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: nginx
    env:
    - name: SECRET_PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: password
```

### 12. Describe the Kubernetes networking model, including CNI and NetworkPolicies

**Answer:**

* Flat network: every Pod gets a unique IP, and any Pod can reach any other Pod by default.
* CNIs implement this: they allocate IPs and set up routes.
* **NetworkPolicy** objects define ingress/egress rules at the Pod level, enforced by the CNI plugin (e.g., Calico or Cilium).

### 13. Explain the difference between `livenessProbe` and `readinessProbe`

![Liveness probe and readiness probe](https://i.postimg.cc/j5JZqLq9/image.png)

**Answer:**

Both are used to check the health of a container, but they have different purposes:

- **`livenessProbe`**: Checks if the container is running. If the probe fails, the kubelet kills the container and restarts it. This is useful for catching deadlocks or other situations where the application is running but not responding.
- **`readinessProbe`**: Checks if the container is ready to accept traffic. If the probe fails, the container's IP address is removed from the endpoints of all Services that match the Pod. This is useful for when an application needs some time to start up before it can handle requests.

**YAML Example (Probes):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: my-app
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 3
      periodSeconds: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

### 14. Explain how a service mesh (e.g., Istio) integrates with Kubernetes and show an example `VirtualService`

![Virtual Service](https://miro.medium.com/v2/resize:fit:1400/1*PW_ZjLoKcAxVTimGJRhXyQ.png)

**Answer:**

* Service mesh runs sidecar proxies (Envoy) in Pods to intercept traffic.
* Control plane (Pilot, Galley) uses CRDs to configure routing and policies.
* Example:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-vs
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: "jason"
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```

This routes user `jason` to `v2` of the reviews service.

### 15. What is a Persistent Volume (PV) and a Persistent Volume Claim (PVC)?

![PVC and PV](https://cdn.prod.website-files.com/636dbee261df29040c8db281/6524d92b073ef164e0cde694_Dynamic-volume-provisioning-using-storage-classes.png)

**Answer:**

In Kubernetes, Pods are ephemeral. When a Pod dies, its data is lost. To solve this, we have **Persistent Volumes (PVs)** and **Persistent Volume Claims (PVCs)**.

- **Persistent Volume (PV)**: A piece of storage in the cluster that has been provisioned by an administrator. It's a resource in the cluster, just like a CPU or memory.
- **Persistent Volume Claim (PVC)**: A request for storage by a user. It's similar to how a Pod consumes CPU and memory. A PVC consumes PV resources.

This separation of concerns allows developers to request storage without needing to know the details of the underlying storage infrastructure.

**YAML Example (PV and PVC):**

```yaml
# Persistent Volume
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"

---
# Persistent Volume Claim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```

### 16. How do you write and deploy a ValidatingAdmissionWebhook? Provide a sample manifest

![Validating admission webhook and controller](https://kmitevski.com/wp-content/uploads/2021/08/Untitled-Diagram2.png)

**Answer:**

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: no-latest-tag
webhooks:
- name: no-latest.example.com
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE","UPDATE"]
    resources: ["pods"]
  clientConfig:
    service:
      name: webhook-svc
      namespace: default
      path: "/validate"
    caBundle: <base64-CA-cert>
```

This rejects any Pod using the `:latest` image tag in your validation logic.

### 17. How does network policy work in Kubernetes?

**Answer:**

By default, all Pods in a Kubernetes cluster can communicate with each other. **Network Policies** allow you to restrict this communication. They are like a firewall for your Pods.

You can define rules that specify which Pods are allowed to communicate with which other Pods. Network Policies are implemented by a network plugin, so you need to be using a networking solution that supports them, like Calico or Cilium.

**YAML Example (Network Policy):**

This policy allows traffic to Pods with the label `app=db` only from Pods with the label `app=backend`.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
```

### 18. With PSP deprecated, how do you enforce security standards via PodSecurityAdmission?

**Answer:**

```bash
kubectl label ns default \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=baseline
```

This enforces the “restricted” profile on the `default` namespace, blocking privileged containers and hostPath volumes.

### 19. What is a Sidecar container and what is it used for?

![Sidecar container and another use for an additional container](https://matthewpalmer.net/kubernetes-app-developer/multi-container-pod-design.png)

**Answer:**

A **Sidecar** is a container that runs alongside your main application container in the same Pod. They share the same network namespace and can share volumes. This pattern is used to extend or enhance the functionality of the main container without changing its code.

Common use cases for sidecars include:

- **Logging:** A sidecar can collect logs from the main application and forward them to a central logging system.
- **Monitoring:** A sidecar can export metrics from the application.
- **Service Mesh:** In a service mesh like Istio, a sidecar proxy (like Envoy) is injected into each Pod to handle traffic management, security, and observability.

**YAML Example (Sidecar for logging):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-with-sidecar
spec:
  containers:
  - name: main-app
    image: my-app
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log
  - name: sidecar-container
    image: fluentd
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log
  volumes:
  - name: shared-logs
    emptyDir: {}
```

### 20. Demonstrate a `StorageClass` for CSI-based dynamic provisioning

![Storage Class](https://i.postimg.cc/RZn1zF18/image.png)

**Answer:**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-sc
provisioner: csi.example.com
parameters:
  type: gp2
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

This uses the CSI driver `csi.example.com` to provision volumes on-demand.

### 21. How would you secure a Kubernetes cluster?

**Answer:**

Securing a Kubernetes cluster is a multi-layered process. Here are some key areas to focus on:

- **RBAC (Role-Based Access Control):** Use RBAC to control who can access the Kubernetes API and what they can do. Be as specific as possible with permissions.
- **Network Policies:** As discussed earlier, use Network Policies to restrict traffic between Pods.
- **Pod Security Policies (or their successor):** Enforce security standards for Pods, such as preventing them from running as root or accessing the host network.
- **Secrets Management:** Use a secure solution for managing secrets, not just the default Kubernetes Secrets.
- **Image Scanning:** Scan your container images for vulnerabilities before deploying them.
- **Regularly update Kubernetes:** Keep your cluster up to date with the latest security patches.
- **Secure `etcd`:** Restrict access to `etcd` and encrypt its data at rest.

### 22. Compare `kube-proxy` modes and show how to enable IPVS

**Answer:**

* **iptables** mode uses Linux iptables rules; works universally but less performant at scale.
* **IPVS** uses the Linux IPVS load-balancer; highly efficient for thousands of services.
  To enable IPVS in `kube-proxy` ConfigMap:

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-proxy
  namespace: kube-system
data:
  mode: "ipvs"
```

### 23. What is a Custom Resource Definition (CRD)?

**Answer:**

A **Custom Resource Definition (CRD)** allows you to extend the Kubernetes API with your own custom resources. This is a powerful way to build custom automation and workflows on top of Kubernetes.

When you create a CRD, you are essentially creating a new type of resource that you can manage with `kubectl`, just like you would with built-in resources like Pods and Deployments. To make these custom resources useful, you typically write a **custom controller** (or **operator**) that watches for changes to your custom resources and takes action based on them.

**YAML Example (CRD):**

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
```

### 24. How do you write a custom controller using `controller-runtime`? Outline the key components

**Answer:**

* Scaffold a project with Kubebuilder.
* Define a `Reconciler` struct implementing `Reconcile(ctx, req)`.
* Use `mgr.GetClient()` to `Get`, `Update`, `Patch` resources.
* Register your controller with `builder.ControllerManagedBy(mgr)`.
* Deploy as a Deployment and RBAC in-cluster.

### 25. What is Helm and why would you use it?

![Helm](https://miro.medium.com/v2/resize:fit:1200/1*mClrYLFakC6B6f62vVnhcA.png)

**Answer:**

**Helm** is a package manager for Kubernetes. It allows you to define, install, and upgrade even the most complex Kubernetes applications. Think of it like `apt` or `yum` for Kubernetes.

Helm uses a packaging format called **charts**. A chart is a collection of files that describe a related set of Kubernetes resources. With Helm, you can:

- Manage the complexity of application deployments.
- Easily share and reuse applications.
- Perform repeatable deployments.
- Manage application releases.

### 26. How do you enforce policies with OPA Gatekeeper? Provide a `ConstraintTemplate` and `Constraint`

![Constraint Template and Constraint](https://miro.medium.com/v2/resize:fit:1400/0*FmfaihS_LjhywVuX.png)

**Answer:**

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels
      violation[{"msg": msg}] {
        input.review.object.metadata.labels["app"] == ""
        msg := "Every resource must have an ‘app’ label"
      }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-app-label
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
```

This rejects Pods without an `app` label.

### 27. How do you drain a node for maintenance?

**Answer:**

To safely perform maintenance on a node, you should first **drain** it. This will cordon the node (mark it as unschedulable) and then evict all the Pods running on it. The evicted Pods will be rescheduled on other available nodes, as long as they are managed by a controller like a Deployment.

**Command:**

```bash
kubectl drain <node-name> --ignore-daemonsets
```

The `--ignore-daemonsets` flag is often necessary because DaemonSet Pods are not evicted.

After maintenance is complete, you can uncordon the node to allow Pods to be scheduled on it again:

```bash
kubectl uncordon <node-name>
```

### 28. Explain API aggregation and provide an `APIService` example

**Answer:**

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
spec:
  group: metrics.k8s.io
  version: v1beta1
  service:
    name: metrics-server
    namespace: kube-system
  caBundle: <base64-CA-cert>
  groupPriorityMinimum: 100
  versionPriority: 10
```

This makes the custom `metrics.k8s.io` API available through the main API server.

### 29. What is the role of the kube-proxy?

![kube-proxy](https://us1.discourse-cdn.com/flex016/uploads/kubernetes/original/2X/1/10265ce5af254f993e116f8d78852922d63dc716.png)

**Answer:**

`kube-proxy` is a network proxy that runs on each node in your cluster. It is responsible for implementing the Kubernetes Service concept. It maintains network rules on nodes that allow network communication to your Pods from network sessions inside or outside of your cluster.

`kube-proxy` can operate in several modes, such as `iptables`, `ipvs`, or `userspace`. The `iptables` mode is the default and most common.

### 30. How do you configure an HPA to scale on a Prometheus metric `rps`?

**Answer:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: rps
      target:
        type: AverageValue
        averageValue: 100
```

Install and configure `prometheus-adapter` with rules mapping PromQL to `custom.metrics.k8s.io`.

### 31. What are StatefulSets and how are they different from Deployments?

![Deployment vs StatefulSets](https://habrastorage.org/webt/9l/0a/av/9l0aavsk3bu_6unlc71-osxxeeq.png)

**Answer:**

**StatefulSets** are used to manage stateful applications, such as databases. They are similar to Deployments, but they provide a few key features that are important for stateful apps:

- **Stable, unique network identifiers:** Pods in a StatefulSet have a persistent, unique hostname based on their name and an ordinal index (e.g., `web-0`, `web-1`).
- **Stable, persistent storage:** Each Pod in a StatefulSet gets its own persistent storage that is tied to its identity. If a Pod is rescheduled, it will be reattached to the same storage.
- **Ordered, graceful deployment and scaling:** Pods in a StatefulSet are created, updated, and deleted in a specific order.
- **Ordered, automated rolling updates.**

**YAML Example (StatefulSet):**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"
  replicas: 3
  template:
    # ...
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

### 32. Show a `ServiceMonitor` to scrape metrics from a Pod

![ServiceMonitor](https://user-images.githubusercontent.com/100563973/173386686-fa3be8bd-4ae4-44c6-9393-e47f70fc1693.png)

**Answer:**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-service-monitor
  labels:
    team: frontend
spec:
  selector:
    matchLabels:
      app: frontend
  endpoints:
  - port: http-metrics
    path: /metrics
    interval: 30s
```

Prometheus Operator watches this CRD and configures scraping accordingly.

### 33. How can you limit resource usage for a Pod?

**Answer:**

You can limit the CPU and memory resources that a Pod can use by setting **resource requests and limits** in the Pod's specification.

- **Requests:** The amount of resources that are guaranteed for the container. If a container requests a resource, Kubernetes will only schedule it on a node that can provide that amount.
- **Limits:** The maximum amount of resources that a container can use. If a container tries to exceed its limit, it may be terminated (for memory) or throttled (for CPU).

**Resource requests and limits for Pod may look like:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: my-app
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### 34. How do you use ephemeral containers to debug a running Pod?

**Answer:**

```bash
kubectl debug -it my-pod \
  --image=nicolaka/netshoot \
  --target=my-pod \
  -- /bin/bash
```

This attaches a debug container into the Pod’s namespace, letting you run tools like `tcpdump` or check in-Pod logs.

### 35. What is a DaemonSet?

![Daemonset](https://i.postimg.cc/c4VRgrpM/image.png)

**Answer:**

A **DaemonSet** ensures that all (or some) nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to them. As nodes are removed from the cluster, those Pods are garbage collected.

This is useful for deploying cluster-wide services, such as:

- A log collector like Fluentd or Logstash.
- A node monitoring agent like Prometheus Node Exporter.
- A cluster storage daemon like Glusterd or Ceph.

**YAML Example (DaemonSet):**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        # ...
```

### 36. How do you rotate component certificates using Kubernetes CSR API?

**Answer:**

* Generate CSR manifest:

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: kube-apiserver-csr
spec:
  request: <base64-CSR>
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
```

* Create with `kubectl apply -f csr.yaml`.
* Approve: `kubectl certificate approve kube-apiserver-csr`.
* Fetch signed cert: `kubectl get csr kube-apiserver-csr -o jsonpath='{.status.certificate}' | base64 -d > apiserver.crt`.

### 37. How do you troubleshoot a service that is not accessible?

**Answer:**

If you can't access a Service, here's a troubleshooting guide:

1. **Check the Service description:** Make sure the Service exists and has the correct labels.

   ```bash
   kubectl describe service <service-name>
   ```

2. **Check the endpoints:** Verify that the Service has endpoints (i.e., it's connected to running Pods).

   ```bash
   kubectl get endpoints <service-name>
   ```

   If there are no endpoints, check the label selector in your Service and the labels on your Pods. They must match.

3. **Check the Pods:** Ensure that the Pods targeted by the Service are running and healthy.

4. **Check Network Policies:** If you are using Network Policies, make sure they are not blocking traffic to the Service.

5. **Check DNS:** Try to resolve the Service's DNS name from another Pod in the cluster.

   ```bash
   kubectl exec -it <another-pod> -- nslookup <service-name>
   ```

### 38. Demonstrate `postStart` and `preStop` hooks in a Pod

**Answer:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: demo
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh","-c","echo 'Started' > /usr/share/message"]
      preStop:
        exec:
          command: ["/bin/sh","-c","nginx -s quit"]
```

`postStart` runs right after container start; `preStop` runs before termination.

### 39. What is the Operator pattern?

**Answer:**

The **Operator pattern** is a way to package, deploy, and manage a Kubernetes application. An Operator is a custom controller that uses Custom Resource Definitions (CRDs) to manage an application and its components.

The goal of an Operator is to automate the entire lifecycle of a complex, stateful application, including:

- Deployment and configuration.
- Scaling and high availability.
- Backups and recovery.
- Upgrades.

Essentially, an Operator encodes the operational knowledge of a human operator into software.

### 40. How do you deploy VPA for automatic resource right-sizing?

**Answer:**

```bash
kubectl apply -f https://github.com/kubernetes/autoscaler/raw/master/vertical-pod-autoscaler/deploy/vpa-v1-crd-gen.yaml
kubectl apply -f vpa-rbac.yaml
```

Then create:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: sample-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: sample-deployment
  updatePolicy:
    updateMode: "Auto"
```

VPA’s Recommender, Updater, and Admission Controller components handle resource adjustments.

### 41. How would you handle a situation where a node is `NotReady`?

**Answer:**

A `NotReady` status means the node controller has not heard from the node within the `node-monitor-grace-period`. This could be due to a network partition, the kubelet crashing, or the node being down.

1. **Describe the node:** Get more information about the node's status and any events.

   ```bash
   kubectl describe node <node-name>
   ```

2. **Check the kubelet:** SSH into the node and check the status of the kubelet service.

   ```bash
   systemctl status kubelet
   ```

   Check the kubelet logs for errors:

   ```bash
   journalctl -u kubelet
   ```

3. **Check network connectivity:** Ensure the node can communicate with the master nodes.

4. **Check resources:** The node might be under heavy resource pressure (CPU, memory, or disk), which could be preventing the kubelet from functioning correctly.

### 42. How do you write a Falco rule to detect shells inside containers?

**Answer:**

```yaml
- rule: Detect Shell in Container
  desc: Alert when bash is spawned in a container
  condition: container.id != host and proc.name = bash
  output: "Shell in container (user=%user.name container=%container.id)"
  priority: WARNING
```

Apply via `falco_rules.local.yaml` and restart Falco.

### 43. What are init containers and when would you use them?

**Answer:**

**Init containers** are containers that run before the main application containers in a Pod. They must run to completion before the main containers are started. If an init container fails, the Pod will be restarted.

You can use init containers for setup tasks that need to complete before the application starts, such as:

- Waiting for a database or another service to be available.
- Cloning a git repository into a volume.
- Performing database migrations.
- Setting up necessary permissions or configuration files.

**YAML Example (Init Container):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
spec:
  containers:
  - name: my-app-container
    image: busybox
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
```

### 44. How do you enable etcd at-rest encryption for Secrets?

**Answer:**

1. Create `/etc/kubernetes/enc/encryptionConfig.yaml`:

   ```yaml
   apiVersion: apiserver.config.k8s.io/v1
   kind: EncryptionConfiguration
   resources:
     - resources:
       - secrets
       providers:
       - aescbc:
           keys:
           - name: key1
             secret: <base64-32-byte-key>
       - identity: {}
   ```
2. Edit `/etc/kubernetes/manifests/kube-apiserver.yaml` to add:

   ```bash
   - --encryption-provider-config=/etc/kubernetes/enc/encryptionConfig.yaml
   - --encryption-provider-config-type=aescbc
   ```
3. Restart API server pods; now Secrets are stored encrypted in etcd.

### 45. How does Kubernetes handle container runtime interfaces (CRI)?

![CRI](https://imgur-backup.hackmd.io/htiLYpz.png)

**Answer:**

The **Container Runtime Interface (CRI)** is a plugin interface that enables the kubelet to use a wide variety of container runtimes, without the need to recompile. The kubelet acts as a client, and the CRI shim for a given runtime acts as the server.

This allows you to use different container runtimes like **containerd** or **CRI-O** instead of the historically used Docker (via dockershim, which is now deprecated). This modularity is a key strength of Kubernetes.

### 46. How do you capture traffic between two Pods using `netshoot` and `tcpdump`?

**Answer:**

```bash
# Launch netshoot sidecar
kubectl debug -it my-pod --image=nicolaka/netshoot --target=my-pod

# Inside netshoot shell:
tcpdump -i any -nn dst port 80 -w /tmp/traffic.pcap
```

Or use `kubectl netshoot run tmp-shell` via plugin.

### 47. What is the difference between a `ClusterIP`, `NodePort`, and `LoadBalancer` service type?

![Node Port and Load Balancer](https://i.postimg.cc/gk3mv7wX/image.png)

**Answer:**

These are the three main types of Kubernetes Services:

- **`ClusterIP`**: This is the default service type. It exposes the Service on an internal IP in the cluster. This IP is only reachable from within the cluster.
- **`NodePort`**: This exposes the Service on a static port on each node's IP. You can access the Service from outside the cluster by requesting `<NodeIP>:<NodePort>`. `ClusterIP` is automatically created.
- **`LoadBalancer`**: This exposes the Service externally using a cloud provider's load balancer. `NodePort` and `ClusterIP` services are automatically created, and the external load balancer routes to them.

### 48. Show basic Velero commands to back up and restore a namespace

**Answer:**

```bash
# Install Velero (once)
velero install --provider aws --bucket velero-backups --secret-file ./credentials-velero

# Backup namespace
velero backup create nginx-backup --include-namespaces nginx-example

# Simulate disaster
kubectl delete ns nginx-example

# Restore
velero restore create --from-backup nginx-backup
```

For PV snapshotting, add `--snapshot-volumes`.

### 49. How do you manage cluster upgrades?

**Answer:**

Upgrading a Kubernetes cluster is a critical operation that needs to be done carefully. The general process is:

1. **Read the release notes:** Carefully read the release notes for the version you are upgrading to. Pay attention to any deprecated APIs or breaking changes.
2. **Upgrade the master nodes:** Upgrade the master components (`kube-apiserver`, `kube-scheduler`, `kube-controller-manager`) one at a time.
3. **Upgrade the kubelet on worker nodes:** After the master nodes are upgraded, you can upgrade the `kubelet` on the worker nodes. This is typically done by draining each node, performing the upgrade, and then uncordoning it.
4. **Upgrade `kube-proxy`:** Upgrade the `kube-proxy` DaemonSet.
5. **Upgrade other components:** Upgrade other cluster components like CoreDNS, CNI plugin, etc.

Using a managed Kubernetes service (like GKE, EKS, or AKS) can greatly simplify this process.

### 50. How do you renew all kubeadm-managed certificates?

**Answer:**

```bash
# Check expirations
kubeadm certs check-expiration

# Renew all certs
kubeadm certs renew all

# Restart control-plane static pods (kube-apiserver, controller-manager, scheduler)
```

On HA clusters, run on each control-plane node.

### 51. What is a Pod Disruption Budget (PDB)?

![Pod Disruption Budget](https://i.postimg.cc/HLyJ5zC6/image.png)

**Answer:**

A **Pod Disruption Budget (PDB)** limits the number of Pods of a replicated application that are simultaneously down from voluntary disruptions. For example, when you drain a node, the PDB will ensure that a certain number of Pods for your application remain running.

This is crucial for maintaining the availability of your applications during planned maintenance.

**YAML Example (PDB):**

This PDB ensures that at least 2 Pods with the label `app=nginx` are available at all times.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: nginx
```

### 52. How do you configure the audit policy and log backend for the API server?

**Answer:**

1. Create `/etc/kubernetes/audit-policy.yaml`:

   ```yaml
   apiVersion: audit.k8s.io/v1
   kind: Policy
   rules:
   - level: Metadata
   ```
2. Mount in `kube-apiserver` static Pod and add flags:

   ```
   --audit-policy-file=/etc/kubernetes/audit-policy.yaml
   --audit-log-path=/var/log/kubernetes/audit/audit.log
   --audit-webhook-config-file=/etc/kubernetes/audit-webhook.yaml
   ```
3. Optionally configure batching and webhook.

### 53. How can you control the scheduling of a Pod to a specific node?

**Answer:**

There are several ways to control where a Pod gets scheduled:

- **`nodeSelector`**: The simplest way. You add a `nodeSelector` field to your Pod specification with a set of key-value pairs. The Pod will only be scheduled on nodes that have all of those labels.
- **`nodeAffinity`**: A more expressive way to specify node selection. It allows you to use more complex rules, like "soft" preferences (`preferredDuringSchedulingIgnoredDuringExecution`) and more advanced operators.
- **`podAffinity` and `podAntiAffinity`**: These allow you to schedule Pods based on the labels of other Pods that are already running on a node. For example, you can co-locate Pods of the same service on the same node (`podAffinity`) or spread them across different nodes (`podAntiAffinity`).
- **Taints and Tolerations**: As discussed earlier, this is another mechanism to control scheduling.

**YAML Example (nodeAffinity):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
```

### 54. Demonstrate how to create a `PriorityClass` and a high-priority Pod that can preempt lower ones

![PriorityClass](https://d2908q01vomqb2.cloudfront.net/fe2ef495a1152561572949784c16bf23abb28057/2023/01/12/pic-01-flow-1.png)

**Answer:**

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "Critical workload"
---
apiVersion: v1
kind: Pod
metadata:
  name: critical-pod
spec:
  containers:
  - name: app
    image: nginx
  priorityClassName: high-priority
```

Higher-priority pods enter the scheduling queue first, and preempt lower ones if resources are scarce.

### 55. What is the role of the cloud-controller-manager?

**Answer:**

The **cloud-controller-manager** is a Kubernetes control plane component that embeds cloud-specific control logic. It allows you to link your cluster into your cloud provider's API, and separates out the components that interact with that cloud platform from components that just interact with your cluster.

It is responsible for things like:

- **Node Controller:** For checking the cloud provider to determine if a node has been deleted in the cloud after it stops responding.
- **Route Controller:** For setting up routes in the underlying cloud infrastructure.
- **Service Controller:** For creating, updating, and deleting cloud provider load balancers.

---

Well, that's all for now. I hope you successfully answered all 55 questions and found the answers useful, helping to increase your understanding of Kubernetes. 

If some questions were too difficult or if you were unfamiliar with any of these Kubernetes concepts, feel free to share your thoughts in the comments.

And if you want to get in touch with me, you can find me on X (twitter) — @immanuel_vibe. Bye!
