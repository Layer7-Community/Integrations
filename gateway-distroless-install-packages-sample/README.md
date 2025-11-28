# Add Packages to Gateway Container

## Description
The container image for Layer 7 API Gateway 11.2.0 is built on the distroless base image and does not have a package manager pre-installed.
The Dockerfile provides an example of how to add packages to the 11.2.0 container image.

## To Add package(s) to the Gateway

To add another package to the Gateway image:

Check the dependencies of the package to be added using `apt-cache depends <package_name>` and `apt-rdepends <package_name>` and see if the dependencies contain virtual packages or a dependency choice
`apt-cache depends` indicates virtual package dependencies by package name surrounded by `<` and `>`, and dependency choice by `|` in front of Depends or PreDepends

e.g. Following shows that iproute2 depends on either debconf or debconf-2.0, which is a virtual package dependency that can be satisfied by cdebconf or debconf itself
```
# apt-cache depends iproute2
iproute2
 |Depends: debconf
  Depends: <debconf-2.0>
    cdebconf
    debconf
```

`apt-rdepends` shows all dependencies of the package to be added. It not only shows direct dependencies but also all dependencies of dependencies.

The sample dockerfile shows adding curl and jq packages along with dependencies found using `apt-cache depends` and `apt-rdepends` .

## Execute the following command to build the image:

`DOCKER_BUILDKIT=1 docker buildx build -t <IMAGE_NAME>   --platform linux/amd64 --no-cache --build-arg GW_IMAGE=<GATEWAY_IMAGE> -f Dockerfile .`