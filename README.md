# Private_Registry_on_Ubuntu
This project contains README document only with the steps I used to Create Private Registry Server

## Create 2 AWS EC2 instances (I created UBUNTU AMIs)
   Registry Server
   My Web Server

## check ssh connection works correctly

   ssh -i ~/.ssh/keys/aws_ec2_gitlab_key.pem ubuntu@3.85.112.86 - my web server

   ssh -i ~/.ssh/keys/aws_ec2_gitlab_key.pem ubuntu@3.86.145.0  - registry server

   

## Ensure that nslookup works correctly on both the servers. (THIS IS THE KEY STEP BASICALLY servers should able to recognize each other IPs)
   e.g. 
   ssh -i ~/.ssh/keys/aws_ec2_gitlab_key.pem ubuntu@3.85.112.86
   from my webserver execute
   nslookup ip-172-31-94-110  hostname of registry server
   
   ubuntu@ip-172-31-16-60:~$ nslookup ip-172-31-94-110
   Server:         127.0.0.53
   Address:        127.0.0.53#53

   Non-authoritative answer:
   Name:   ip-172-31-94-110.ec2.internal
   Address: 172.31.94.110




   similarly
   ssh -i ~/.ssh/keys/aws_ec2_gitlab_key.pem ubuntu@3.86.145.0  - registry server
   from registry server execute
   nslookup ip-172-31-16-60 hostname of my web server

   ubuntu@ip-172-31-94-110:~$ nslookup ip-172-31-16-60
   Server:         127.0.0.53
   Address:        127.0.0.53#53

   Non-authoritative answer:
   Name:   ip-172-31-16-60.ec2.internal
   Address: 172.31.16.60


## install docker on each server
install docker on ubuntu
https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository


sudo apt install docker-compose

https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user

## Now login to the registry server


   ssh -i ~/.ssh/keys/aws_ec2_gitlab_key.pem ubuntu@3.86.145.0
   
   and run following commands
   
   (a) mkdir -p ~/registry/auth
   
   (b) docker run --entrypoint htpasswd \
         registry:2.7.0 -Bbn docker password > ~/registry/auth/htpasswd

   
   (c) mkdir -p ~/registry/certs


   (d) echo $HOSTNAME  ---> ip-172-31-94-110

   (e) 
   ~~~~sh
    openssl req \
  -newkey rsa:4096 -nodes -sha256 -keyout ~/registry/certs/domain.key \
  -x509 -days 365 -out ~/registry/certs/domain.crt \
  -addext "subjectAltName = DNS:ip-172-31-94-110"
   ~~~~

   this command will ask
   Common Name (e.g. server FQDN or YOUR name) []:ip-172-31-94-110
   supply the host name here which you received in step (d)
  
   (f) 
   ~~~~sh
   docker run -d \
  -p 443:443 \
  --restart=always \
  --name registry \
  -v /home/ubuntu/registry/certs:/certs \
  -v /home/ubuntu/registry/auth:/auth \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  registry:2.7.0
  ~~~~

   verify that a container is up with name registry ... check inside the container and verify in /certs and /auth directories if password and certificates are 
   created correctly


   (g) curl -k https://localhost:443 -> verify this should not give you any error



## Now login to my webserver
   ssh -i ~/.ssh/keys/aws_ec2_gitlab_key.pem ubuntu@3.85.112.86

   the hostname found above of the registry server was -> ip-172-31-94-110
   based on this information run the following commands on web server

   (a) sudo mkdir -p /etc/docker/certs.d/ip-172-31-94-110:443

   (b) copy the ssh key on 'my webserver' location could be ~/.ssh/keys/aws_ec2_gitlab_key.pem
       sudo chmod 400 aws_ec2_gitlab_key.pem

   (c) sudo scp -i ~/.ssh/keys/aws_ec2_gitlab_key.pem ubuntu@3.86.145.0:/home/ubuntu/registry/certs/domain.crt /etc/docker/certs.d/ip-172-31-94-110:443

   (d) finally verify
       docker login ip-172-31-94-110:443
      
       docker pull ubuntu
       docker tag ubuntu ip-172-31-94-110:443/test-image:1
       docker push ip-172-31-94-110:443/test-image:1

       now delete the image locally
       and pull it again from registry
       docker image rm ip-172-31-94-110:443/test-image:1
       docker image rm ubuntu:latest
       docker pull ip-172-31-94-110:443/test-image:1
   


## REFERENCES:
- https://docs.docker.com/registry/deploying/
- https://docs.docker.com/registry/deploying/#restricting-access
- https://docs.docker.com/registry/insecure/#use-self-signed-certificates


- https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository
- https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user

