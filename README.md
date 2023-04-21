# lima setup

## installation and configuration
- `brew install lima socket_vmnet docker docker-compose docker-credential-helper`
- Edit `~/.lima/_config/networks.yaml` to update `shared` network range and set `socketVMNet` to `/opt/homebrew/Cellar/socket_vmnet/VERSION/bin/socket_vmnet` where `VERSION` is the actual version (lima does not like symlinks here, unfortunately)
- `limactl sudoers > /tmp/lima.sudo`
- `sudo install -o root /tmp/lima.sudo /etc/sudoers.d/lima`
- `rm -f /tmp/lima.sudo`
- `networks.yaml` changes require you to update sudoers file

## Run Docker locally (arm64 + amd64 containers on M1/M2)
- create the `local-docker.yaml` configuration template for lima-vm:
- create the VM: `limactl start --tty=false --name docker local-docker.yaml`
- follow the instructions at the end to test Docker locally
- test running an amd64-only container: `docker run --name cassandra bitnami/cassandra:latest`
- from another window run: `docker exec cassandra ps aux` 
  - you should see a bunch of `/mnt/lima-rosetta/rosetta` processes running

## k3s master/control plane node

## k3s worker node
