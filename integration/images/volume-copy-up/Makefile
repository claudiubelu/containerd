#   Copyright The containerd Authors.

#   Licensed under the Apache License, Version 2.0 (the "License");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at

#       http://www.apache.org/licenses/LICENSE-2.0

#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.

all: build

PROJ=gcr.io/k8s-cri-containerd
VERSION=1.1
IMAGE=$(PROJ)/volume-copy-up:$(VERSION)

# Architectures supported: amd64, arm, arm64, ppc64le and s390x
ARCH ?= amd64
# Operating systems supported: linux, windows
OS ?= linux
# OS Version for the Windows images: 1809, 1903, 1909 2004, 20H2
OSVERSION ?= 1809 1903 1909 2004 20H2

# The output type could either be docker (local), or registry.
# If it is registry, it will also allow us to push the Windows images.
OUTPUT_TYPE ?= docker

ALL_OS = linux windows
ALL_ARCH.linux = amd64
ALL_OS_ARCH.linux = $(foreach arch, ${ALL_ARCH.linux}, linux-$(arch))
ALL_ARCH.windows = amd64
ALL_OSVERSIONS.windows := 1809 1903 1909 2004 20H2
ALL_OS_ARCH.windows = $(foreach arch, $(ALL_ARCH.windows), $(foreach osversion, ${ALL_OSVERSIONS.windows}, windows-$(arch)-${osversion}))
ALL_OS_ARCH = $(foreach os, $(ALL_OS), ${ALL_OS_ARCH.${os}})

BASE.linux := busybox
BASE.windows := mcr.microsoft.com/windows/nanoserver
BASE := ${BASE.${OS}}

all-container: all-container-docker

all-push: all-container-registry push-manifest

all-container-docker: $(addprefix sub-container-docker-,$(ALL_OS_ARCH.linux))
all-container-registry: $(addprefix sub-container-registry-,$(ALL_OS_ARCH))

# split words on hyphen, access by 1-index
word-hyphen = $(word $2,$(subst -, ,$1))
sub-container-%:
	$(MAKE) OUTPUT_TYPE=$(call word-hyphen,$*,1) OS=$(call word-hyphen,$*,2) ARCH=$(call word-hyphen,$*,3) OSVERSION=$(call word-hyphen,$*,4) container

container: .container-${OS}-$(ARCH)

.container-linux-$(ARCH):
	docker buildx build --pull --output=type=${OUTPUT_TYPE} --platform ${OS}/${ARCH} \
		-t $(IMAGE)-${OS}-${ARCH} --build-arg BASE=${BASE} .

.container-windows-$(ARCH):
	docker buildx build --pull --output=type=${OUTPUT_TYPE} --platform ${OS}/${ARCH} \
		-t $(IMAGE)-${OS}-${ARCH}-${OSVERSION} --build-arg BASE=${BASE}:${OSVERSION} .

push-manifest:
	docker manifest create --amend $(IMAGE) $(shell echo $(ALL_OS_ARCH) | sed -e "s~[^ ]*~$(IMAGE)\-&~g")
	set -x; for arch in $(ALL_ARCH.linux); do docker manifest annotate --os linux --arch $${arch} ${IMAGE} ${IMAGE}-linux-$${arch}; done
	# For Windows images, we also need to include the "os.version" in the manifest list, so the Windows node can pull the proper image it needs.
	# At the moment, docker manifest annotate doesn't allow us to set the os.version, so we'll have to it ourselves. The manifest list can be found locally as JSONs.
	# See: https://github.com/moby/moby/issues/41417
	# If the ${REGISTRY} is on dockerhub, the prefix "docker.io/" is optional. However, we need the full
	# registry name for setting the os.version for Windows images below.
	# If the registry name does not contain any slashes, we prepend "docker.io/"
	# TODO(claudiub): Clean this up once the above issue has been fixed.
	set -x; \
	registry_prefix=$(shell (echo ${PROJ} | grep -Eq ".*\/.*") && echo "" || echo "docker.io/"); \
	manifest_image_folder=`echo "$${registry_prefix}${IMAGE}" | sed "s|/|_|g" | sed "s/:/-/"`; \
	for arch in $(ALL_ARCH.windows);  do \
		for osversion in ${ALL_OSVERSIONS.windows}; do \
			docker manifest annotate --os windows --arch $${arch} ${IMAGE} ${IMAGE}-windows-$${arch}-$${osversion}; \
			BASEIMAGE=${BASE.windows}:$${osversion}; \
			full_version=`docker manifest inspect ${BASE.windows}:$${osversion} | grep "os.version" | head -n 1 | awk '{print $$2}'` || true; \
			sed -i -r "s/(\"os\"\:\"windows\")/\0,\"os.version\":$${full_version}/" "${HOME}/.docker/manifests/$${manifest_image_folder}/$${manifest_image_folder}-windows-$${arch}-$${osversion}"; \
		done; \
	done
	docker manifest push --purge ${IMAGE}

.PHONY: all-container container all-push
