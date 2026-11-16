# Private cloud infrastructure
A private company file-sharing platform built with Linux,
KubeVirt, Nextcloud, NFS, Ansible, Terraform, Prometheus,
and Grafana.
# stage1
## Infrastructure roles

- cloud-app: Nextcloud application
- cloud-db: MariaDB
- cloud-storage: NFS storage
- cloud-ops: Automation and monitoring

## Objectives

- Centralized company file sharing
- Linux user and access management
- Persistent storage and NFS
- Infrastructure automation
- Monitoring and alerting
- Tested backups and disaster recovery
- Future AWS deployment using Terraform

## Deployment stages

1. Infrastructure design
2. Linux VMs and networking
3. NFS storage
4. Nextcloud
5. Ansible automation
6. Prometheus and Grafana
7. Backup and recovery
8. AWS deployment

----
# stag2 (output+shape of the infrastructure)
- Nextcloud diagram
  - <img width="944" height="884" alt="image" src="https://github.com/user-attachments/assets/164c01c2-2df5-4362-af4a-fc3e8d1f1a35" />
- the entire cluster
  - <img width="893" height="885" alt="image" src="https://github.com/user-attachments/assets/122eb2f5-62cf-4661-a442-973a99a8913a" />

- thanks for kubevirt : https://kubevirt.io/user-guide/architecture/architecture/
