# Little demo for UBI

## Prerequisite 
podman installed

## Build 
podamn search ubi7

podman build -t rh/ubi-demo -f Dockerfile-ubi

podman images


## Run
podman run -d -p 8080:80 rh/ubi-demo


## Verify
curl localhost:8080
Hello CAP
