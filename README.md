# cka-preparation

### Scenario-1

Ques: etcd backup & restore

backup:

```bash
export ETCDCTL_API=3

etcdctl --endpoints=https://172.18.0.2:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save snapshot.db
```

restore:

```
etcdctl --data-dir /var/lib/etcd-backup snapshot restore snapshot.db
```

### Scenario-2

Scenario: Create a user "pulak" and grant him permission to cluster. "pulak" should only able to create, get, list, delete pods.

1. Create a key/csr for the user:

```bash
openssl genrsa -out pulak.key 2048
openssl req -new -key pulak.key -out pulak.csr -subj "/CN=pulak"
```

2. Create a CSR object:

Example:
```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: pulak
spec:
  request: <base 64 encoded pulak.csr>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
```

3. Approve the csr

```bash
kubectl certificate approve pulak
```

4. Collect the signed certificate from the .status.certificate from the csr object

```bash
kubectl get csr pulak -o jsonpath='{.status.certificate}'| base64 -d > pulak.crt
```

5. Create Role/Cluster Role & RoleBinding/ClusterBinding accordinly

Example using kubectl:
```bash
kubectl create role developer --verb=create --verb=get --verb=list --verb=update --verb=delete --resource=pods
kubectl create rolebinding developer-binding-myuser --role=developer --user=pulak
```

6. Add the user to the kubeconfig

```bash
kubectl config set-credentials pulak --client-key=pulak.key --client-certificate=pulak.crt --embed-certs=true
kubectl config set-context pulak --cluster=kubernetes --user=pulak
kubectl config use-context pulak
```

### Scenario-3

Scenario: A pod is not running state because of available nodes have some taints. Fix it.

To simulate the process:

1. Add taint to nodes:

```bash
kubectl taint nodes node1 key1=value1:NoSchedule
```

2. Create deployment and pod should be pending because of the taint.
```bash
kubectl create deploy nginx --image=nginx:alpine
```

3. Edit deploy to tolerate the taint:

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"

## or

tolerations:
- key: "key1"
  operator: "Exists"
  effect: "NoSchedule"
```

Doc Ref: https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/

### Scenario-4

Question: Create a pod "nginx-cka" using image "nginx" and initContainer "git-cka" with image "alpine/git".
Volume mount path of the main container "/usr/share/nginx/html".
Nginx index.html need to be override with shared volume. index.html file cloned from path
"https://github.com/jhawithu/k8s-nginx.git".

Pod yaml:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-cka
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      ports:
      - containerPort: 80
      volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
  initContainers:
    - name: git-k8s
      image: alpine/git
      args:
      - clone
      - --single-branch
      - --
      - https://github.com/jhawithu/k8s-nginx.git
      - /data
      volumeMounts:
      - mountPath: /data
        name: data
  volumes:
    - name: data
      emptyDir: {}
```

### Scenario-5

Question: Upgrade k8s cluster version from 1.27.x to 1.28.x

1. Install Kubeadm

```bash
apt update
apt install kubeadm=1.28.0-00
```

2. Drain controlplane

```bash
kubectl drain controlplane --ignore-daemonsets
```

3. Upgrade controlplane node
   
```bash
kubeadm upgrade apply v1.28.2
```

4. Update and restart kubectl

```bash
apt install kubelet=1.28.2-00
systemctl restart kubelet
```

5. Uncordon controlplane

```bash
kubectl uncordon controlplane
```

For each worker node:

6. Drain the node

```bash
kubectl drain <node-name> --ignore-daemonsets
```

7. Upgrade worker node

```bash
kubeadm upgrade node
```

8. Upgrade & restart kubelet

```bash
apt install kubelet=1.28.2-00
systemctl restart kubelet
```

9. Uncordon the node

```bash
kubectl uncordon <node-name>
```

### Scenario-6

Question: Create a new pod called "admin-pod" with image busybox. Allow the pod to be able to set system_time.

The container should sleep for 3200 seconds.

1. Create pod yaml using imperative style:

```bash
kubectl run admin-pod --image=busybox --command sleep 3200 --dry-run=client -o yaml
```

2. Add capabilities to container

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: admin-pod
  name: admin-pod
spec:
  containers:
  - command:
    - sleep
    - "3200"
    image: busybox
    name: admin-pod
    securityContext:
      capabilities:
        add: ["SYS_TIME"]
```

Doc Ref: https://kubernetes.io/docs/tasks/configure-pod-container/security-context/

### Scenario-7

Question: Create a namespace "devops" and create a NetworkPolicy that blocks all trafic to pods in
devops namespace, except for traffic from pods in the same namespace on port 8080.

1. Create namespace with a label:

```bash
kubectl create ns devops
kubectl label ns app=devops
kubectl get ns devops --show-labels
```

2. Create NetworkPolicy yaml:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: devops-np
  namespace: devops
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              app: devops
      ports:
        - protocol: TCP
          port: 8080
```

### Scenario-8

Create a NetworkPolicy that denies all access to to the payroll Pod in the accounting namespace.

1. Create network policy assuming the payroll pod has app=payroll label.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payroll-network-policy
  namespace: accounting
spec:
  podSelector:
    matchLabels:
      app: payroll
  policyTypes:
    - Ingress
    - Egress
```
Note: No rule for Ingress & Egress means all denied.

Doc Ref: https://kubernetes.io/docs/concepts/services-networking/network-policies/

### Scenario-9

Setup Liveness, Readiness & Startup Probes

Liveness probe example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: registry.k8s.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```
kubelet will restart the pod if liveness probe fails.

Readiness probe example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: readiness
  name: readiness-exec
spec:
  containers:
  - name: readiness
    image: registry.k8s.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    readinessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

readiness probes are configured exactly same way as liveness probes.

kubelet won't restart the pod in this case rather marks the pod as Not Ready.
So, the service won't send any traffic to this unhealthy pod.

Startup probe example:

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: liveness-port
  failureThreshold: 30
  periodSeconds: 10
```

Startup probe will save the lazy applications from liveness probe failure. It will give apps
some breathing room at the startup before liveness probe check starts.

### Scenario-10

Question: Create a pod with name `project-tiger` of image `httpd:2.4.41-alpine`. Find out in which node the pod in scheduled. 
Using command `crictl`, findout:

1. info.runtimeType of the pod
2. container logs

Solution:

1. Run the pod and find the node

```bash
k run project-tiger --image=httpd:2.4.41-alpine
```

2. ssh into that node and find the container id

```bash
$ crictl ps | grep project-tiger
030b066c10cb3       54b0995a63052       50 seconds ago      Running             project-tiger       0           ed43bf9847f06       project-tiger
```

3. copy the container id & inspect the runtimeType

```bash
$ crictl inspect 030b066c10cb3 | grep runtimeType
    "runtimeType": "io.containerd.runc.v2",
```

4. check the logs using `crictl logs`

```bash
$ crictl logs 030b066c10cb3
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 192.168.1.3. Set the 'ServerName' directive globally to suppress this message
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 192.168.1.3. Set the 'ServerName' directive globally to suppress this message
[Wed Dec 06 08:22:44.832652 2023] [mpm_event:notice] [pid 1:tid 139768630623560] AH00489: Apache/2.4.41 (Unix) configured -- resuming normal operations
[Wed Dec 06 08:22:44.832899 2023] [core:notice] [pid 1:tid 139768630623560] AH00094: Command line: 'httpd -D FOREGROUND'
```

