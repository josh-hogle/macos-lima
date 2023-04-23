# lima setup


## Why Not....

- Docker Desktop - requires a commercial license now; requires k3d for multi-node k8s cluster
- Rancher Desktop - allows you to run containers but does not allow you to run / configure the VM running Docker/containerd quite as much; requires k3d for multi-node k8s cluster
- Fusion - Doesn't support Rosetta 2 passthru; not free
- Parallels - Doesn't support Rosetta 2 passthru; not free
- UTM - has issues communicating across NAT'd network currently; requires more manual configuration
- docker CLI - you may run into scenarios where you need to run a container image only available for amd64

## installation and configuration
- `brew install lima socket_vmnet docker docker-compose docker-credential-helper helm`
- Edit `~/.lima/_config/networks.yaml` to update `shared` network range and set `socketVMNet` to `/opt/homebrew/Cellar/socket_vmnet/VERSION/bin/socket_vmnet` where `VERSION` is the actual version (lima does not like symlinks here, unfortunately)
- `limactl sudoers > /tmp/lima.sudo`
- `sudo install -o root /tmp/lima.sudo /etc/sudoers.d/lima`
- `rm -f /tmp/lima.sudo`
- `networks.yaml` changes require you to update sudoers file

## Known bugs
- stopping VM may result in panic for goroutine (BUG in upstream vz in MacOS Ventura); just CTRL+C and ignore it

## Run Docker from your MacOS Desktop (arm64 + amd64 containers on M1/M2)
- create the VM: `limactl start --tty=false --name docker lima/templates/docker-desktop.yaml`
- follow the instructions at the end to test Docker locally
- test running an amd64-only container: `docker run --rm --name cassandra bitnami/cassandra:latest`
- from another window run: `docker exec cassandra ps aux` 
  - you should see a bunch of `/mnt/lima-rosetta/rosetta` processes running
- stop the container: `docker stop cassandra`

## k3s master/control plane node
- create the VM
  ```
  CLUSTER_CIDR=10.240.0.0/16
  SERVICE_CIDR=10.241.0.0/16
  EXTRA_SANS="lima-k8s.internal"
  KUBECONFIG_CONTEXT=lima-k8s
  KUBECONFIG_CLUSTER=lima-k8s

  limactl start --tty=false --name k3s-master-01 \
    --set ".env.CLUSTER_CIDR = \"${CLUSTER_CIDR}\" | \
      .env.SERVICE_CIDR = \"${SERVICE_CIDR}\" | \
      .env.EXTRA_SANS = \"${EXTRA_SANS}\" | \
      .env.KUBECONFIG_CONTEXT = \"${KUBECONFIG_CONTEXT}\" | \
      .env.KUBECONFIG_CLUSTER = \"${KUBECONFIG_CLUSTER}\"" \
    lima/templates/k3s-master.yaml
  ```
- follow the instructions at the end to test connectivity to k8s cluster
- alternate hosts method of connectivity, if desired
- if you have multiple k8s clusters to connect to, add to config.d or config and set KUBECONFIG appropriately
- add functions to zshrc/bashrc for using `kubeconfig`
  
## k3s worker node
- create the worker VMs:
  ```
  MASTER_IP=$(limactl shell k3s-master-01 sudo ifconfig lima0 | grep 'inet ' | awk '{print $2}')
  CLUSTER_TOKEN=$(limactl shell k3s-master-01 sudo cat /var/lib/rancher/k3s/server/node-token)
  for i in 1 2 3; do
    limactl start --tty=false --name k3s-worker-0$i \
      --set ".env.MASTER_IP = \"${MASTER_IP}\" | .env.CLUSTER_TOKEN = \"${CLUSTER_TOKEN}\"" \
      lima/templates/k3s-worker.yaml
  done
  ```

## workloads
- k3s dashboard: https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/
- komodor: https://app.komodor.com
- metallb: https://metallb.universe.tf/installation/

## remove node from cluster

- Remove the node: `kubectl drain <node-name>`
  - You might have to ignore daemonsets and local-data in the machine:
    `kubectl drain <node-name> --ignore-daemonsets --delete-local-data`
- Run: `kubectl delete node <node-name>`