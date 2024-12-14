# VERSION defines the version for the docker containers.
# To build a specific set of containers with a version,
# you can use the VERSION as an arg of the docker build command (e.g make docker VERSION=0.0.2)
VERSION ?= frankenphp

# REGISTRY defines the registry where we store our images.
# To push to a specific registry,
# you can use the REGISTRY as an arg of the docker build command (e.g make docker REGISTRY=my_registry.com/username)
# You may also change the default value if you are using a different registry as a default
REGISTRY ?= ghcr.io/victorjuniorrb/laravel-react-tasklist

# Commands
docker: docker-build docker-push

docker-build:
	docker build ../ -f ./Dockerfile -t ${REGISTRY}:${VERSION}

docker-push:
	docker push ${REGISTRY}:${VERSION}
