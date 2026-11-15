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
```
```
# step3 (conviguring pvc for the servers,snapshots,backups.....)
# step4 (initializing the servers using ansible ....)
# step5 (build an nfs between cloud-app,cloud-storage)
# step6 (build a mariadb server on cloud-db,mariadb-client on cloud-app)
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
