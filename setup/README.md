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
   3. Install munge on the worker:
   ```
   ansible-playbook -u user -K post_config.yaml -vv
   ```
   4. Prepare DB for SLURM
   ```
   # In master
   cd /data
   git clone https://github.com/mknoxnv/ubuntu-slurm.git
   # Install prereqs
   sudo apt-get install git gcc make ruby ruby-dev libpam0g-dev libmysqlclient-dev mariadb-server build-essential libssl-dev -y
   sudo gem install fpm
   # Next we set up MariaDB for storing SLURM data:

   sudo systemctl enable mysql
   sudo systemctl start mysql
   sudo mysql -u root

   # Within mysql:

   create database slurm_acct_db;
   create user 'slurm'@'localhost';
   set password for 'slurm'@'localhost' = password('slurmdbpass');
   grant usage on *.* to 'slurm'@'localhost';
   grant all privileges on slurm_acct_db.* to 'slurm'@'localhost';
   flush privileges;
   exit

   cp /data/ubuntu-slurm/slurmdbd.conf /data
   ```

   5. Install slurm
   ```
   cd /data
   wget https://download.schedmd.com/slurm/slurm-21.08.8-2.tar.bz2
   tar xvf slurm-21.08.8-2.tar.bz2
   cd slurm-21.08.8-2
   ./configure --prefix=/data/slurm-build --sysconfdir=/etc/slurm --enable-pam --with-pam_dir=/lib/x86_64-linux-gnu/security/ --without-shared-libslurm
   make -j $(nproc)
   make -j $(nproc) contrib
   make install
   cd ..

   sudo fpm -s dir -t deb -v 1.0 -n slurm-21.08.8-2 --prefix=/usr -C /data/slurm-build .
   sudo dpkg -i slurm-21.08.8-2_1.0_amd64.deb
   ```
   6. Configure slurm
   ```
   sudo mkdir -p /etc/slurm /etc/slurm/prolog.d /etc/slurm/epilog.d /var/spool/slurm/ctld /var/spool/slurm/d /var/log/slurm

   sudo chown slurm /var/spool/slurm/ctld /var/spool/slurm/d /var/log/slurm

   sudo cp /data/ubuntu-slurm/slurmdbd.service /etc/systemd/system/

   sudo cp /data/ubuntu-slurm/slurmctld.service /etc/systemd/system/

   sudo cp /data/slurmdbd.conf /etc/slurm/

   sudo systemctl daemon-reload
   sudo systemctl enable slurmdbd
   sudo systemctl start slurmdbd
   sudo systemctl enable slurmctld
   sudo systemctl start slurmctld
   ```

   7. Install slurm in worker
   ```
   ansible-playbook -u user -K slurm_worker.yaml -vv
   ```

   8. Configure slurm
   ```
   # in master node
   # need open ports for srun
   sudo ufw allow 30000:60000/tcp
   sudo ufw allow 30000:60000/udp
   cp /data/ubuntu-slurm/slurm.conf /data/slurm.conf
   sudo slurmd -C # print out the machine specs.
   # sample: NodeName=storage CPUs=40 Boards=1 SocketsPerBoard=2 CoresPerSocket=10 ThreadsPerCore=2 RealMemory=64027
   ```
   Take this line and put it at the bottom of slurm.conf (we should add lines for all machines)
   Next, setup the gres.conf file. Lines in gres.conf should look like:
   ```
   NodeName=master Name=gpu File=/dev/nvidia0
   NodeName=master Name=gpu File=/dev/nvidia1
   ```
   Finally, we need to copy .conf files on all machines.

   10. Finally copy configs to each machine..
   ```
   sudo cp /data/ubuntu-slurm/cgroup* /etc/slurm/
   sudo cp /data/slurm.conf /etc/slurm/
   sudo cp /data/gres.conf /etc/slurm/

   # reload slurm
   ansible-playbook -u user -K update_configall.yaml -vv
   ansible-playbook -u user -K post_config.yaml -vv
   ```

   11. Create cluster
   ```
   sudo sacctmgr add cluster adacomp-cluster
   ```