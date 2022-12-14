CLUSTER_NAME ?= kyma
KYMA_PROFILE ?= production
KYMA_SOURCE ?= main
REGISTRY_PORT ?= 5001
REGISTRY_NAME ?= ${CLUSTER_NAME}-registry

MANAGER_IMAGE_NAME ?= serverless-manager-dev-local
MANAGER_IMAGE_TAG ?= 0.0.1

PROJECT_ROOT ?= ../../..

TEMPLATE_PATH ?= ${PROJECT_ROOT}/template.yaml

KYMA ?= ${PROJECT_ROOT}/bin/kyma-unstable

.PHONY: kyma
kyma:
	@make -C ${PROJECT_ROOT} kyma

.PHONY: delete-k3d-registry
delete-k3d-registry:
	-k3d registry delete ${REGISTRY_NAME}

.PHONY: delete-k3d-cluster
delete-k3d-cluster:
	-k3d cluster delete ${CLUSTER_NAME}

.PHONY: create-k3d
create-k3d: 
	${KYMA} provision k3d --registry-port ${REGISTRY_PORT} --name ${CLUSTER_NAME}
	${KYMA} deploy --source ${KYMA_SOURCE} --profile ${KYMA_PROFILE} --component cluster-essentials --ci

.PHONY: delete-k3d
delete-k3d: delete-k3d-cluster delete-k3d-registry

.PHONY: module-image
module-image:
	@make -C ${PROJECT_ROOT} module-image \
		IMG=localhost:${REGISTRY_PORT}/${MANAGER_IMAGE_NAME}:${MANAGER_IMAGE_TAG}

.PHONY: module-build
module-build:
	@make -C ${PROJECT_ROOT} module-build \
		IMG=k3d-${REGISTRY_NAME}:${REGISTRY_PORT}/${MANAGER_IMAGE_NAME}:${MANAGER_IMAGE_TAG} \
		MODULE_REGISTRY=localhost:${REGISTRY_PORT}/unsigned

.PHONY: patch-coredns
patch-coredns:
	${PROJECT_ROOT}/hack/get_kyma_localhost_registry_name.sh k3d-${REGISTRY_NAME}

.PHONY: fix-template
fix-template:
	@cat ${PROJECT_ROOT}/template.yaml \
	| sed -e 's/remote/control-plane/g' \
		-e 's/${REGISTRY_PORT}/5000/g' \
	      	-e 's/localhost/k3d-${REGISTRY_NAME}.localhost/g' \
	> ${PROJECT_ROOT}/template-k3d.yaml

.PHONY: enable-module
enable-module:
	kubectl patch kymas.operator.kyma-project.io \
		default-kyma \
		-n kcp-system \
		--type=merge \
		--patch-file ./k3d-patches/patch-kyma.yaml

.PHONY: patch-mod-mgr-role
patch-mod-mgr-role:
	kubectl patch clusterrole \
		module-manager-manager-role \
		--patch-file=./k3d-patches/patch-k3d-module-manager-clusterrole.yaml

.PHONY: install-module
install-module:
	${KYMA} alpha deploy \
		--template=${PROJECT_ROOT}/template-k3d.yaml

.PHONY: run
run: kyma create-k3d module-image patch-coredns module-build fix-template install-module patch-mod-mgr-role enable-module verify

.PHONY: stop
stop: delete-k3d

.PHONY: verify
verify:
	@${PROJECT_ROOT}/hack/verify_kyma_status.sh
