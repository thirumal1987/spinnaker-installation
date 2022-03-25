# spinnaker-installation with github

1) Create new VM with OS ubuntu 12gb of RAM.
3) create a clone repository from github with below command 
4) # git clone https://github.com/thirumal1987/spinnaker-installation.git and switch to ubuntu user  then follow the below steps.
5) execution scripts will located in below path spinnekar-data --->scripts
6) 1-create-user.sh

 #!/bin/bash

groupadd ubuntu
useradd -g ubuntu -G admin -s /bin/bash -d /home/ubuntu ubuntu
mkdir -p /home/ubuntu
cp -r /root/.ssh /home/ubuntu/.ssh
chown -R ubuntu:ubuntu /home/ubuntu
echo "ubuntu	ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers

8) 2-swapon.sh

#!/bin/bash

sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
sudo swapon /swapfile

10) 3-install-halyard.sh 

#!/bin/bash

set -e

sudo add-apt-repository ppa:openjdk-r/ppa -y

sudo apt-get update
sudo apt-get -y install jq openjdk-11-jdk

curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh
sudo bash InstallHalyard.sh --user ubuntu
curl -fsSL get.docker.com -o get-docker.sh
sh get-docker.sh
sudo usermod -aG docker ubuntu
sudo docker run -p 127.0.0.1:9090:9000 -d --name minio1 -v /mnt/data:/data -v /mnt/config:/root/.minio minio/minio server /data

sudo apt-get -y install jq apt-transport-https

MINIO_SECRET_KEY="minioadmin"
MINIO_ACCESS_KEY="minioadmin"

echo $MINIO_SECRET_KEY | hal config storage s3 edit --endpoint http://127.0.0.1:9090 \
    --access-key-id $MINIO_ACCESS_KEY \
    --secret-access-key

hal config storage edit --type s3

12) 4-configure-oath.sh  ---> we have create client ID and screct key from github settings--> Developers settings-->Oauth access here we have to give VM IP  http:// 3.84.156.82:8084/login then it will create client ID and secret key and update like below .

#!/bin/bash

# env flags that need to be set:
CLIENT_ID=5a36ca610f0a8f5c54a1
CLIENT_SECRET=87e38e2783a944a3640650db7026b6ac6493022b
PROVIDER=github
REDIRECT_URI=http://ip:8084/login

set -e

if [ -z "${CLIENT_ID}" ] ; then
  echo "CLIENT_ID not set"
  exit
fi
if [ -z "${CLIENT_SECRET}" ] ; then
  echo "CLIENT_SECRET not set"
  exit
fi
if [ -z "${PROVIDER}" ] ; then
  echo "PROVIDER not set"
  exit
fi
if [ -z "${REDIRECT_URI}" ] ; then
  echo "REDIRECT_URI not set"
  exit
fi

MY_IP=`curl -s ifconfig.co`

hal config security authn oauth2 edit \
  --client-id $CLIENT_ID \
  --client-secret $CLIENT_SECRET \
  --provider $PROVIDER
hal config security authn oauth2 enable

hal config security authn oauth2 edit --pre-established-redirect-uri $REDIRECT_URI

hal config security ui edit \
    --override-base-url http://${MY_IP}:9000

hal config security api edit \
    --override-base-url http://${MY_IP}:8084
    
14) 5-deploy-spinnekar.sh

#!/bin/bash

# install dependencies
sudo apt update
sudo apt-get -y install redis-server
sudo systemctl enable redis-server
sudo systemctl start redis-server

echo 'spinnaker.s3:
  versioning: false
' > ~/.hal/default/profiles/front50-local.yml

# env flag that need to be set:
SPINNAKER_VERSION=1.21.5

set -e

if [ -z "${SPINNAKER_VERSION}" ] ; then
  echo "SPINNAKER_VERSION not set"
  exit
fi

sudo hal config version edit --version $SPINNAKER_VERSION

sudo hal deploy apply

16) restart-spinnekar.sh

#!/bin/bash

sudo systemctl restart apache2
sudo systemctl restart gate
sudo systemctl restart orca
sudo systemctl restart igor
sudo systemctl restart front50
sudo systemctl restart echo
sudo systemctl restart clouddriver
sudo systemctl restart rosco
