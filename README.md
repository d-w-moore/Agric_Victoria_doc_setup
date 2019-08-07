
# S3 setup for Agriculture Victoria iRODS demo

## OS changes

   - edit `/etc/selinux/config`
     replace `enforcing` with `disabled` and reboot
   - configure passwordless `sudo`
     `loginName ALL=(ALL) NOPASSWD: ALL` as last
     line of `sudoers`
   - `sudo yum install -y epel-release wget git vim nano`
   - disable IPTABLES and any firewalls
     * `sudo yum remove -y firewalld iptables iptables-services`
     * `iptables -F`

## Install and start up PostgreSQL

   - `sudo yum install -y postgresql-server`
   - `sudo su - postgres -c '/bin/pg_ctl initdb'`
   - `systemctl enable postgresql ; systemctl start postgresql`

## Install iRODS server and development pkgs
   - setup ICAT: identical to  Ubuntu/PostgreSQL, follow instruction at :
     `https://docs.irods.org/4.2.6/getting_started/installation/#database-setup`
   - install iRODS packages

      * Add repository:
         1. `sudo rpm --import https://packages.irods.org/irods-signing-key.asc`

         2. `wget -qO - https://packages.irods.org/renci-irods.yum.repo | sudo tee /etc/yum.repos.d/renci-irods.yum.repo`

   - `sudo yum install -y irods-server irods-devel irods-database-plugin-postgres`

   - `sudo python /var/lib/irods/scripts/setup_irods.py </var/lib/irods/packaging/localhost_setup_postgres.input`

## Install iRODS externals and other build pre-req's; and Compile/Install S3 plugin

   - `sudo yum install -y irods-externals\* gcc openssl-devel curl-devel libxml2-devel rpm-build`

   - `mkdir ~/github ; cd ~/github ; git clone https://github.com/irods/irods_resource_plugin_s3`

   - `cd irods_resource_plugin_s3 &&git checkout 4-2-stable ; /opt/irods-externals/cmake3.11.4-0/bin/cmake ../irods*s3`

   - `make package ; sudo make install`

## As the iRODS admin, make the resource

   - `sudo su - irods`

   - Put S3 credentials into `/var/lib/irods/s3.keypair`

   - ```
   iadmin mkresc s3resc s3  $(hostname):/avr-irods-data "S3_DEFAULT_HOSTNAME=s3.ap-southeast-2.amazonaws.com;S3_AUTH_FILE=/var/lib/irods/s3.keypair;S3_REGIONNAME=ap-southeast-2;S3_RETRY_COUNT=2;S3_WAIT_TIME_SEC=3;S3_PROTO=HTTP;HOST_MODE=cacheless_attached;S3_SIGNATURE_VERSION=4;S3_ENABLE_MPU=1;S3_MPU_THREADS=30;S3_MPU_CHUNK=256"
   ```

   - Register S3 contents with: `ireg -R s3resc -C /avr-irods-data/"Example Data" /tempZone/home/rods/"Example Data"`

   - Unregister S3 contents with (in this case): `irm -Ur /tempZone/home/rods/"Example Data"`
   
