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
