.PHONY: install update add_dep add_dev_dep test build_deps build_test build create_docker_network run run_dev destroy push docker_compose

project_name ?= docker-php-example

# Set image names to use when building docker images.
test_image_name ?= $(project_name)-test
browser_tests_image_name ?= $(project_name)-browser-tests
api_tests_image_name ?= $(project_name)-api-tests
deps_image_name ?= $(project_name)-deps
backend_image_name ?= $(project_name)-backend
server_image_name ?= $(project_name)-server
dev_backend_image_name ?= $(project_name)-dev-backend
dev_server_image_name ?= $(project_name)-dev-server
dev_test_image_name ?= $(project_name)-dev-test

# Specify local docker network names.
docker_network_name ?= $(project_name)
docker_server_name ?= $(project_name)-server
docker_backend_name ?= $(project_name)-backend
dev_docker_network_name ?= $(project_name)-dev
dev_docker_server_name ?= $(project_name)-dev-server
dev_docker_backend_name ?= $(project_name)-dev-backend

# Local docker host address.
host_address ?= host.docker.internal
xdebug_port ?= 9000

# Set the uid to use when building images.
ifeq ($(shell expr $$(id -u) \>= 999), 1)
	uid ?= $(shell echo $$(id -u))
else
	uid ?= 82
endif

# Set the gid to use when building images.
ifeq ($(shell expr $$(id -g) \>= 999), 1)
	gid ?= $(shell echo $$(id -g))
else
	gid ?= 82
endif

default: build

# Build an image to use to handle all installation and dependency management.
build_deps:
	docker build -t $(deps_image_name) --target dev_deps .

# Build an image to use to handle all testing.
build_test:
	docker build -t $(test_image_name) --target test .

build_browser_tests:
	docker build -t $(browser_tests_image_name) --target browser-tests .

build_api_tests:
	docker build -t $(api_tests_image_name) --target api-tests .

# Build na image to use to handle local testing.
build_test_dev:
	docker build -t $(dev_test_image_name) --build-arg uid=$(uid) --build-arg gid=$(gid) --target test .

# Build the projects docker images for production.
build:
	docker build -t $(server_image_name) --target server .
	docker build -t $(backend_image_name) --target backend .

# Build the projects docker images for development.
build_dev:
	docker build -t $(dev_server_image_name) --build-arg uid=$(uid) --target server .
	docker build -t $(dev_backend_image_name) --build-arg uid=$(uid) --build-arg gid=$(gid) --build-arg with_xdebug=true --target backend .

# Setup the docker container network.
create_docker_network:
	-docker network create $(docker_network_name)
	-docker network create $(dev_docker_network_name)

# Install all dependencies, including dev dependencies.
install: build_deps
	docker run -v $(CURDIR):/var/app:cached -u "$(uid):$(gid)" $(deps_image_name) install --no-plugins --ignore-platform-reqs

# Update all dependencies, including dev dependencies.
update: build_deps
	docker run -v $(CURDIR):/var/app:cached $(deps_image_name) update --no-plugins --ignore-platform-reqs

# Add a dependency package.
add_dep: build_deps
	docker run -it -v $(CURDIR):/var/app:cached $(deps_image_name) require --no-plugins --ignore-platform-reqs $(package)

# Add a dev-only dependency package.
add_dev_dep: build_deps
	docker run -it -v $(CURDIR):/var/app:cached $(deps_image_name) require --no-plugins --ignore-platform-reqs --dev $(package)

# Test our package for production.
test: destroy build_test create_docker_network
	docker run --network=$(docker_network_name) $(test_image_name)

browser-tests: run build_browser_tests create_docker_network
	docker run --network=$(docker_network_name) -e "APP_URL=http://$(docker_server_name):8080" $(browser_tests_image_name)

api-tests: run build_api_tests create_docker_network
	docker run --network=$(docker_network_name) -e "APP_URL=http://$(docker_server_name):8080" $(api_tests_image_name)

# Test our package locally.
test_dev: destroy build_test_dev create_docker_network
	docker run --network=$(dev_docker_network_name) -u "$(uid):$(gid)" -v $(CURDIR):/var/app:delegated $(dev_test_image_name)

# Spin up a production ready copy of the app.
run: destroy build create_docker_network
	docker run -d --network=$(docker_network_name) --name $(docker_backend_name) $(backend_image_name)
	docker run -d --network=$(docker_network_name) -p $(server_port):8080 -e PHP_HOST=$(docker_backend_name):9000 --name $(docker_server_name) $(server_image_name)

# Spin up a dev ready copy of the app, mounting local files, and watching backend logs.
run_dev: destroy install build_dev create_docker_network
	docker run -d --network=$(dev_docker_network_name) -u "$(uid):$(gid)" -e XDEBUG_CONFIG="remote_enable=1 remote_host=$(host_address) remote_port=$(xdebug_port)" --name $(dev_docker_backend_name) -v $(CURDIR):/var/app:cached $(dev_backend_image_name)
	docker run -d --network=$(dev_docker_network_name) -u "$(uid):$(gid)" -p $(server_port):8080 -e PHP_HOST=$(dev_docker_backend_name):9000 --name $(dev_docker_server_name) -v $(CURDIR)/public:/var/app/public:delegated $(dev_server_image_name)
	docker logs -f $(dev_docker_backend_name)

# Destroy any setup containers for the app.
destroy:
	-docker container rm --force $(docker_server_name) $(docker_backend_name) $(dev_docker_server_name) $(dev_docker_backend_name) $(browser_tests_image_name)
	-docker network rm $(docker_network_name) $(dev_docker_network_name)

# Build docker-compose images ready to use.
docker_compose_build:
	docker app render --set uid=$(uid) --set gid=$(gid) | docker-compose -f - build

# Render the docker compose file with the users uid/gid already set.
docker_compose:
	@docker app render --set uid=$(uid) --set gid=$(gid)

docker-compose.yml: docker-php-example.dockerapp
	docker app render --set uid=$(uid) --set gid=$(gid) > docker-compose.yml

k8s_dev: destroy install build test
	/bin/cp -rf $(CURDIR)/docker/k8s/overlays/development/deployment-patch-example.yml $(CURDIR)/docker/k8s/overlays/development/deployment-patch.yml
	sed -i "s#/path/to/project/dir#$(CURDIR)#g" $(CURDIR)/docker/k8s/overlays/development/deployment-patch.yml
	sed -i "s#SERVER_IMAGE#$(server_image_name)#g" $(CURDIR)/docker/k8s/overlays/development/deployment-patch.yml
	sed -i "s#BACKEND_IMAGE#$(backend_image_name)#g" $(CURDIR)/docker/k8s/overlays/development/deployment-patch.yml
