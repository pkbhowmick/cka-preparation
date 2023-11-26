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
