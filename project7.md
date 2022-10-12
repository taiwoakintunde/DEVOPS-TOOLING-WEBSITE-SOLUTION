## The prerequisites for this project.
Launch 5 instances in AWS EC2 
- 3 Web servers using RedHat OS
- 1 Database server using Ubuntu OS
- 1 NFS server using RedHat OS
- Create and attach 3 EBS volumes for NFS server 
![](images/2.PNG)

### Step 1: prepare NFS Server
Launch an instance for NFS server 
![](images/3.PNG)

Create 3 EBS(Elastic Block Store) volumes with 10G and attach them to nfs server
![](images/4.PNG)

ssh to nfs server using Windows shell
![](images/5.PNG)

use lsblk command to inspect what block devices are attached to the server
`lsblk`
![](images/6.PNG)

Use gdisk utility to create a single partition on each of the 3 disks
`sudo gdisk /dev/xvdf`
`sudo gdisk /dev/xvdg`
`sudo gdisk /dev/xvdh`
![](images/7.PNG)

Use `lsblk` utility to view the newly configured partition on each of the 3 disks.
![](images/8.PNG)

Install lvm2 package using `sudo yum install lvm2 -y`
![](images/9.PNG)

Run `sudo lvmdiskscan` command to check for available partitions
![](images/10.PNG)

Use pvcreate utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM
`sudo pvcreate /dev/xvdf1`
`sudo pvcreate /dev/xvdg1`
`sudo pvcreate /dev/xvdh1`
![](images/11.PNG)

Verify that your Physical volume has been created successfully by running `sudo pvs`
![](images/12.PNG)

Use vgcreate utility to add all 3 PVs to a volume group (VG). Name the VG webdata-vg
`sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1`
![](images/13.PNG)

Verify that your VG has been created successfully by running `sudo vgs`
![](images/14.PNG)

Use lvcreate utility to create 3 logical volumes. apps-lv (Use half of the PV size), and logs-lv Use the remaining space of the PV size. NOTE: apps-lv will be used to store data for the Website while, logs-lv will be used to store data for logs.
`sudo lvcreate -n apps-lv -L 9G webdata-vg`
`sudo lvcreate -n logs-lv -L 9G webdata-vg`
`sudo lvcreate -n opt-lv -L 9G webdata-vg`
![](images/15.PNG)

Verify that your Logical Volume has been created successfully by running `sudo lvs`
![](images/16.PNG)

Verify the entire setup
`sudo vgdisplay -v #view complete setup - VG, PV, and LV`
`sudo lsblk`
![](images/17.PNG)

Use mkfs.xfs to format the logical volumes with xfs filesystem
`sudo mkfs -t xfs /dev/webdata-vg/lv-apps`
`sudo mkfs -t xfs /dev/webdata-vg/lv-logs`
`sudo mkfs -t xfs /dev/webdata-vg/lv-opt`
![](images/21.PNG)

Create mnt directory for the logical volumes as follow:
`sudo mkdir /mnt/apps`
`sudo mkdir /mnt/logs`
`sudo mkdir /mnt/opt`
![](images/19.PNG)

Mount points on mnt directory for the logical volumes as follow:
`sudo mount /dev/webdata-vg/apps-lv /mnt/apps`
`sudo mount /dev/webdata-vg/logs-lv /mnt/logs`
`sudo mount /dev/webdata-vg/opt-lv /mnt/opt`
![](images/22.PNG)

Install NFS server, configure it to start on reboot and make sure it is up and running
`sudo yum -y update`
`sudo yum install nfs-utils -y`
`sudo systemctl start nfs-server.service`
`sudo systemctl enable nfs-server.service`
`sudo systemctl status nfs-server.service`
![](images/23.PNG)

Export the mounts for webservers’ subnet cidr to connect as clients. To check for subnet cidr – open EC2 details in AWS web console and locate ‘Networking’ tab and open a Subnet link:
![](images/24.PNG)

Set up permission that will allow our Web servers to read, write and execute files on NFS:
`sudo chown -R nobody: /mnt/apps`
`sudo chown -R nobody: /mnt/logs`
`sudo chown -R nobody: /mnt/opt`
![](images/25.PNG)

`sudo chmod -R 777 /mnt/apps`
`sudo chmod -R 777 /mnt/logs`
`sudo chmod -R 777 /mnt/opt`
![](images/26.PNG)

Restart the NFS server
`sudo systemctl restart nfs-server.service`

Check the NFS server
`sudo systemctl status nfs-server.service`
![](images/27.PNG)

Configure access to NFS for clients within the same subnet
`sudo vi /etc/exports`
Then paste the following 
`/mnt/apps 172.31.80.0/20(rw,sync,no_all_squash,no_root_squash)`
`/mnt/logs 172.31.80.0/20(rw,sync,no_all_squash,no_root_squash)`
`/mnt/opt 172.31.80.0/20(rw,sync,no_all_squash,no_root_squash)`
![](images/28.PNG)

To see the exportfs
`sudo exportfs -arv`
![](images/29.PNG)

Check which port is used by NFS and open it using Security Groups
`rpcinfo -p | grep nfs`
![](images/30.PNG)

## Step 2: Configure the database server

Launch an instance for database server using ubuntu
![](images/31.PNG)

ssh to database server using Windows shell
![](images/32.PNG)

Install MySQL server
`sudo apt update`
`sudo apt install mysql-server -y`
Check status
`sudo systemctl status mysql`
![](images/33.PNG)

Create a database and name it tooling
`sudo mysql`

`create database tooling;`

`Create a database user and name it webaccess`

 Grant permission to webaccess user on tooling database to do anything only from the webservers subnet cidr

`grant all privileges on tooling.* to 'webaccess'@'172.31.80.0/20';`

`flush privileges;`

`show databases;`

`use tooling;`

`ext;`
![](images/34.PNG)

Check mysql status
`sudo systemctl status mysql`
![](images/35.PNG)

Change the bind address
`sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf`

bind-address   = 0.0.0.0

mysqlx-bind-address   = 0.0.0.0
![](images/36.PNG)

Restart mysql
`sudo systemctl restart mysql`
Check staus
`sudo systemctl status mysql`
![](images/37.PNG)





## Step 3: Prepare the webservers
1. Launch an instance for Web server
![](images/39.PNG)

ssh to web server using Windows shell
![](images/38.PNG)

2. Install NFS client
`sudo yum install nfs-utils nfs4-acl-tools -y`
![](images/40.PNG)

3. Mount /var/www/ and target the NFS server’s export for apps
`sudo mkdir /var/www`
`sudo mount -t nfs -o rw,nosuid`
`172.31.85.207:/mnt/apps /var/www`

4. Verify that NFS was mounted successfully by running `df -h`. 
![](images/41.PNG)

Make sure that the changes will persist on Web Server after reboot:
`sudo vi /etc/fstab`

add following line
172.31.85.207:/mnt/apps /var/www nfs defaults 0 0

5. Install Remi’s repository, Apache and PHP
`sudo yum install httpd -y`

`sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm`

`sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm`

`sudo dnf module reset php`

`sudo dnf module enable php:remi-7.4`

`sudo dnf install php php-opcache php-gd php-curl php-mysqlnd`

`sudo systemctl start php-fpm`

`sudo systemctl enable php-fpm`

`sudo setsebool -P httpd_execmem 1`
![](images/42.PNG)

Repeat steps 1-5 for the other 2 Web Servers.

6. Verify that Apache files and directories are available on the Web Server in /var/www and also on the NFS server in /mnt/apps. 
`ls /var/www`
![](images/43.PNG)

7. Locate the log folder for Apache on the Web Server and mount it to NFS server’s export for logs
![](images/44.PNG)

`sudo mount -t nfs -o rw,nosuid 172.31.85.207:/mnt/logs /var/log/httpd`

Locate the log folder for Apache on the Web Server and mount it to NFS server’s export for logs. Repeat step №4 to make sure the mount point will persist after reboot.
`sudo vi /etc/fstab`
add following line

`172.31.85.207:/mnt/apps /var/www nfs defaults 0 0`
`172.31.85.207:/mnt/logs /var/log/httpd nfs defaults 0 0`
![](images/45.PNG)

8. Fork the tooling source code from Darey.io Github Account to your Github account
Install git on the web server:
`sudo yum install git`

click on code and make sure it's on https and copy the link
git clone (the link from github)
![](images/46.PNG)
![](images/47.PNG)

ls (to see the tooling file)
cd tooling
ls to see all files under tooling 
![](images/48.PNG)

Deploy the tooling website’s code to the Webserver.
check for the html file by running this command ls /var/www
`sudo cp -R html/. /var/www/html`
`ls /var/www/html`
`ls html`

Open port http 80 on the web server and allow to anywhere
![](images/49.PNG)

The web server ip address is not rachable so I will check the permission of /var/www/html folder and also disable SELinux sudo setenforce 0

To make this change permanent – I opened following config file sudo vi /etc/sysconfig/selinux and set SELINUX=disabledthen

![](images/50.PNG)

Restart the apache httpd 
`sudo systemctl restart httpd`
`sudo systemctl status httpd`
![](images/51.PNG)

Update the website’s configuration to connect to the database. 
`sudo vi  /var/www/html/functions.php file`
change the following under connect to database 
$db = mysqli_connect('172.31.81.144', 'webaccess', 'password', 'tooling');
![](images/52.PNG)

Add inbound rule for database server
Mysql/aurora
![](images/54.PNG)

Apply tooling-db.sql script to database using this command
Install mysql using `sudo yum install mysql` then run the following command
`mysql -h 172.31.81.144 -u webaccess -p tooling < tooling-db.sql`
![](images/55.PNG)

Create in MySQL a new admin user with username: myuser and password: password:

![](images/56.PNG)

Open the website in your browser http://54.197.70.183/index.php and make sure you can login into the websute with myuser user.
![](images/57.PNG)

![](images/58.PNG)














