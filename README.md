# This is the slurm system for adacomp.
### Use ansible as the management tools. (This is used in the setup stage)
##### Upgrade to the latest version. Note that it may take a long time to upgrade the system. 
```
ansible-playbook -u user -K upgrade.yaml -vv
```

Install freeIPA server in [master](https://computingforgeeks.com/install-and-configure-freeipa-server-on-ubuntu/) and freeIPA client in [workers]().

Note: choose centos 8 stream for FreeIPA [docker](https://computingforgeeks.com/run-freeipa-server-in-docker-podman-containers/) if system is newer than ubuntu 20.04.
```
sudo docker build -t freeipa-server -f Dockerfile.centos-8-stream .
```

### FAQ
1. DNS zone XXX already exists in DNS and is handled by server(s).

    This case can be handled by specifying ipa-server-install --allow-zone-overlap option.

# Management Use