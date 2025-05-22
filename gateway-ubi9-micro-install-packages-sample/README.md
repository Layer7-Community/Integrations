# Add Packages to Gateway Container

## Description
The container image for Layer 7 API Gateway 11.1.2 is built on the ubi9-micro base image and does not have a package manager pre-installed. 
The Dockerfile provides an example of how to add packages to the 11.1.2 container image.

## Execute the following command to build the image:

   `DOCKER_BUILDKIT=1 docker build -t <IMAGE_NAME> --no-cache --build-arg GW_IMAGE=<GATEWAY_IMAGE> -f Dockerfile .`