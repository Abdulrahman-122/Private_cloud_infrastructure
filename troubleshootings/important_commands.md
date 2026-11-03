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
- to make sure that nfs is running on your server,client
```yml
- on the server
sudo systemctl status nfs-kernel-server
- if your server isn't active
sudo systemctl enable --now nfs-kernel-server
-verify exports as active
sudo exportfs -v
- look for the port
sudo ss -tulpn | grep -E '2049|111'

- on your client
sudo systemctl status rpcbind
- activate it if it not
sudo systemctl enable --now rpcbind
- check port
nc -zv <SERVER_IP_ADDRESS> 111
- also add some files on the shared file and check whether the clien will read them and vice verse
```
- on the client i made that
- <img width="1029" height="162" alt="image" src="https://github.com/user-attachments/assets/3782024e-7265-49b1-9bab-3d24b480ba99" />
- on ther server the file moves there at the same time
- <img width="546" height="59" alt="image" src="https://github.com/user-attachments/assets/92814672-ca3c-437a-898f-773d90091d02" />
