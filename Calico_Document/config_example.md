
```
cat << EOF | calicoctl apply -f -
- apiVersion: v1
  kind: profile
  metadata:
    name: net01
    labels:
      role: calico_net01
- apiVersion: v1
  kind: profile
  metadata:
    name: openstack-sg-dcad3bf4-ef1f-4ac3-adcd-6001d4f817a8
    labels:
      role: calico_net01
EOF
```

```
cat << EOF | calicoctl create -f -
- apiVersion: v1
  kind: policy
  metadata:
    name: calico_net01
  spec:
    order: 0
    selector: role == 'calico_net01'
    ingress:
    - action: allow
      source:
        selector: role == 'calico_net01'
    egress:
    - action: allow
```
