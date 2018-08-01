SHELL:=/bin/bash
ENGINE_DIR:=$(CURDIR)/../../engine
CLI_DIR:=$(CURDIR)/../../cli
VERSION?=0.0.0-dev
STATIC_VERSION=$(shell ./gen-static-ver $(ENGINE_DIR) $(VERSION))
CHOWN=docker run --rm -v $(CURDIR):/v -w /v alpine chown
HASH_CMD=docker run -v $(CURDIR):/sum -w /sum debian:jessie bash hash_files
DIR_TO_HASH:=build/linux
CONTAINERD_REPO=containerd/containerd
CONTAINERD_DIR=$(shell basename $(CONTAINERD_REPO))
CONTAINERD_REF=468a545b9edcd5932818eb9de8e72413e616e86e
CTR_BIN=$(CURDIR)/$(CONTAINERD_DIR)/bin/ctr.exe
CONTAINERD_BIN=$(CURDIR)/$(CONTAINERD_DIR)/bin/containerd.exe
SHIM_BIN=$(CURDIR)/$(CONTAINERD_DIR)/bin/containerd-shim.exe
CONTAINDER_BINARIES=$(CTR_BIN) $(CONTAINERD_BIN) $(SHIM_BIN)

.PHONY: help
help: ## show make targets
	@awk 'BEGIN {FS = ":.*?## "} /^[a-zA-Z_-]+:.*?## / {sub("\\\\n",sprintf("\n%22c"," "), $$2);printf " \033[36m%-20s\033[0m  %s\n", $$1, $$2}' $(MAKEFILE_LIST)

.PHONY: clean
clean: ## remove build artifacts
	[ ! -d build ] || $(CHOWN) -R $(shell id -u):$(shell id -g) build
	$(RM) -r build

.PHONY: static
static: static-linux cross-mac cross-win cross-arm ## create all static packages

.PHONY: static-linux
static-linux: static-cli static-engine ## create tgz with linux x86_64 client and server
	mkdir -p build/linux/docker
	cp $(CLI_DIR)/build/docker build/linux/docker/
	for f in dockerd docker-containerd docker-containerd-ctr docker-containerd-shim docker-init docker-proxy docker-runc; do \
		cp -L $(ENGINE_DIR)/bundles/binary-daemon/$$f build/linux/docker/$$f; \
	done
	tar -C build/linux -c -z -f build/linux/docker-$(STATIC_VERSION).tgz docker

.PHONY: hash_files
hash_files:
	@echo "Hashing directory $(DIR_TO_HASH)"
	$(HASH_CMD) "$(DIR_TO_HASH)"

.PHONY: cross-mac
cross-mac: cross-all-cli ## create tgz with darwin x86_64 client only
	mkdir -p build/mac/docker
	cp $(CLI_DIR)/build/docker-darwin-amd64 build/mac/docker/docker
	tar -C build/mac -c -z -f build/mac/docker-$(STATIC_VERSION).tgz docker

.PHONY: cross-win
cross-win: cross-all-cli cross-win-engine $(CONTAINERD_BINARIES) ## create zip file with windows x86_64 client and server
	mkdir -p build/win/docker
	cp $(CLI_DIR)/build/docker-windows-amd64 build/win/docker/docker.exe
	cp $(ENGINE_DIR)/bundles/cross/windows/amd64/dockerd-$(VERSION).exe build/win/docker/dockerd.exe
	for binary in $(CONTAINERD_BINARIES); do ; \
		cp $$binary build/win/docker/$$(basename $$binary); \
	done
	docker run --rm -v $(CURDIR)/build/win:/v -w /v alpine sh -c 'apk update&&apk add zip&&zip -r docker-$(STATIC_VERSION).zip docker'
	$(CHOWN) -R $(shell id -u):$(shell id -g) build

.PHONY: cross-arm
cross-arm: cross-all-cli ## create tgz with linux armhf client only
	mkdir -p build/arm/docker
	cp $(CLI_DIR)/build/docker-linux-arm build/arm/docker/docker
	tar -C build/arm -c -z -f build/arm/docker-$(STATIC_VERSION).tgz docker

.PHONY: static-cli
static-cli:
	$(MAKE) -C $(CLI_DIR) -f docker.Makefile VERSION=$(VERSION) build

.PHONY: static-engine
static-engine:
	$(MAKE) -C $(ENGINE_DIR) VERSION=$(VERSION) binary

.PHONY: cross-all-cli
cross-all-cli:
	$(MAKE) -C $(CLI_DIR) -f docker.Makefile VERSION=$(VERSION) cross

.PHONY: cross-win-engine
cross-win-engine:
	$(MAKE) -C $(ENGINE_DIR) VERSION=$(VERSION) DOCKER_CROSSPLATFORMS=windows/amd64 cross

$(CONTAINERD_DIR):
	git clone git@github.com:$(CONTAINERD_REPO)
	git checkout -b build $(CONTAINERD_REF)

$(CTR_BIN): $(CONTAINERD_DIR)
	$(MAKE) -C $(CONTAINERD_DIR) bin/ctr

$(CONTAINDERD_BIN): $(CONTAINDERD_DIR)
	$(MAKE) -C $(CONTAINERD_DIR) bin/containerd

$(SHIM_BIN): $(CONTAINERD_DIR)
	$(MAKE) -C $(CONTAINERD_DIR) bin/containerd-shim
