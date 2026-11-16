# How to build a private cloud for yourself on your own servers or even old laptop
# requirements:
1. old machine or even a chunk of disk on your pc
2. be familiar with Linux
3. understand how does nfs(network file sharing) work
4. understand kubernetes clusters
5. understand kubevirt and how to build servers on top of the clusters

---
# step 0 (configuring your server(old machine) )
```yml
- in my case i will work on Ubuntu 24.04.5 LTS
- you can pick any linux os and build it as private cloud for your stuff,your team.
- in my case i built these servers on my cluster (kubernetes cluster)
    - why cluster -> as it give me more privileges than using the  general whole old machine(server)
- i used kubevirt on my cluster to build my servers
- so that you will clone the project:
- then start new cluster on your machine
- then try to build these VMS folder after you understnd how kubevirt works
- then will need to build a pvc from the servers
- also you need to install : iso image for ubuntu or even cloud image (the best choice) then try to build a pvc for it in order to clone that pvc at any server later.
(i will clarify that below)
```
-  these are the servers on my cluster
  -  <img width="453" height="107" alt="image" src="https://github.com/user-attachments/assets/73311cf6-0f73-498a-b4da-6ed89cb656a0" />
---
# step1 (Build a cluster -> in order to be able to build a servers on it using kubevirt plugin)
```
- i will assume that you will build that cluster on your old machine so you don't need any cloud provider as we want to make our own stuff
- so use : kind for this mission (it will work with you for long period without any error )
- note: i am using nvim as a editor your can use your favorite editor like : nano,vi......,visual studio code)
```

```yml
nvim cloud-cluster.yml 
---
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
  extraMounts:
    - host: /some/path/on/your/machine
      containerPath: /var/local-path-provisioner/host_path/worker1
- role: worker
  extraMounts:
    - host: /some/path/on/your/machine
      containerPath: /var/local-path-provisioner
```
# step2 (install the plugins for kubevirt,drivers,linux image ....,
```yml
- install kubevirt on the Kubernetes control machine

export VERSION=$(curl -s https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-operator.yaml
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-cr.yaml
kubectl -n kubevirt wait kv kubevirt --for condition=Available --timeout=10m
#in case the nodes doesn't contain hardware virtualization (VMS) install it
kubectl -n kubevirt patch kubevirt kubevirt --type=merge \
  -p '{"spec":{"configuration":{"developerConfiguration":{"useEmulation":true}}}}'

#install virtctl (cli for start/stop the virtualization)
curl -L -o virtctl https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/virtctl-${VERSION}-linux-amd64
chmod +x virtctl && sudo mv virtctl /usr/local/bin/
virtctl version

# install CDI(containerized data importer)->iports disk images into pvcs
export CDI_VERSION=$(curl -s https://api.github.com/repos/kubevirt/containerized-data-importer/releases/latest | grep tag_name | cut -d '"' -f4)
kubectl apply -f https://github.com/kubevirt/containerized-data-importer/releases/download/${CDI_VERSION}/cdi-operator.yaml
kubectl apply -f https://github.com/kubevirt/containerized-data-importer/releases/download/${CDI_VERSION}/cdi-cr.yaml
kubectl get cdi

#build a namespace,check kubevirt
kubectl create namespace pri-cloud
kubectl get storageclass            # note the name, used as STORAGE_CLASS below
kubectl get pods -n kubevirt

```
# step3 (conviguring pvc for the servers,snapshots,backups.....)
```yml

#make a data volume for your image that you want your servers to work with like: fedora,ubuntu...
vim Datavolume_image.yml
# then put this
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: server_name-disk
  namespace: pri-cloud
spec:
  source:
    http:
      url: https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
  storage:
    storageClassName: any_storage_class
    resources:
      requests:
        storage: 20Gi
#so that -> you will make an image with base of 20G size
# to see your stuff
- kubectl  get dv -n namespace 

# now make a 4 servers (cloud-app,cloud-db,cloud-storage,cloud-ops)
#vim cloud-app.yml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: name-server
  namespace: any-ns 
spec:
  runStrategy: Always
  dataVolumeTemplates:
  - metadata:
      name: cloud-name-root
    spec:
      storage:
        storageClassName: csi-hostpath-sc
        accessModes: ["ReadWriteOnce"]
        volumeMode: Filesystem
        resources:
          requests:
            storage: 10Gi          # must be bigger than the golden image (~7.4Gi)
      source:
        pvc:                       # clone the golden Ubuntu image
          name: name
          namespace: name-of-ns-that-contain-pvc(image)
  - metadata:
      name: cloud-name-data
    spec:
      storage:
        storageClassName: csi-hostpath-sc
        accessModes: ["ReadWriteOnce"]
        volumeMode: Filesystem
        resources:
          requests:
            storage: 2Gi
      source:
        blank: {}
  template:
    metadata:
      labels:
        app: cloud-name
    spec:
      domain:
        cpu:
          cores: 2
        memory:
          guest: 1Gi
        resources:
          requests:
            cpu: "250m"
            memory: 1425Mi
          limits:
            cpu: "1"
            memory: 1425Mi
        devices:
          disks:
          - name: rootdisk
            disk: {bus: virtio}
          - name: datadisk
            serial: appdata01
            disk: {bus: virtio}
          - name: cloudinit
            disk: {bus: virtio}
          interfaces:
          - name: default
            bridge: {}
          - name: private
            bridge: {}
      networks:
      - name: default
        pod: {}
      - name: private
        multus:
          networkName: private-lan
      volumes:
      - name: rootdisk
        dataVolume: {name: cloud-name-root}
      - name: datadisk
        dataVolume: {name: cloud-name-data}
      - name: cloudinit
        cloudInitNoCloud:
          networkData: |
            version: 2
            ethernets:
              all-en:
                match:
                  name: "en*"
                dhcp4: true
          userData: |
            #cloud-config
            hostname: cloud-name
            user: name
            password: any-password
            chpasswd: {expire: false}
            ssh_pwauth: true
            packages:
              - qemu-guest-agent
            runcmd:
              - systemctl enable --now qemu-guest-agent
              - blkid /dev/disk/by-id/virtio-appdata01 || mkfs.ext4 -L appdata /dev/disk/by-id/virtio-appdata01
              - mkdir -p /data
              - echo 'LABEL=appdata /data ext4 defaults,nofail 0 2' >> /etc/fstab
              - mount -a
---
apiVersion: v1
kind: Service
metadata:
  name: cloud-name-ssh
  namespace:  any-ns
spec:
  type: NodePort
  selector:
    app: cloud-name
  ports:
  - name: ssh
    port: 22
    targetPort: 22
    nodePort: 30030
# i will assume that -> name in the manifest above will be the suffix of our servers (app,db,storage,ops)
# apply that server
k apply -f cloud-app.yml

#now let's build a snapshots out of our vms

vim cloud-app-snap.yml
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineSnapshot
metadata:
  name: cloud-app-snap-1
  namespace: name-of-ns
spec:
  source:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: cloud-app

# k apply -f cloud-app-snap.yml
# kubectl get vmsnapshot -n pri-cloud

#now to return that snapshot to be used instead of the current state of our server
kubectl apply -f - <<'EOF'
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineRestore
metadata:
  name: restore-cloud-app-1
  namespace: pri-cloud
spec:
  target:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: cloud-app
  virtualMachineSnapshotName: cloud-app-snap-1
EOF

#some commands you can use it
kubectl get vm,vmi -n pri-cloud
virtctl console cloud-app -n pri-cloud
virtctl ssh cloud@cloud-app -n pri-cloud
virtctl stop cloud-app -n pri-cloud && virtctl start cloud-app -n pri-cloud

```
# step4 (initializing the servers using ansible ....)
```yml
apt install -y ansible
mkdir -p ~/private-cloud/ansible && cd ~/private-cloud/ansible

vim inventory.ini
[app]
cloud-app ansible_host=<app_IP> ansible_user=cloud

[db]
cloud-db ansible_host=<db_IP> ansible_user=db

[storage]
cloud-storage ansible_host=<storage_IP> ansible_user=storage

[ops]
cloud-ops ansible_host=<OPS_IP> ansible_user=ops

[cloud:children]
app
db
storage
ops

#now to access these servers you need to copy the ssh key id to each machine and test ssh connection without password

ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_name
ssh-copy-id -i /~/.ssh/id_name app@app-ip
ssh-copy-id -i /~/.ssh/id_name db@db-ip
ssh-copy-id -i /~/.ssh/id_name storage@storage-ip
ssh-copy-id -i /~/.ssh/id_name ops@ops-ip
ansible -i inventory.ini cloud -m ping     #test connection to these servers

#do the basic tasks using ansible
- hosts: cloud
  become: true
  tasks:
    - name: Set hostname
      hostname:
        name: "{{ inventory_hostname }}"

    - name: Add all VMs to /etc/hosts
      blockinfile:
        path: /etc/hosts
        block: |
          192.168.50.12 cloud-app
          192.168.50.13 cloud-db
          192.168.50.14 cloud-storage
          <OPS_IP> cloud-ops

    - name: Update and install base packages
      apt:
        update_cache: true
        upgrade: dist
        name: [rsync, curl, vim, htop, ufw, fail2ban, unattended-upgrades, chrony]
        state: present

    - name: Passwordless sudo for the admin user
      copy:
        dest: "/etc/sudoers.d/90-{{ ansible_user }}"
        content: "{{ ansible_user }} ALL=(ALL) NOPASSWD:ALL\n"
        mode: "0440"
        validate: "visudo -cf %s"

    - name: Time sync on
      service:
        name: chrony
        state: started
        enabled: true
#then apply it
ansible-playbook -i inventory.ini base.yml
#there's more to add but this is the base of ansible
```
# step5 (build an nfs between cloud-app,cloud-storage)
```yml

#on cloud-storage make these(install nfs-server,make a share directory to the other server)
sudo apt update && sudo apt install -y nfs-kernel-server

sudo mkdir -p /srv/nfs/nextcloud
sudo chown 33:33 /srv/nfs/nextcloud      # 33 = www-data on cloud-app (NFS matches owners by number)
sudo chmod 750 /srv/nfs/nextcloud

echo '/srv/nfs/nextcloud ip-of-cloud-app(rw,sync,no_subtree_check)' | sudo tee -a /etc/exports
sudo exportfs -ra
sudo systemctl enable --now nfs-kernel-server
sudo exportfs -v
#then mount that nfs-folder share on /srv/nfs/nextcloud
lsblk
sudo mkfs.ext4 /dev/vdb
sudo mkdir -p /srv/nfs/nextcloud
echo "UUID=$(sudo blkid -s UUID -o value /dev/vdb) /srv/nfs/nextcloud ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
sudo mount -a

#on the other hand (cloud-app) make the nfs-client that will look for his server
sudo apt install -y nfs-common
#test connection to his server on 2049(nfs-server)
nc -zv 192.168.50.14 2049
showmount -e 192.168.50.14   #this ip in my case

sudo mkdir -p /mnt/nextcloud-data
sudo mount -t nfs4 ip-of-cloud-storage:/srv/nfs/nextcloud /mnt/nextcloud-data

df -h /mnt/nextcloud-data
sudo -u www-data touch /mnt/nextcloud-data/test && ls -l /mnt/nextcloud-data
sudo rm /mnt/nextcloud-data/test

#to make that mount permenant
echo '192.168.50.14:/srv/nfs/nextcloud /mnt/nextcloud-data nfs4 defaults,_netdev,nofail 0 0' | sudo tee -a /etc/fstab
sudo umount /mnt/nextcloud-data && sudo mount -a && mount | grep nextcloud-data

```
# step6 (build a mariadb server on cloud-db,mariadb-client on cloud-app)
```yml
- on cloud-db do the following
#install mariadb-server
sudo apt install -y mariadb-server
sudo mysql_secure_installation
#add these rules to it in order to make it accept client connection from other servers.
sudo tee /etc/mysql/mariadb.conf.d/60-nextcloud.cnf <<'EOF'
[mysqld]
bind-address = DB_IP
character-set-server = utf8mb4
collation-server = utf8mb4_general_ci
transaction-isolation = READ-COMMITTED
binlog_format = ROW
EOF
sudo systemctl restart mariadb
sudo mariadb -e "SELECT @@version, @@tx_isolation, @@bind_address;"

sudo mariadb <<'EOF'
CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'nextcloud'@'APP_IP' IDENTIFIED BY 'NC_DB_PASS';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'APP_IP';
FLUSH PRIVILEGES;
EOF
#now check the server port
sudo ss -tlnp | grep 3306

#on cloud-app install mariadb-client
sudo apt install -y mariadb-client
mariadb -h cloud-db -u nextcloud -p nextcloud -e "SELECT 1;"
#if that work great then we are great.

#now on cloud-app install nextcloud
sudo usermod -aG cloudshare www-data
ls -ln /mnt/cloud-storage            # nextcloud-data must show 33 33

sudo apt update
sudo apt install -y apache2 libapache2-mod-php php php-mysql php-gd php-curl php-mbstring \
  php-xml php-zip php-intl php-bcmath php-gmp php-apcu bzip2 unzip curl

cd /tmp
curl -fLO https://download.nextcloud.com/server/releases/latest.tar.bz2
curl -fLO https://download.nextcloud.com/server/releases/latest.tar.bz2.sha256
diff <(sha256sum latest.tar.bz2 | cut -d' ' -f1) <(cut -d' ' -f1 latest.tar.bz2.sha256) && echo CHECKSUM OK
# if you didn't got OK use this
grep -q "$(sha256sum latest.tar.bz2 | cut -d' ' -f1)" latest.tar.bz2.sha256 && echo CHECKSUM OK
	
sudo tar -xjf latest.tar.bz2 -C /var/www/
sudo chown -R www-data:www-data /var/www/nextcloud

PHPV=$(php -r 'echo PHP_MAJOR_VERSION.".".PHP_MINOR_VERSION;')
sudo tee /etc/php/$PHPV/apache2/conf.d/99-nextcloud.ini <<'EOF'
memory_limit = 512M
upload_max_filesize = 2G
post_max_size = 2G
max_execution_time = 360
opcache.enable = 1
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 10000
opcache.memory_consumption = 128
opcache.save_comments = 1
opcache.revalidate_freq = 60
EOF
sudo tee /etc/php/$PHPV/cli/conf.d/99-nextcloud.ini <<'EOF'
memory_limit = 512M
apc.enable_cli = 1
EOF


```
# step7 (configuring nextcloud on our cloud-app )
# step8 (build a group for our  users,make a private cloud for each ,configure password for each...)
# step9 (build a monitoring server on our cloud-ops using grafana,prometheus)
# step10 (Hardening the entire project (firewalls,SELinux...)
# step11(build a backup on our cloud-ops for the entire cloud)
# step10 (open nextcloud and configure the stuff there)
```
ssh -N -p 30030 -L 8443:localhost:443 hostname@ip
```
# notes
