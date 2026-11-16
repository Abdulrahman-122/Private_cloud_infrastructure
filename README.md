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
- <img width="1920" height="1064" alt="image" src="https://github.com/user-attachments/assets/72a03ab1-3c37-4b05-a32b-8b20886c3bee" />
- <img width="1909" height="881" alt="image" src="https://github.com/user-attachments/assets/50c3126a-b266-4a70-8ac5-99ec8e292910" />
- <img width="1909" height="881" alt="image" src="https://github.com/user-attachments/assets/e3f2b55a-21c4-4b16-8de7-62ac60cc1cef" />
- <img width="1919" height="877" alt="image" src="https://github.com/user-attachments/assets/95c8bc62-aa2b-4725-bf81-69cf0c4f6c0e" />
- <img width="1499" height="856" alt="image" src="https://github.com/user-attachments/assets/9cd17d42-968d-4451-8e17-abc9202db415" />
- <img width="1908" height="797" alt="image" src="https://github.com/user-attachments/assets/41e0fba7-53ea-4deb-9b06-01efc7548128" />
- <img width="1920" height="826" alt="image" src="https://github.com/user-attachments/assets/6a80769d-ccc6-4b6f-806f-fed1e776c038" />
- <img width="1919" height="351" alt="image" src="https://github.com/user-attachments/assets/82b6e1cb-acdd-4076-94aa-59313260f7da" />

go to this file (if you want to build this project): setup.md
- listen to these videos to see the outer image of this project: https://drive.google.com/drive/folders/1z9xigBwCSrbOY5_4HtW7SQjAdJGQaU1V?usp=sharing
- thanks for kubevirt : https://kubevirt.io/user-guide/architecture/architecture/
