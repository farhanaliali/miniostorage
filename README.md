# Minio distributed setup
##  Author: Farhan ali


You will need to build out your 5 base servers. 4 of them will need the extra 4 disks each
I'm assuming all servers are in DNS as the following:

    minio1.domain.com
    minio2.domain.com
    minio3.domain.com
    minio4.domain.com
    minio.domain.com (This is the load balance)
All 4 servers should have /dev/sdb,sdc,sdd,sde as the 4 empty disks.
Obviously, you can change as you see fit but will have to adapt this guide.

## Step 1: Initial Set Up

On each Minio server set up the mount locations on each server:

    sudo mkdir /mnt/minio1
    sudo mkdir /mnt/minio2
    sudo mkdir /mnt/minio3
    sudo mkdir /mnt/minio4

Create a partition on each disk:

    sudo fdisk /dev/sdb
    sudo fdisk /dev/sdc
    sudo fdisk /dev/sdd
    sudo fdisk /dev/sde

It will prompt for the following sequence. You can accept the defaults mostly. It's n for the new partition, enter for the default partition number, enter for the first sector, enter for the last sector, and w to write the partition to disk.
    
    Repeat that for all disks.

Add a file system. Specifically xfs!

    mkfs.xfs /dev/sdb1
    mkfs.xfs /dev/sdc1
    mkfs.xfs /dev/sdd1
    mkfs.xfs /dev/sde1

Repeat that for all disks.

Now mount the disks on all servers accordingly 

    sudo vi /etc/fstab

    /dev/sdb    /mnt/minio1 xfs defaults 0 1 
    /dev/sdc    /mnt/minio2 xfs defaults 0 1
    /dev/sdd    /mnt/minio3 xfs defaults 0 1
    /dev/sde    /mnt/minio4 xfs defaults 0 1
save and mount the disks

    sudo mount -a 

## Install minio 

Use the following commands to download the latest stable MinIO DEB and install it:

    wget https://dl.min.io/server/minio/release/linux-amd64/archive/minio_20230904195737.0.0_amd64.deb -O minio.deb
    sudo dpkg -i minio.deb

We need to create a default config file:

    
    sudo vi /etc/default/minio
This is the file contents for me:

    #Set the hosts and volumes MinIO uses at startup
    # The command uses MinIO expansion notation {x...y} to denote a
    # sequential series.
    #
    # The following example covers four MinIO hosts
    # with 4 drives each at the specified hostname and drive locations.
    # The command includes the port that each MinIO server listens on
    # (default 9000)

    MINIO_VOLUMES="https://minio{1...4}.domain.com:9000/mnt/minio{1...4}"

    # Set all MinIO server options
    #
    # The following explicitly sets the MinIO Console listen address to
    # port 9001 on all network interfaces. The default behavior is dynamic
    # port selection.

    MINIO_OPTS="--console-address :9001"

    # Set the root username. This user has unrestricted permissions to
    # perform S3 and administrative API operations on any resource in the
    # deployment.
    #
    # Defer to your organizations requirements for superadmin user name.

    MINIO_ROOT_USER=minioadmin

    # Set the root password
    #
    # Use a long, random, unique string that meets your organizations
    # requirements for passwords.

    MINIO_ROOT_PASSWORD=SuperSecretPassword

    # Set to the URL of the load balancer for the MinIO deployment
    # This value *must* match across all MinIO servers. If you do
    # not have a load balancer, set this value to to any *one* of the
    # MinIO hosts in the deployment as a temporary measure.
    MINIO_SERVER_URL="https://minio.domain.com:9000"

The key things to ensure are the volume names are correct, your admin username and password are as desired and your load balancer URL is set.

We need a mini user too and to give permissions to that user:

    sudo groupadd -r minio-user
    sudo useradd -M -r -g minio-user minio-user
    sudo chown minio-user:minio-user /mnt/minio1/ /mnt/minio2/ /mnt/minio3/ /mnt/minio4/

## SSL setup
We want to use SSL for all of the communication so let's use LetsEncrypt for free certs! Again edit the domain to reflect the node you are doing this on.

Now on mini1

    sudo apt install certbot
    sudo certbot certonly --standalone -d minio1.domain.com
    sudo mkdir -p /home/minio-user/.minio/certs
    cp /etc/letsencrypt/live/minio1.domain.com/fullchain.pem /home/minio-user/.minio/certs/public.crt
    cp /etc/letsencrypt/live/minio1.domain.com/privkey.pem /home/minio-user/.minio/certs/private.key
    chown -R minio-user:minio-user /home/minio-user/

We need Minio to bind to ports below 1024:

    setcap 'cap_net_bind_service=+ep' /usr/local/bin/minio

And we are finally ready to start Minio!
    
    sudo systemctl restart minio
    
Make sure it’s all healthy.

Now repeart the same configuration remining servers

On loadbalancer server 

Install nginx and certbot.

    sudo apt install nginx certbot
Create a config for Minio:

    sudo vi /etc/nginx/sites-available/minio

This is the file contents for me:

    upstream minio_servers {
            server minio1.domain.com:9000;
            server minio2.domain.com:9000;
            server minio3.domain.com:9000;
            server minio4.domain.com:9000;
            }
    upstream minio_console {
            ip_hash;
            server minio1.domain.com:9001;
            server minio2.domain.com:9001;
            server minio3.domain.com:9001;
            server minio4.domain.com:9001;
            }
    map $http_upgrade $connection_upgrade {
      default upgrade;
      '' close;
            }
    server {
            listen 443 ssl http2 default_server;
            listen [::]:443 ssl http2 default_server;
            server_name minio.domain.com;
            client_max_body_size 0;
            ssl_certificate "/etc/letsencrypt/live/minio.domain.com/fullchain.pem";
            ssl_certificate_key "/etc/letsencrypt/live/minio.domain.com/privkey.pem";

         location / {
              proxy_http_version 1.1;
              proxy_set_header Host $http_host;
              proxy_pass https://minio_console;
              proxy_set_header Upgrade $http_upgrade;
              proxy_set_header Connection $connection_upgrade;
               }
            }

    server {
              listen       9000 ssl;
              listen  [::]:9000;
              server_name minio.domain.com;
              ignore_invalid_headers off;
              client_max_body_size 0;
              proxy_buffering off;
              proxy_request_buffering off;
              ssl_certificate "/etc/letsencrypt/live/minio.domain.com/fullchain.pem";
              ssl_certificate_key "/etc/letsencrypt/live/minio.domain.com/privkey.pem";
    
        location / {
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_connect_timeout 300;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            chunked_transfer_encoding off;
            proxy_pass https://minio_servers;
        }
    }

Create the SSL cert from LetsEncrypt again:

    sudo systemctl stop nginx
    sudo certbot certonly --standalone --preferred-challenges http -d minio.domain.com

Verify the nginx config is all correct and no mistakes or missed characters!

    
Link the config to the sites-enabled:

    sudo ln -s /etc/nginx/sites-available/minio /etc/nginx/sites-enabled/minio
test the config

    sudo nginx -t
Lastly, start nginx.

    sudo service nginx start 

Assuming all of that went fine you now have a working Minio cluster. 4 nodes, 16 disks in total and all load balanced.

access the webui 

    htts://minio.domain.com

