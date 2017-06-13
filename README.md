# Jenkins
This repository is used to demonstrate a Jenkins pipeline for building and deploying Docker containers. The Jenkins server is built from the root Dockerfile and tutorials exist in the tutorials repo.

## Build
Build from the Dockerfile.
```
docker build -t <image_name> .
```
Build from the docker-compose file.
```
docker-compose build
```

## Run
Using Docker run.
```
docker run -d -p 8080:8080 <image_name>
```
Using docker-compose.
```
docker-compose up -d
```


## Deploy
Most users should be satisfied running the tutorials locally, however, if you want to test triggered builds the server will need to be available to Github. It is recommended you use the docker-compose.production.yml file for deploying to a server that is available over the internet. This setup uses a proxy that integrates with Let's Encrypt to serve Jenkins over HTTPS.

_Make sure to modify the docker-compose.production.yml file to use your domain name under leproxy command._

```
docker-compose -f docker-compose.production.yml up -d
```

_Note: the setup as is has not been validated for a production environment and should be used for test and demo purposes only._
