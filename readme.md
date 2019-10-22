# Personal Multinode Kubernetes Playground.


## Up and Running

```bash

# install addiitonal plugins
vagrant plugin install vagrant-sshfs
vagrant plugin install vagrant-persistent-storage

# up and running
vagrant up

# wrap up
mkdir -p ~/.kube
cp .vagrant/provisioners/ansible/inventory/artifacts/admin.conf ~/.kube/config

# test
kubectl get nodes

> NAME         STATUS   ROLES    AGE   VERSION
> master-1     Ready    master   2m   v1.16.2
> node-lrg-1   Ready    <none>   1m   v1.16.2
> node-lrg-2   Ready    <none>   1m   v1.16.2
> node-lrg-3   Ready    <none>   1m   v1.16.2
> node-mid-1   Ready    <none>   1m   v1.16.2
> node-mid-2   Ready    <none>   1m   v1.16.2
> node-sml-1   Ready    <none>   1m   v1.16.2
> node-sml-2   Ready    <none>   1m   v1.16.2
```

## Customize

Change setting on what you like (and can handle). You can customize: `number` (machines number), `cpu` (cpu to give), `ram` (megabytes of memory to share).

```yaml
# example k8s-nodex.yml contents

node_types:
  master:
    number: 1
    cpu: 2
    ram: 4096

  node-sml:
    number: 2
    cpu: 1
    ram: 2048

  node-mid:
    number: 2
    cpu: 1
    ram: 4096

  node-lrg:
    number: 3
    cpu: 2
    ram: 8192

```

## Ciao!

Happy Hacking!

