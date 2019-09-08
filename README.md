
# iRODS and S3 setup for Agriculture Victoria iRODS demo

- Centos 7 based setup
- if we're on a VM, need >= 15 GB on root partn to start. will move database to a different
  partition, later, if needed to support large amounts of metadata.
- allot 4 CPUs (ie 8 cores, or threads, as shown by `htop` display)

## Linux / CentOS-7 related 

   - edit `/etc/selinux/config`
     replace `enforcing` with `disabled` and reboot
     (on next reboot `sestatus` should show that SELinux is **disabled**)
   - configure passwordless `sudo`
     `loginName ALL=(ALL) NOPASSWD: ALL` as last
     line of `sudoers`
   - `sudo yum install -y epel-release wget git vim nano tmux htop`
   - disable firewall
     `sudo systemctl stop firewalld ; sudo systemctl disable firewalld`

Install Docker CE.  Source [here](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-centos-7) for full instructions.
This should be done as the sudo-enabled user (necessary for hosting Metalnx)
    ```
    $ sudo yum check-update
 
    curl -fsSL https://get.docker.com/ | sh
 
    sudo systemctl start docker

    `sudo usermod -aG docker $(whoami)`
    and optionally: `sudo usermod -aG docker irods`
    
   - log out, then back in (to refresh shell environment) - or reboot if necessary
    
    $ sudo systemctl status docker
    $ docker pull alpine
    $ docker run -it --rm alpine echo hello world
    ```
If docker is running properly, enable for auto start on reboot:
```
sudo systemctl enable docker
```

## Install and start up PostgreSQL

   - `sudo yum install -y postgresql-server`
   - `sudo su - postgres -c '/bin/pg_ctl initdb'`
   - `sudo systemctl enable postgresql ; sudo systemctl start postgresql`

## Install iRODS server and development pkgs
   - setup ICAT: same `psql` commands as with Ubuntu/PostgreSQL setup, so follow instruction at :
     `https://docs.irods.org/4.2.6/getting_started/installation/#database-setup`

   - install iRODS packages

      * Add repository:
         1. `sudo rpm --import https://packages.irods.org/irods-signing-key.asc`

         2. `wget -qO - https://packages.irods.org/renci-irods.yum.repo | sudo tee /etc/yum.repos.d/renci-irods.yum.repo`

   - `sudo yum install -y irods-server irods-devel irods-database-plugin-postgres`

   - `sudo python /var/lib/irods/scripts/setup_irods.py </var/lib/irods/packaging/localhost_setup_postgres.input`

## Install iRODS  cacheless S3 resource plugin

   - `sudo yum install -y irods-externals\* gcc openssl-devel curl-devel libxml2-devel`
   - (this will take a few minutes so get coffee)
   - `mkdir ~/github ; cd ~/github ; git clone https://github.com/irods/irods_resource_plugin_s3`

   - `cd irods_resource_plugin_s3 && git checkout 4-2-stable ; mkdir ../obj ; cd ../obj ; /opt/irods-externals/cmake3.11.4-0/bin/cmake ../irods*s3`

   - `make -j3 && sudo install libs3.so /usr/lib/irods/plugins/resources`

## As the iRODS admin, make the resource

   - `sudo su - irods`

   - Put S3 credentials into `/var/lib/irods/s3.keypair`

   - ```
     iadmin mkresc s3resc s3  $(hostname):/avr-irods-data "S3_DEFAULT_HOSTNAME=s3.ap-southeast-2.amazonaws.com;S3_AUTH_FILE=/var/lib/irods/s3.keypair;S3_REGIONNAME=ap-southeast-2;S3_RETRY_COUNT=2;S3_WAIT_TIME_SEC=3;S3_PROTO=HTTP;HOST_MODE=cacheless_attached;S3_SIGNATURE_VERSION=4;S3_ENABLE_MPU=1;S3_MPU_THREADS=30;S3_MPU_CHUNK=256"
     ```
   - Register S3 contents (note - serially, without parallelism, an initial registration type ingest can take hours) with: 

     `ireg -R s3resc -C /avr-irods-data/"Example Data" /tempZone/home/rods/"Example Data"`

   - When/if necessary, unregister S3 contents with (in this case): 

     `irm -Ur /tempZone/home/rods/"Example Data"`
     (Do not use `irm -fr ... ` as this could delete bucket contents.)   

### install s3fs and mounting an s3 bucket as filesystem

The fuse layer doesn't provide a seek operation other than by reading, in Proof Of Concept we can still use an s3fs  mounted bucket to allow arbitrary POSIX-like accesses to file objects in the bucket without necessitating a full download.  This is beneficial for cases in which a extracting relevant metadata from the "file" involves header-only (with onlyl a minority of the file being read) or modification of the tool to use iRODS read API calls is impossible or cumbersome . The setup:
   
   - `sudo yum install -y s3fs-fuse` (as admin enabled user)
   
   - setup example (as iRODS) : (having placed <Access_Key>:<secret_key> into `~irods/.passwd-s3fs`
     issue this command, as `irods` user:
     ```
     cd ; mkdir irods_s3 ; s3fs avr-irods-data `pwd`/irods_s3 -o passwd_file=$HOME/.passwd-s3fs,umask=033,allow_other,endpoint="ap-southeast-2",url="https://s3.ap-southeast-2.amazonaws.com"
     ```
     The command `fusermount -u irods_s3` will unmount the filesystem.
     
   - Appending the following line in the `/etc/fstab` will cause the bucket to be mounted  automatically at boot time.
   ```
   s3fs#avr-irods-data /mnt/avr-irods-data fuse ro,allow_other,umask=022,passwd_file=/etc/avr-irods-data_passwd-s3fs,endpoint=ap-southeast-2,url=https://s3.ap-southeast-2.amazonaws.com
   ```

## Setting up for greater efficiency and economy in S3 file ingest

We registering from S3 while extracting metadata extraction from large images' EXIF headers

Large images (ie TIFs) can include key-value metadata in EXIF headers, and these can be distributed throughout the body of the image file intermixed with data segments.  For extraction via the post-register hook we can access these very efficiently/economically using the python-irodsclient's data_object read method with the iRODS automated ingest tool.  see next section.

## install iRODS-capability-automated-ingest
   - `sudo yum install -y python2-pip python-virtualenv python36`
   - as irods: `cd ~irods ; pip install irods_capability_automated_ingest`

## MetaLnx
   - `docker pull irods/metalnx`
   - `git clone irods-contrib/metalnx-web`; copy the etc/irods-ext
   ```
   docker run -d --add-host hostcomputer:172.17.0.1 -p 8080:8080 --rm -it -v `pwd`/irods-ext:/etc/irods-ext:ro  irods/metalnx
   ```
   
## WebDAV
