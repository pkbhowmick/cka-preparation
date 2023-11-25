# cka-preparation

### etcd backup & restore

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


