# This is the slurm system for adacomp.
### Use ansible as the management tools. (This is used in the setup stage)
##### Upgrade to the latest version. Note that it may take a long time to upgrade the system. 
```
ansible-playbook -u user -K upgrade.yaml -vv
```

Install IPA Server and client for User and account management.
```
# ssh to the master machine
docker volume create freeipavol
docker-compose up -d # start the freeipa server
```

Install IPA client.
```
ansible-playbook -u user -K ipa_client.yaml -vv

# set hostname accordingly.
# for example, this needs to ssh to the server once on the server side.
hostnamectl set-hostname crane0.d2.comp.nus.edu.sg 
sudo ipa-client-install --hostname=`hostname -f` \
--mkhomedir \
--server=crane3.d2.comp.nus.edu.sg \
--domain d2.comp.nus.edu.sg \
--realm D2.COMP.NUS.EDU.SG
```

Install slurm.
1. Adduser (only need once.)
```
ansible-playbook -u user -K slurm.yaml -vv
```

2. An example to install conda environment in workers only. (Optional)
```
ansible-playbook --user=user conda.yaml -vv
```

3. Install shared storage
   master side
   ```
   # ssh to master
   sudo apt install nfs-kernel-server -y

   # we need to add rules for the shared location. This is done with:
   sudo vim /etc/exports

   # Then adding the line:
   /data *(rw,sync,no_root_squash)

   sudo systemctl start nfs-kernel-server.service
   sudo ufw allow from any to any port nfs
   ```
4. Ready to install slurm now.
   1. Passwordless SSH from master to all workers.
   ```
   ssh-keygen
   ssh-copy-id user@crane0.d2.comp.nus.edu.sg # etc...
   ```
   2. Install munge on the master:
   ```
   sudo apt-get install libmunge-dev libmunge2 munge -y
   sudo systemctl enable munge
   sudo systemctl start munge
   ```
   Test munge if you like: munge -n | unmunge | grep STATUS
   ```
   sudo cp /etc/munge/munge.key /data/
   sudo chown munge /data/munge.key
   sudo chmod 400 /data/munge.key
   ```


# Management Use