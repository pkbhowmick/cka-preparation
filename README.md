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

### Scenario-11

Question: One of the nodes is in NotReady state because of the kubelet. Fix the issue and make the node as Ready state.

1. Check the nodes

```bash
controlplane $ k get nodes
NAME           STATUS     ROLES           AGE   VERSION
controlplane   Ready      control-plane   22d   v1.28.1
node01         NotReady   <none>          22d   v1.28.1
```

2. ssh into node01 & check the kubelet status. In error scenario kubelet status won't be active.

```bash
node01 $ systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Tue 2023-11-14 10:59:39 UTC; 3 weeks 1 days ago
       Docs: https://kubernetes.io/docs/home/
   Main PID: 26490 (kubelet)
      Tasks: 11 (limit: 2339)
     Memory: 37.2M
     CGroup: /system.slice/kubelet.service
             └─26490 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kub>

Dec 07 06:31:27 node01 kubelet[26490]: I1207 06:31:27.604142   26490 reconciler_common.go:300] "Volume detached for volum>
```

3. Make sure kubelet is running with proper binary path

```bash
node01 $ whereis kubelet
kubelet: /usr/bin/kubelet
node01 $ cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf 
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/default/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
```

4. If any change is needed in the config file then

```bash
node01 $ systemctl daemon-reload & systemctl restart kubelet
[1] 34600
Warning: The unit file, source configuration file or drop-ins of kubelet.service changed on disk. Run 'systemctl daemon-reload' to reload units.
```

### Scenario-12

Question: Create a pod name `secret-pod` of image `busybox:1.31.1` which should keep running for some time.

Secret1 yaml is provided below. Create it and mount into the pod at `/tmp/secret`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret1
data:
  somedata: UG91cmluZzYlRW1vdGljb24lU2N1YmE=    
```

Create a secret `secret2` which should contain user=pulak, pass=1234. These entries should be available inside the pod
as APP_USER and APP_PASS env.

Solution:

1. Create the pod

```bash
k run secret-pod --image=busybox:1.31.1 sleep 1d
```

2. Create the secret from the above yaml

```bash
controlplane $ cat 12.yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret1
data:
  somedata: UG91cmluZzYlRW1vdGljb24lU2N1YmE=
controlplane $ k apply -f 12.yaml
secret/secret1 created
```

3. Create the other secret `secret2`

```bash
k create secret generic secret2 --from-literal="user=pulak" --from-literal="pass=1234"
```

4. Update pod with secret ref. Key parts that will be added:

```yaml
apiVersion: v1
kind: Pod
...
...
spec:
  volumes:
    - name: secret-volume
      secret:
        secretName: secret1
  
  containers:
  - name: container1
    ...
    ...
    volumeMounts:
    - name: secret-volume
      readOnly: true
      mountPath: "/tmp/secret"
    env:
    - name: APP_USER
      valueFrom:
        secretKeyRef:
          name: secret2
          key: user
    - name: APP_PASS
      valueFrom:
        secretKeyRef:
          name: secret2
          key: pass
```

Doc Ref: https://kubernetes.io/docs/concepts/configuration/secret/

### Scenario-13

Question: Create a `Static Pod` with image `nginx:alpine` and have resource requests for `10m` CPU and `20Mi` memory.
Create a NodePort service to expose that static Pod on port 80 and check it has endpoints and reachablt through the internal ip address.

Solution:

1. Get a pod yaml using dry-run:

```bash
controlplane $ k run static-pod --image=nginx:alpine --dry-run=client -o yaml > 13.yaml
controlplane $ cat 13.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: static-pod
  name: static-pod
spec:
  containers:
  - image: nginx:alpine
    name: static-pod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

2. Setting up resource request accordingly:

```bash
controlplane $ cat 13.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: static-pod
  name: static-pod
spec:
  containers:
  - image: nginx:alpine
    name: static-pod
    resources: 
      requests:
        cpu: 10m
        memory: 20Mi
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

3. Create the pod. As this is a static pod, we just need to put this inside manifests folder. Pod will be automatically created.

```bash
controlplane $ pwd
/etc/kubernetes/manifests
controlplane $ cat static-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: static-pod
  name: static-pod
spec:
  containers:
  - image: nginx:alpine
    name: static-pod
    resources: 
      requests:
        cpu: 10m
        memory: 20Mi
  dnsPolicy: ClusterFirst
  restartPolicy: Always
controlplane $ k get pods
NAME                      READY   STATUS    RESTARTS   AGE
static-pod-controlplane   1/1     Running   0          91s
```

4. Create the NodePort service and check the Endpoint

```bash
controlplane $ k expose pod static-pod-controlplane --port=80 --type=NodePort
service/static-pod-controlplane exposed
controlplane $ k get svc
NAME                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes                ClusterIP   10.96.0.1      <none>        443/TCP        6d23h
static-pod-controlplane   NodePort    10.109.9.187   <none>        80:32032/TCP   5s
controlplane $ k get ep 
NAME                      ENDPOINTS         AGE
kubernetes                172.30.1.2:6443   6d23h
static-pod-controlplane   192.168.0.7:80    15s
controlplane $ k get pods -o wide
NAME                      READY   STATUS    RESTARTS   AGE     IP            NODE           NOMINATED NODE   READINESS GATES
static-pod-controlplane   1/1     Running   0          2m55s   192.168.0.7   controlplane   <none>           <none>
```

5. Check the service is accessible via NodePort:

```bash
controlplane $ k get nodes -o wide
NAME           STATUS   ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
controlplane   Ready    control-plane   6d23h   v1.28.4   172.30.1.2    <none>        Ubuntu 20.04.5 LTS   5.4.0-131-generic   containerd://1.6.12
node01         Ready    <none>          6d23h   v1.28.4   172.30.2.2    <none>        Ubuntu 20.04.5 LTS   5.4.0-131-generic   containerd://1.6.12
controlplane $ curl 172.30.1.2:32032
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Doc Ref:
- https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
- https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/


### Scenario-14 (Join a node to the Cluster)

Question: Please join `node01` worker node to the cluster, and you have to deploy a pod in the `node01`, pod name should be `web` and image should be `nginx`.

Solution: 

1. Get a join command from the controlplane node:

```bash
controlplane $ kubeadm token create --print-join-command
kubeadm join 172.30.1.2:6443 --token e2ohcp.trxxo6qxxzriqmwe --discovery-token-ca-cert-hash sha256:533673b654759980b932c982ffe4fb647dab69687385004889743982ff9f8eee 
```

2. ssh to node01 and run the join command

```bash
node01 $ kubeadm join 172.30.1.2:6443 --token e2ohcp.trxxo6qxxzriqmwe --discovery-token-ca-cert-hash sha256:533673b654759980b932c982ffe4fb647dab69687385004889743982ff9f8eee 
[preflight] Running pre-flight checks
error execution phase preflight: [preflight] Some fatal errors occurred:
        [ERROR FileAvailable--etc-kubernetes-kubelet.conf]: /etc/kubernetes/kubelet.conf already exists
        [ERROR Port-10250]: Port 10250 is in use
        [ERROR FileAvailable--etc-kubernetes-pki-ca.crt]: /etc/kubernetes/pki/ca.crt already exists
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
```
note: you can ignore those error but always go to check

3. Check the kubelet status and if not running run it

```bash
node01 $ systemctl status kubelet
node01 $ systemctl start kubelet
```

Docs: https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/

### Scenario-15 (PV & PVC)

Question: Create a PV and PVC. Use it in a Deployment.
Given that:
PV name: web-pv, capacity: 2Gi, hostPath: /vol/data, accessMode: ReadWriteOnce, no Storage class defined
PVC name: web-pvc, ns: production, capacity: 2Gi, accessMode: ReadWriteOnce, no Storage class defined
Deployment name: web-deploy, ns: production, image: nginx:1.14.2, Volume mount path: /tmp/web-data

Solution:

1. Create the PV

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: web-pv
  labels:
    type: local
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/vol/data"
```

```bash
controlplane $ k apply -f 15-pv.yaml
persistentvolume/web-pv created
controlplane $ k get pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
web-pv   2Gi        RWO            Retain           Available                          <unset>                          13s
```

2. Create PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

```bash
controlplane $ k create ns production
namespace/production created
controlplane $ k apply -f 15-pvc.yaml
persistentvolumeclaim/web-pvc created
controlplane $ k get pvc -n production
NAME      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
web-pvc   Pending                                      local-path     <unset>                 8s
```

3. Create deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: production
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      volumes:
      - name: data
        persistentVolumeClaim:
           claimName: web-pvc
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: "/tmp/web-data"
          name: data
```

```bash
controlplane $ k apply -f 15-dpl.yaml
deployment.apps/nginx-deployment created
controlplane $ k get deploy -n production
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   0/3     3            0           9s
controlplane $ k get pvc -n production
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
web-pvc   Bound    pvc-badf3c7e-413b-4907-93ad-2aeff2e626d3   2Gi        RWO            local-path     <unset>                 5m38s
controlplane $ k get pods -n production
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5f599f4f8b-7jfpz   1/1     Running   0          35s
nginx-deployment-5f599f4f8b-dn9lw   1/1     Running   0          35s
nginx-deployment-5f599f4f8b-ftff5   1/1     Running   0          35s
```
