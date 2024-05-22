include .env
export $(shell sed 's/=.*//' .env)

ifeq ($(OS), Windows_NT)
        SHELL := powershell.exe
        .SHELLFLAGS := -NoProfile -Command
        SHELL_VERSION = $(shell (Get-Host | Select-Object Version | Format-Table -HideTableHeaders | Out-String).Trim())
        OS = $(shell "{0} {1}" -f "windows", (Get-ComputerInfo -Property OsVersion, OsArchitecture | Format-Table -HideTableHeaders | Out-String).Trim())
        PACKAGE = $(shell (Get-Content go.mod -head 1).Split(" ")[1])
        CHECK_DIR_CMD = if (!(Test-Path $@)) { $$e = [char]27; Write-Error "$$e[31mDirectory $@ doesn't exist$${e}[0m" }
        HELP_CMD = Select-String "^[a-zA-Z_-]+:.*?\#\# .*$$" "./Makefile" | Foreach-Object { $$_data = $$_.matches -split ":.*?\#\# "; $$obj = New-Object PSCustomObject; Add-Member -InputObject $$obj -NotePropertyName ('Command') -NotePropertyValue $$_data[0]; Add-Member -InputObject $$obj -NotePropertyName ('Description') -NotePropertyValue $$_data[1]; $$obj } | Format-Table -HideTableHeaders @{Expression={ $$e = [char]27; "$$e[36m$$($$_.Command)$${e}[0m" }}, Description
else
        SHELL := bash
        SHELL_VERSION = $(shell echo $$BASH_VERSION)
        UNAME := $(shell uname -s)
        VERSION_AND_ARCH = $(shell uname -rm)
        ifeq ($(UNAME),Darwin)
                OS = macos ${VERSION_AND_ARCH}
        else ifeq ($(UNAME),Linux)
                OS = linux ${VERSION_AND_ARCH}
        else
                $(error OS not supported by this Makefile)
        endif
        PACKAGE = $(shell head -1 go.mod | awk '{print $$2}')
        CHECK_DIR_CMD = test -d $@ || (echo "\033[31mDirectory $@ doesn't exist\033[0m" && false)
        HELP_CMD = grep -E '^[a-zA-Z_-]+:.*?\#\# .*$$' Makefile | awk 'BEGIN {FS = ":.*?\#\# "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'
endif

.DEFAULT_GOAL := help
.PHONY: clean compile static-tests unit-tests integration-tests package format run-containers stop-containers about help

clean: $@ ## Clean generated files
        [ -d ./api/gen ] && rm -r ./api/gen || true
        rm -r ./bin/* || true
        #go clean -modcache
        #find . -type f -name '*.pb.go' -delete
        #find . -type f -name '*.pb.gw.go' -delete

NORBIX_GIT_COMMIT= $(shell git log -n 1 --format=%h)
NORBIX_GOOS=l$(shell go env GOOS)
NORBIX_GOARCH=$(shell go env GOARCH)
compile: ## Compile the code
        go build -o ./bin/${NORBIX_GIT_COMMIT}-${NORBIX_GOOS}-${NORBIX_GOARCH}-webapp.bin ./cmd/main.go

static-tests: yamllint golanci-lint ## Run static tests

yamllint:
        yamllint -c ./yamllint.yaml -s . --no-warnings

golangci_lint_version = v1.52.2

golangci-lint:
        docker run --rm -v $(shell pwd):/app -v ~/.cache/golangci-list/${golangco-lint_version}:/root/.cache \
                                                                                 -w /app golangci/golangci-lint:${golangci_lint_version} \
                                                                                 golangci-lint \
                                                                                 run \
                                                                                 -c .golangci.yaml \
                                                                                 -v

unit-tests: ## Run unit tests
        go clean -testcache
        go test -v ./... -coverprofile=coverage.out

integration-tests: ## Run integration tests
        go clean -testcache
        go test -v ./... -coverprofile=coverage.out -tags=integration

NETRC_MACHINE ?= replace
NETRC_USER ?= replace
NETRC_PASSWORD ?= replace
DOCKERFILE_NAME ?= Dockerfile
package: ## Package the compiled code into a Docker image
        # Evaluate the variables
        echo "${NETRC_MACHINE}"
        echo "${NETRC_USER}"
        echo "${NETRC_PASSWORD}"
        echo "${DOCKERFILE_NAME}"
        # Build the docker image
        docker build --build-arg NETRC_MACHINE="${NETRC_MACHINE}" \
                                 --build-arg NETRC_USER="${NETRC_USER}" \
                                 --build-arg NETRC_PASSWORD="${NETRC_PASSWORD}" \
                                 -f "${DOCKERFILE_NAME}" \
                                 -t local/golang-example-backend-app:latest \
                                 .

format: format-code format-imports ## Format the code

format-code:
        gofmt -w .

format-imports:
        goimports -w .

NETRC_MACHINE ?= replace
NETRC_USER ?= replace
NETRC_PASSWORD ?= replace
run-containers: ## Run application locally in Docker engine
        docker-compose up -d && docker-compose ps

stop-containers: ## Stop application locally in Docker engine
        docker-compose down

about: ## Display info related to the build
        @echo "OS: ${OS}"
        @echo "Shell: ${SHELL} ${SHELL_VERSION}"
        @echo "Protoc version: $(shell protoc --version)"
        @echo "Go version: $(shell go version)"
        @echo "Go package: ${PACKAGE}"
        @echo "Openssl version: $(shell openssl version)"

help: ## Show this help
        @${HELP_CMD}

MOCKERY = $(shell pwd)/bin/mockery
mockery:
        @echo "Downloading mockery locally if necessary"
        brew install mockery || true

MOCK_DIRS := controller middleware repository service

.PHONY: mocks-local
mocks-local:
        @echo "Generating mocks locally"
        rm -rf ./internal/mocks && mkdir -p ./internal/mocks
        for dir in $(MOCK_DIRS); do \
                mockery --dir internal/$$dir --all --keeptree --output internal/mocks/$$dir; \
        done

install-golangci-lint:
        @echo "Installing golangci-lint"
        @mkdir -p bin
        test ! -f bin/golangci-lint &&
        curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s ${golangci_lint_version}

.PHONY: install-imports
install-imports:
        @echo "Installing goimports"
        $(if $(shell which goimports),$(),$(shell go install golang.org/x/tools/cmd/goimports@latest))

run:
        go run ./cmd/main.go