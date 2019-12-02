
# iRODS and S3 setup for Agr. Vict. & iRODS 

- Centos 7 based setup
- if we're on a VM, need >= 15 GB on root partn to start. will move database to a different
  partition, later, if needed to support large amounts of metadata.
- allot 4 CPUs (ie 8 cores, or threads, as shown by `htop` display)

## Linux / CentOS-7 related 

   - edit `/etc/selinux/config`
     replace `enforcing` with `disabled` and reboot
     (on next reboot `sestatus` should show that SELinux is disabled)
   - configure passwordless `sudo`:
      * `sudo visudo` and enter login password when challenged (for the last time ever!)
      * `loginName ALL=(ALL) NOPASSWD: ALL` as last
         line of `sudoers`
   - `sudo yum install -y epel-release wget git vim nano tmux htop`
   - Either take steps to expose needed ports (1247 for irods, 80 & 8080 for web, etc) or disable firewalld

Install Docker CE.  Source [here](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-centos-7) for full instructions. Docker CE will be necessary for hosting Metalnx.
This should be done as the sudo-enabled user: 

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
   - (this will take a few minutes, so go and get coffee...)
   - `mkdir ~/github ; cd ~/github ; git clone https://github.com/irods/irods_resource_plugin_s3`

   - `cd irods_resource_plugin_s3 && git checkout 4-2-stable ; mkdir ../obj ; cd ../obj ; /opt/irods-externals/cmake3.11.4-0/bin/cmake ../irods*s3`

   - `make -j3 && sudo install libs3.so /usr/lib/irods/plugins/resources`

## As the iRODS admin, make the resource

   - `sudo su - irods`

   - Put S3 credentials into `/var/lib/irods/s3.keypair`

   - ```
     iadmin mkresc s3resc s3  $(hostname):/avr-irods-data "S3_DEFAULT_HOSTNAME=s3.ap-southeast-2.amazonaws.com;S3_AUTH_FILE=/var/lib/irods/s3.keypair;S3_REGIONNAME=ap-southeast-2;S3_RETRY_COUNT=2;S3_WAIT_TIME_SEC=3;S3_PROTO=HTTP;HOST_MODE=cacheless_attached;S3_SIGNATURE_VERSION=4;S3_ENABLE_MPU=1;S3_MPU_THREADS=30;S3_MPU_CHUNK=256"
     ```
   - If we were registering the s3 files the traditional iRODS way, aka the SLOW approach:
   
      * One would register S3 contents (note - serially ie. without parallelism, an initial registration type ingest can take hours) with: 

        `ireg -R s3resc -C /avr-irods-data/"Example Data" /tempZone/home/rods/"Example Data"`

      * When/if necessary, you would unregister S3 contents with (in this case): 

        `irm -Ur /tempZone/home/rods/"Example Data"`
        (Do not use `irm -fr ... ` as this could delete bucket contents.)  

   - instead, we'll do the registration with the automated ingest tool (we'll come to this soon!)

### install s3fs and mounting an s3 bucket as filesystem

The fuse layer doesn't provide a seek operation other than by reading, in Proof Of Concept we can still use an s3fs  mounted bucket to allow arbitrary POSIX-like accesses to file objects in the bucket without necessitating a full download.  This is beneficial for cases in which a extracting relevant metadata from the "file" involves header-only (with onlyl a minority of the file being read) or modification of the tool to use iRODS read API calls is impossible or cumbersome . The setup:
   
   - `sudo yum install -y s3fs-fuse` (as admin enabled user)
   
   - setup example (as iRODS) : (having placed <Access_Key>:<secret_key> into `~irods/.passwd-s3fs`
     issue this command, as `irods` user:
     ```
     cd ; mkdir irods_s3 ; s3fs avr-irods-data `pwd`/irods_s3 -o passwd_file=$HOME/.passwd-s3fs,umask=022,allow_other,endpoint="ap-southeast-2",url="https://s3.ap-southeast-2.amazonaws.com"
     ```
   - Additionally, `user_allow_other` needs to be set in `/etc/fuse.conf`
   - The command `fusermount -u irods_s3` will unmount the filesystem.
     
   - Appending the following line in the `/etc/fstab` will cause the bucket to be mounted  automatically at boot time.
   
   ```
   s3fs#avr-irods-data /mnt/avr-irods-data fuse ro,allow_other,umask=022,passwd_file=/etc/avr-irods-data_passwd-s3fs,endpoint=ap-southeast-2,url=https://s3.ap-southeast-2.amazonaws.com
   ```

## Setting up for better efficiency and economy in S3 file ingest

We'll be registering from S3 while extracting metadata extraction from large images' EXIF headers

Large images (ie TIFs) can include key-value metadata in EXIF headers, and these can be distributed throughout the body of the image file intermixed with data segments.  For extraction via the post-register hook we can access these very efficiently/economically using the python-irodsclient's data_object read method with the iRODS automated ingest tool.  

## install iRODS automated-ingest tool and register S3 files and metadata

- Here, we create the entries in iRODS's ICAT (object catalog) that reflect what is in the S3 bucket.
   - We also extract metadata from TIF files (add'l metadata options will exist later.)
   - We can take advantage of client's access to S3 bucket to register S3 objects in place, extract metadata and attach it to the objects in the iRODS catalog, and take advantage of multiple cores ie use the CPU parallelism available
   - `sudo yum install -y python2-pip python-virtualenv python36`
   - as irods: 
   ```
   $ cd ~irods 
   $ virtualenv -p python3 rodssync
   $ source rodssync/bin/activate
   $ pip install irods_capability_automated_ingest
   $ pip install exifread 
   ```

## Performing the initial ingest

   # in separate terminals (or tmux windows)
   As service account user `irods`, with the rodssync virtual environment enabled (ie do `source ~irods/rodssync/bin/activate` under bash shell):
   
   ```             
       - start celery workers:
         $ celery -A irods_capability_automated_ingest.sync_task worker -l error -Q restart,path,file
         
       - do the ingest:
         $ python -m irods_capability_automated_ingest.irods_sync start "/avr-irods-data/Example Data/" "/tempZone/home/rods/Example Data" --s3_keypair /var/lib/irods/s3.keypair --s3_endpoint_domain s3.ap-southeast-2.amazonaws.com --s3_region_name ap-southeast-2 --event_handler tiff_event_handler --synchronous --initial_ingest
```

## MetaLnx

To deploy Metalnx  from a docker container, as a service available on the web at port 8080:
   - `docker pull irods/metalnx`
   - `git clone irods-contrib/metalnx-web`; cd to just above `irods-ext` and `cp -r irods-ext $HOME/.`
   - modify `~/irods-ext/metalnx.properties` irods.* and db.*  settings and configure `ssl.negotiation.policy=CS_NEG_REFUSE`
   - also change  the `acPreConnect()` rule in `/etc/irods/core.re` [or `.py`] file such that `CS_NEG_REFUSE` is the value returned
   - in the `pg_hba.conf` add the line:
   ```
   host    all             all             172.17.0.1/16           md5
   ```
   (where the `172.17.0.1/16` mask is the bridge network provided by the docker network)
   - and in `postgresql.conf` add the line:
   ```
   listen_addresses = '*'
   ```
   - Start up the Metalnx application
   ```
   docker run -d --add-host hostcomputer:172.17.0.1 -p 8080:8080 --rm -it -v $HOME/irods-ext:/etc/irods-ext:ro  irods/metalnx
   ```
   - access port 8080 on the host's IP interface: `http://yourhostname.org:8080/metalnx`
   
## WebDAV
   - install package httpd : `sudo yum install -y httpd`
   
   - Follow instructions at https://github.com/UtrechtUniversity/davrods
   
   - in `/etc/httpd/irods/irods_environment.json` configure CS_NEG_REFUSE and irods zone, host and port params
     (can use `localhost` to refer to iRODS server host, if httpd and iRODS are resident on same server)
   - in virtual host ServerName declaration, give valid DNS hostname eg `dav.example.com` and make sure this
     hostname is matched by host or network DNS configuration (eg. put entry for `dav.example.com` in etc/hosts)
   - if another port is desired, say port 8000:
      * add `Listen 8000` to `/etc/httpd/httpd.conf`
      * configure VirtualHost declaration with \*:8000
   - `systemctl restart httpd ; systemctl  enable httpd`
   
