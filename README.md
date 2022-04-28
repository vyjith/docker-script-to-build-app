# docker-bash-script-to-build-app
docker-script-to-build-app
```

dockercompose_os(){
os=`hostnamectl | grep "Operating System" | awk '{print$3,$4,$5}'`
servername="Amazon Linux 2"

if [ "$os" = "$servername" ];

then
  echo ""
  docker_script

else
  echo "I'm sorry, we are unable to execute this script on this server. Please run this script on amazon linux 2"
  exit 1
fi
}

## step 3 :: checking the docker is installed or not on the server.....

dockercompose_requiredpackage(){

echo ""
echo "Checking the required docker packge is installed or not on your server. Please hold on..............................................................................."
echo ""
dockerstatus=`sudo systemctl show -p ActiveState docker.service | sed 's/ActiveState=//g'`
if [ "$dockerstatus" = active ];
	then	
	echo "The docker is already installed on teh server"
else 
    echo "The docker version is not installed on the server."
	echo "Installing docker on your server, please hold a moment"
	sudo yum install docker -y > /dev/null
	sudo systemctl start docker.service > /dev/null
	sudo systemctl enable docker.service > /dev/null
	echo "Docker installation has been completed now. "
	

fi
}


# Step 4 : Checking the docker-compose is installed or not

dockercomposeversion(){

docker-compose --version || error_code=$?;

if [[ "$error_code" -eq 0 ]];

        then
        echo "The docker compose version is already installed on the server"
        else
        echo "The docker-compose not installed on the server. Installing the docker-compose version. Please hold a moment........."
        sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose 
		sudo chmod +x /usr/local/bin/docker-compose
		sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
		echo "Docker installation hs been completed now"
fi
}

# Steps5 : Everything fine now, so that we are starting....

docker_script(){

dockercompose_requiredpackage
dockercomposeversion

mkdir /home/ec2-user/ip-api
mkdir /home/ec2-user/ip-api/nginx
echo ""
echo -n "Please let me know the domain name you would like to use for this : "
read domain

echo ""
echo -n "Please let me know the API key, you will get the API key by login here : https://app.ipgeolocation.io/auth/login : "
read API_KEY

echo -n "Please let me know the docker network, you would like to use : "
read mynet

docker network create $mynet

## Creating Nginx configration-------------------------------------------------- 
cat <<EOF > /home/ec2-user/ip-api/nginx/nginx.conf
upstream ipgeo {
        server ipgeolocation-frontend-service-1:8081;
        server ipgeolocation-frontend-service-1:8082;
        server ipgeolocation-frontend-service-1:8083;
}
server {
        listen 80;
        server_name $domain;
        location / {
            return 301 https://$domain$request_uri;
        }
}
server {
        listen 443 ssl;
        server_name $domain;
        ssl_certificate /etc/ssl/certs/server.crt;
        ssl_certificate_key /etc/ssl/certs/server.key;
        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;
        location / {
            proxy_pass         http://ipgeo;
        }
    }
EOF
## Docker compose file creating here-------------------------------------------------- 

cat <<EOF > /home/ec2-user/ip-api/docker-compose.yml 
version: '3'
services:
  app:
    image: redis:latest
    container_name: ipgeolocation-caching-service
    networks:
      - $mynet
  api_servcie:
    container_name: ipgeolocation-api-service
    env_file: .env
    environment:
      - REDIS_PORT=6379
      - REDIS_HOST=ipgeolocation-caching-service
      - APP_PORT=8080
      - API_KEY=$API_KEY
    networks:
      - $mynet
    ports:
      - "8080:8080"
    image: vyjith/ipgeolocation-api-service:v1
  frontend1:
    container_name: ipgeolocation-frontend-service-1
    image: vyjith/ipgeolocation-frontend:v1
    env_file: .env
    environment:
      - API_SERVER=ipgeolocation-api-service
      - API_SERVER_PORT=8080
      - API_PATH=/api/v1/
      - APP_PORT=8080
    networks:
      - $mynet
    ports:
      - "8081:8080"
  frontend2:
    container_name: ipgeolocation-frontend-service-2
    image: vyjith/ipgeolocation-frontend:v1
    env_file: .env
    environment:
      - API_SERVER=ipgeolocation-api-service
      - API_SERVER_PORT=8080
      - API_PATH=/api/v1/
      - APP_PORT=8080
    networks:
      - $mynet
    ports:
      - "8082:8080"
  frontend3:
    container_name: ipgeolocation-frontend-service-3
    image: vyjith/ipgeolocation-frontend:v1
    env_file: .env
    environment:
      - API_SERVER=ipgeolocation-api-service
      - API_SERVER_PORT=8080
      - API_PATH=/api/v1/
      - APP_PORT=8080
    networks:
      - $mynet
    ports:
      - "8083:8080"
  webserver:
    container_name: nginx
    image: nginx:1.15.12-alpine
    env_file: .env
    volumes:
      - ./nginx/:/etc/nginx/conf.d/
      - ./data/certs:/etc/nginx/certs
    networks:
      - $mynet
    ports:
      - "80:80"
      - "443:443"
    restart: always

networks:
  $mynet:
    driver: bridge
EOF

cat <<EOF > /home/ec2-user/ip-api/.env 

# If you would like to any details of env file here 
EOF


cat << EOF > /home/ec2-user/ip-api/data/certs/server.crt

Copy paste your ssl information here 

EOF

cat << EOF > /home/ec2-user/ip-api/data/certs/server.key

Copy paste your ssl key information here 

EOF


cd /home/ec2-user/ip-api
docker-compose up -d

}

docker_main(){
  docker_script
}
docker_main

```
