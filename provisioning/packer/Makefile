export BRANCH := $(shell git branch --contains | fgrep '*' | cut -d ' ' -f 2)
.DEFAULT_GOAL := all

.PHONY: all
all:
	make echo-commit_hash
	make app-build
	make init
	make build

.PHONY: base-build
base-build:
	make echo-commit_hash
	packer init -upgrade isucon14_base_image.pkr.hcl
	packer build -var "commit_hash=$$(git rev-parse HEAD)" isucon14_base_image.pkr.hcl

.PHONY: go-ami go-base-build go-build check-go-tools
go-ami:
	$(MAKE) check-go-tools
	aws sts get-caller-identity --query '[Account,Arn]' --output text
	$(MAKE) go-base-build
	$(MAKE) go-build

check-go-tools:
	@for tool in aws packer ansible-playbook task go node pnpm; do \
		if ! command -v "$$tool" >/dev/null 2>&1; then \
			echo "Missing required command: $$tool" >&2; \
			exit 1; \
		fi; \
	done

go-base-build:
	packer init -upgrade isucon14_base_image.pkr.hcl
	packer build -var "commit_hash=$$(git rev-parse HEAD)" -var "go_only=true" isucon14_base_image.pkr.hcl

go-build:
	$(MAKE) check-go-tools
	aws sts get-caller-identity --query '[Account,Arn]' --output text
	$(MAKE) app-build
	$(MAKE) init
	packer build -var "commit_hash=$$(git rev-parse HEAD)" -var "go_only=true" isucon14.pkr.hcl

.PHONY: echo-commit_hash
echo-commit_hash:
	git rev-parse HEAD

.PHONY: app-build
app-build: 
	cd ../../provisioning/ansible/ && ./make_latest_files.sh

.PHONY: init
init:
	packer init -upgrade isucon14.pkr.hcl

.PHONY: build
build:
	packer build -var "commit_hash=$$(git rev-parse HEAD)" isucon14.pkr.hcl
