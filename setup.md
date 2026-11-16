# How to build a private cloud for yourself on your own server or even an old laptop
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
```
# step7 (configuring nextcloud on our cloud-app )
```yml
#now on cloud-app install nextcloud:
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

# Add these settings for php
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

#then create the certificate for next-cloud (self-signed certificate)
sudo openssl req -x509 -nodes -days 825 -newkey rsa:2048 \
  -keyout /etc/ssl/private/nextcloud.key -out /etc/ssl/certs/nextcloud.crt \
  -subj "/CN=cloud-app" -addext "subjectAltName=DNS:cloud-app,DNS:localhost,IP:APP_IP"

sudo tee /etc/apache2/sites-available/nextcloud.conf <<'EOF'
<VirtualHost *:80>
  ServerName cloud-app
  DocumentRoot /var/www/nextcloud
  <Directory /var/www/nextcloud>
    Require all granted
    AllowOverride All
    Options FollowSymLinks MultiViews
  </Directory>
</VirtualHost>

<VirtualHost *:443>
  ServerName cloud-app
  DocumentRoot /var/www/nextcloud
  SSLEngine on
  SSLCertificateFile /etc/ssl/certs/nextcloud.crt
  SSLCertificateKeyFile /etc/ssl/private/nextcloud.key
  <Directory /var/www/nextcloud>
    Require all granted
    AllowOverride All
    Options FollowSymLinks MultiViews
  </Directory>
</VirtualHost>
EOF

sudo a2enmod ssl rewrite headers env dir mime
sudo a2dissite 000-default
sudo a2ensite nextcloud
sudo apache2ctl configtest && sudo systemctl restart apache2

#then make apache2(webserver) look for php(nextcloud)files and serve it to the browsers and make the browser accept it.
sudo mkdir -p /etc/systemd/system/apache2.service.d
sudo tee /etc/systemd/system/apache2.service.d/nfs.conf <<'EOF'
[Unit]
RequiresMountsFor=/mnt/cloud-storage
EOF
sudo systemctl daemon-reload

#then connect nextcloud with our database(mariadb) + then set the  trusted-ips to look for + run the cron.php to make maintenance  every 5 mins
findmnt /mnt/cloud-storage            # must show the NFS mount
cd /var/www/nextcloud
sudo -u www-data php occ maintenance:install \
  --database mysql --database-host cloud-db \
  --database-name nextcloud --database-user nextcloud --database-pass 'NC_DB_PASS' \
  --admin-user admin --admin-pass 'NC_ADMIN_PASS' \
  --data-dir /mnt/cloud-storage/nextcloud-data

sudo -u www-data php occ config:system:set trusted_domains 1 --value=localhost:8443
sudo -u www-data php occ config:system:set trusted_domains 2 --value=APP_IP
sudo -u www-data php occ config:system:set trusted_domains 3 --value=cloud-app
sudo -u www-data php occ config:system:set memcache.local --value='\OC\Memcache\APCu'
sudo -u www-data php occ config:system:set default_phone_region --value=EG
sudo -u www-data php occ background:cron
echo '*/5 * * * * php -f /var/www/nextcloud/cron.php' | sudo crontab -u www-data -

#test nextcloud
curk -sk https://localhost/status.php  # installed:true (expected)
#then open the UI on your host
virtctl port-forward vmi/cloud-app 8443:443 -n pri-cloud
# then open https://localhost:8443/  (add localhost:8443 to trusted_domains)

```
# step8 (build a group for our  users,make a private cloud for each ,configure password for each...)
```yml
#enable app package -> teamfolders in order to share a directory between our team that contain data where all of us can see it.
cd /var/www/nextcloud
sudo -u www-data php occ group:add staff
sudo -u www-data env OC_PASS='ChangeMe-Alice1' php occ user:add --password-from-env --display-name="Alice" --group="staff" alice
sudo -u www-data env OC_PASS='ChangeMe-Bob1'   php occ user:add --password-from-env --display-name="Bob"   --group="staff" bob
sudo -u www-data php occ user:setting alice files quota 5GB
sudo -u www-data php occ app:install groupfolders
# password rules and brute-force protection
sudo -u www-data php occ app:enable password_policy
sudo -u www-data php occ app:enable bruteforcesettings
# to reset password of a user
Reset a password: sudo -u www-data --preserve-env=OC_PASS php occ user:resetpassword --password-from-env alice
**Check:**

1. Log in as Alice, upload `test.txt`.
2. On cloud-storage: `sudo find /srv/nfs/share/nextcloud-data/alice/files -type f`. The file must be there.
3. Log in as Bob. He must not see Alice's files, but must see "Company Shared".
```
# step9 (build a monitoring server on our cloud-ops using grafana,prometheus)
```yml
# on the four machines install prometheus-node-exporter
sudo apt install -y prometheus-node-exporter
sudo systemctl enable --now prometheus-node-exporter
curl -s localhost:9100/metrics | head -3
#or using anisble
ansible -i inventory.ini cloud -b -m apt -a "name=prometheus-node-exporter state=present"
###########################################################################
#then osudo apt install -y prometheus prometheus-blackbox-exporter

sudo apt install -y apt-transport-https wget gnupg
sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg >/dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update && sudo apt install -y grafana
sudo systemctl enable --now grafana-servern cloud-ops

#then add these modules to blackbox.yml (in order to make it scrape over http||tcp
sudo tee /etc/prometheus/blackbox.yml <<'EOF'
modules:
  http_2xx_insecure:
    prober: http
    timeout: 10s
    http:
      valid_status_codes: [200]
      tls_config:
        insecure_skip_verify: true
  tcp_connect:
    prober: tcp
    timeout: 5s
EOF
sudo systemctl reset-failed prometheus-blackbox-exporter
sudo systemctl enable --now prometheus-blackbox-exporter
sudo systemctl restart prometheus-blackbox-exporter

curl -s 'localhost:9115/probe?module=tcp_connect&target=192.168.50.14:2049' | grep '^probe_success'
curl -sk 'localhost:9115/probe?module=http_2xx_insecure&target=https://192.168.50.12/status.php' | grep '^probe_success'
#then in /etc/prometheus/prometheus.yml add these
rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  - job_name: private-cloud
    static_configs:
      - targets: ['192.168.50.12:9100']
        labels: {vm: cloud-app}
      - targets: ['192.168.50.13:9100']
        labels: {vm: cloud-db}
      - targets: ['192.168.50.14:9100']
        labels: {vm: cloud-storage}
      - targets: ['localhost:9100']
        labels: {vm: cloud-ops}

  - job_name: blackbox-http
    metrics_path: /probe
    params:
      module: [http_2xx_insecure]
    static_configs:
      - targets: ['https://192.168.50.12/status.php']
        labels: {service: nextcloud}
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115

  - job_name: blackbox-tcp
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
      - targets: ['192.168.50.14:2049']
        labels: {service: nfs}
      - targets: ['192.168.50.13:3306']
        labels: {service: mariadb}
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115
# add these alerts to /etc/prometheus/rules (in order to make our prometheus tell us about it
sudo mkdir -p /etc/prometheus/rules
sudo tee /etc/prometheus/rules/private-cloud.yml <<'EOF'
groups:
  - name: private-cloud
    rules:
      - alert: VMDown
        expr: up{job="private-cloud"} == 0
        for: 2m
        labels: {severity: critical}
        annotations:
          summary: "{{ $labels.vm }} is unreachable"

      - alert: ServiceDown
        expr: probe_success == 0
        for: 2m
        labels: {severity: critical}
        annotations:
          summary: "{{ $labels.service }} check failed ({{ $labels.instance }})"

      - alert: DiskAlmostFull
        expr: node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs|ramfs|devtmpfs"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay|squashfs|ramfs|devtmpfs"} < 0.2
        for: 5m
        labels: {severity: warning}
        annotations:
          summary: "{{ $labels.vm }}: {{ $labels.mountpoint }} below 20% free"

      - alert: LowMemory
        expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.1
        for: 5m
        labels: {severity: warning}
        annotations:
          summary: "{{ $labels.vm }} has under 10% memory available"

      - alert: HighCPU
        expr: (1 - avg by (vm) (rate(node_cpu_seconds_total{mode="idle"}[5m]))) * 100 > 90
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "{{ $labels.vm }} CPU above 90% for 10 minutes"
EOF

promtool check rules /etc/prometheus/rules/private-cloud.yml
promtool check config /etc/prometheus/prometheus.yml
sudo systemctl reload prometheus
# sudo systemctl start apache2  (wait for 2-3 mins untill the stuff return to work)
#start looking for these alerts,endpoints on prometheus open : http://localhost:9090 #on your UI
#then grafana -> open : http://localhost:3000
# login as admin:admin
# go to dashboards and import  1860 (full-exporter dashboard)
# that dashboard will show you the specs of ops-server -> you can extend our dashboard to look for our other servers also .
 
```
# step10 (Hardening the entire project (firewalls,apparmor..)
```yml
# add some stuff to sshd_config.d to hardening it
sudo tee /etc/ssh/sshd_config.d/99-hardening.conf <<'EOF'
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
X11Forwarding no
EOF
sudo sshd -t && sudo systemctl reload ssh
# enable firewalls on all of these servers
# all VMs
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from $OPS to any port 22 proto tcp          # SSH from ops (backups, ansible)
sudo ufw allow from $OPS to any port 9100 proto tcp        # node-exporter scraped by ops
# plus your admin machine/subnet for SSH:
sudo ufw allow from 192.168.50.0/24 to any port 22 proto tcp

# cloud-app
sudo ufw allow 80,443/tcp

# cloud-db: only the app (and ops for the TCP probe) may reach MariaDB
sudo ufw allow from 192.168.50.12 to any port 3306 proto tcp
sudo ufw allow from $OPS to any port 3306 proto tcp

# cloud-storage: only the app (and ops for the probe) may reach NFS
sudo ufw allow from 192.168.50.12 to any port 2049 proto tcp
sudo ufw allow from $OPS to any port 2049 proto tcp

# cloud-ops
sudo ufw allow from 192.168.50.0/24 to any port 3000 proto tcp   # Grafana
sudo ufw allow from 192.168.50.0/24 to any port 9090 proto tcp   # Prometheus

sudo ufw enable
sudo ufw status numbered

#install fail2ban (scans authentication logs + automatically blocks ips with repreated login into the system)
sudo apt install -y fail2ban unattended-upgrades
sudo tee /etc/fail2ban/jail.d/sshd.local <<'EOF'
[sshd]
enabled = true
maxretry = 4
bantime = 1h
EOF
sudo systemctl enable --now fail2ban
sudo dpkg-reconfigure -plow unattended-upgrades
#then harden the nextcloud itself
cd /var/www/nextcloud
sudo -u www-data php occ config:system:set overwriteprotocol --value=https
sudo -u www-data php occ config:system:set default_phone_region --value=EG
sudo -u www-data php occ config:system:set filelocking.enabled --value=true --type=boolean
sudo -u www-data php occ app:enable twofactor_totp
sudo chmod 640 /var/www/nextcloud/config/config.php
sudo -u www-data php occ maintenance:repair
sudo -u www-data php occ setupchecks 2>/dev/null || true
#also do these checks on a daily basis on your database
sudo mysql -e "SELECT user,host FROM mysql.user;"   # no anonymous or remote root users
sudo mysql -e "DROP USER IF EXISTS ''@'localhost'; FLUSH PRIVILEGES;"
#install app-armor
sudo apt install -y apparmor apparmor-utils
sudo aa-status
#if you want to deny certain file,directory
vim /etc/apparmor.d/local/main-file-put-these
# Block reading, writing, and executing a specific file
deny /path/to/secret-file rwklx,

# Block an entire directory and everything inside it
deny /path/to/restricted-folder/** rwklx,
#reload apparmor
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.mysqld

```
# step11(build a backup on our cloud-ops for the entire cloud-servers)
```yml
# on cloud-ops, as root generate that backup-key in order to copy it to the other servers
sudo ssh-keygen -t ed25519  -f /root/.ssh/backup

sudo tee /root/.ssh/config <<'EOF'
Host cloud-app
    User cloud
    IdentityFile /root/.ssh/backup

Host cloud-db
    User db
    IdentityFile /root/.ssh/backup

Host cloud-storage
    User cloud
    IdentityFile /root/.ssh/backup
EOF
sudo chmod 700 /root/.ssh
sudo chmod 600 /root/.ssh/config /root/.ssh/backup
sudo chmod 644 /root/.ssh/backup.pub

for h in cloud-app cloud-db cloud-storage; do sudo ssh-copy-id -i /root/.ssh/backup.pub $h; done
for h in cloud-app cloud-db cloud-storage; do sudo ssh -o BatchMode=yes $h 'sudo -n true && echo OK'; done
#then add that backup script in order to backup the whole servers
sudo mkdir -p /backup/nextcloud
sudo tee /usr/local/bin/nextcloud-backup.sh <<'EOF'
#!/bin/bash
set -euo pipefail

# ---- edit these ----
APP=cloud-app
DB=cloud-db
STORAGE=cloud-storage
DB_NAME=nextcloud
DATA_SRC=/srv/nfs/nextcloud        # data folder on cloud-storage
KEEP_DAYS=7
# --------------------

BASE=/backup/nextcloud
STAMP=$(date +%F_%H%M)
DEST=$BASE/$STAMP
SSH="ssh -o BatchMode=yes"          # user and key come from /root/.ssh/config
OCC="cd /var/www/nextcloud && sudo -u www-data php occ"

mkdir -p "$DEST"
echo "[$(date)] backup start -> $DEST"

# always leave maintenance mode, even when something fails
cleanup() { $SSH $APP "$OCC maintenance:mode --off" || true; }
trap cleanup EXIT

# 1. maintenance mode on (nothing writes while we copy)
$SSH $APP "$OCC maintenance:mode --on"

# 2. database dump (streamed and compressed on cloud-ops)
$SSH $DB "sudo mysqldump --single-transaction $DB_NAME" | gzip > "$DEST/db.sql.gz"

# 3. config directory
mkdir -p "$DEST/config"
rsync -a --numeric-ids -e "$SSH" --rsync-path="sudo rsync" \
  $APP:/var/www/nextcloud/config/ "$DEST/config/"

# 4. data directory (hardlinks to the previous backup keep it small)
LINK=""
[ -d "$BASE/latest/data" ] && LINK="--link-dest=$BASE/latest/data"
mkdir -p "$DEST/data"
rsync -a --numeric-ids --delete -e "$SSH" --rsync-path="sudo rsync" $LINK \
  $STORAGE:$DATA_SRC/ "$DEST/data/"

# 5. sanity checks: a broken backup must never look successful
[ -s "$DEST/db.sql.gz" ] && gzip -t "$DEST/db.sql.gz"
[ -f "$DEST/config/config.php" ]
[ -f "$DEST/data/.ncdata" ]

# 6. mark as latest and prune old backups
ln -sfn "$DEST" "$BASE/latest"
find "$BASE" -maxdepth 1 -mindepth 1 -type d -mtime +$KEEP_DAYS -exec rm -rf {} +

echo "[$(date)] backup OK"
EOF
sudo chmod 700 /usr/local/bin/nextcloud-backup.sh

sudo /usr/local/bin/nextcloud-backup.sh  #this will run the script manually
ls -lh /backup/nextcloud/latest/
du -sh /backup/nextcloud/*
#now we need to automatically run this script every signle day at 2.30 am
sudo tee /etc/systemd/system/nextcloud-backup.service <<'EOF'
[Unit]
Description=Nextcloud backup
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/nextcloud-backup.sh
EOF

sudo tee /etc/systemd/system/nextcloud-backup.timer <<'EOF'
[Unit]
Description=Nightly Nextcloud backup

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now nextcloud-backup.timer
systemctl list-timers | grep nextcloud
journalctl -u nextcloud-backup.service -n 30

#now in case a corruption has been made on these servers so that you can restore the entire data to our system and work again
sudo tee /usr/local/sbin/nextcloud-restore.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
RUN=${1:?usage: nextcloud-restore.sh /srv/backups/nextcloud/<run-or-latest>}
APP=cloud@cloud-app
DB=cloud@cloud-db
STORAGE=cloud@cloud-storage
DATA_DST=/srv/nfs/share/nextcloud-data
SSH="ssh -o BatchMode=yes"
OCC="sudo -u www-data php /var/www/nextcloud/occ"

read -r -p "This OVERWRITES the live Nextcloud with $RUN. Type YES: " ans
[ "$ans" = YES ] || exit 1

$SSH "$APP" "$OCC maintenance:mode --on" || true
$SSH "$APP" "sudo tar -C /var/www/nextcloud -xf -" < "$RUN/config.tar"

$SSH "$DB" "sudo mariadb -e \"DROP DATABASE IF EXISTS nextcloud; CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;\""
gunzip -c "$RUN/nextcloud-db.sql.gz" | $SSH "$DB" "sudo mariadb nextcloud"

rsync -a --delete --numeric-ids -e "$SSH" --rsync-path="sudo rsync" "$RUN/data/" "$STORAGE:$DATA_DST/"

$SSH "$APP" "$OCC maintenance:data-fingerprint"
$SSH "$APP" "$OCC maintenance:mode --off"
$SSH "$APP" "$OCC files:scan --all"
echo "restore finished"
EOF
sudo chmod 700 /usr/local/sbin/nextcloud-restore.sh
```
# step10 (open/close nextcloud)
```yml
# shutdown
cd /var/www/nextcloud && sudo -u www-data php occ maintenance:mode --on   # cloud-app
sudo systemctl stop apache2 && sudo umount /mnt/nextcloud-data            # cloud-app
sudo systemctl stop mariadb && sudo shutdown now                          # cloud-db
sudo systemctl stop nfs-kernel-server && sudo shutdown now                # cloud-storage

# startup: storage first, then db, then app
sudo exportfs -v                                  # cloud-storage
sudo systemctl status mariadb                     # cloud-db
mount | grep nextcloud-data || sudo mount -a      # cloud-app (NFS MUST be mounted before Apache)
cat /mnt/nextcloud-data/.ncdata
sudo systemctl start apache2
cd /var/www/nextcloud && sudo -u www-data php occ maintenance:mode --off
sudo -u www-data php occ status
ssh -N -p 30030 -L 8443:localhost:443 hostname@ip #open it on your UI
# once you get here -> you made 100% of this project (congratulations)
```

