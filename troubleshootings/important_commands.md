# Important commands
```yml
# in order to make the owner of nextcloud-data to be www-data=33 (do the following)
sudo chown 33:33 /srv/nfs/share/nextcloud-data
sudo chmod 750 /srv/nfs/share/nextcloud-data
#note
drwxr-s---  => s here called set group ID it give every file inside nextcloud-data the group of the nextcloud-data itself.
```
- i did that file inside it and here are the output
   -<img width="1008" height="40" alt="image" src="https://github.com/user-attachments/assets/6a06265e-c77f-49ef-92fa-91d501f21e71" />

```yml
sudo mariadb -u root -p   (to open mariadb on your device)
sudo mariadb-secure-installation (to set a password for your server as a root)
```
