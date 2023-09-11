# Create docker registry with nginx and minio storage 
## Author: Farhan ali


## package installation 

install the required packages on server 

    sudo apt install nginx docker.io docker-compose 


### create config for storage driver 

filename config.yml

    version: 0.1
    storage:
      s3:
        accesskey: ejMB93OlyPQsxXsdedzGOyka
        secretkey: lDRE5estwK30cSJPUWfVsdfsMeup7EJ4SFW0tpo2o5Lb
        region: us-east-1
        regionendpoint: https://minio.domain.com.com:9000
        bucket: docker-registry
        encrypt: false
    http:
        addr: :5000
        secret: SECRET_KEY
    
Modify the value according to your values

### Auth configuration

You can obtain the htpasswd utility by installing the apache2-utils package. Do so by running:

    sudo apt install apache2-utils -y
You’ll store the authentication file with credentials under auth. Create it by running:

    mkdir auth
Navigate to it:

    cd auth/
Create the first user, replacing username with the username you want to use. The -B flag orders the use of the bcrypt algorithm, which Docker requires:

    htpasswd -Bc registry.password rvtadmin

## setting up docker-compose


    version: '3'

    services:
      registry:
        image: registry:2
        ports:
        - "5000:5000"
        environment:
          - REGISTRY_AUTH=htpasswd
          - REGISTRY_AUTH_HTPASSWD_REALM=Registry
          - REGISTRY_AUTH_HTPASSWD_PATH=/auth/registry.password
        volumes:
          - ./auth:/auth
          - ./config.yml:/etc/docker/registry/config.yml

You can now start the configuration by running:

    docker-compose up -d 

## configure nginx 

You have set up the /etc/nginx/sites-available/your_domain 


    server {
            listen 443 ssl;
            listen [::]:443 ssl;
            server_name registry.domain.com;
              ssl_certificate "/etc/nginx/ssl/registry.domain.com/registry.domain.com.crt";
              ssl_certificate_key "/etc/nginx/ssl/registry.domain.com/registry.domain.com.key";

         location / {
            # Do not allow connections from docker 1.5 and earlier
            # docker pre-1.6.0 did not properly set the user agent on ping, catch "Go *" user agents
            if ($http_user_agent ~ "^(docker\/1\.(3|4|5(?!\.[0-9]-dev))|Go ).*$" ) {
                return 404;
            }

            proxy_pass                          http://localhost:5000;
            proxy_set_header  Host              $http_host;   # required for docker client's sake
            proxy_set_header  X-Real-IP         $remote_addr; # pass on real client's IP
            proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header  X-Forwarded-Proto $scheme;
            proxy_read_timeout                  900;           
        }

    }

Now create symbolic link in sites-enabled

    sudo ln -s /etc/nginx/sites-available/registry.domain.com /etc/nginx/sites-enabled/registry.domain.com
test nginx configuration
    
    sudo nginx -t
Restart the nginx service   
    
    sudo service nginx reload

## Publishing to Your Private Docker Registry

First, you have to log in:

    docker login https://your_domain
Tag image with your domain name

    docker tag test-image your_domain/test-image
Finally, push the newly tagged image to your registry:

    docker push your_domain/test-image

## Pulling From Your Private Docker Registry

On the main server, log in with the username and password you set up previously:

    docker login https://your_domain
Try pulling the test-image by running:

    docker pull your_domain/test-image

##########################################################################

