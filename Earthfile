VERSION 0.7

base-image:
    FROM alpine:3.18

goreleaser:
    FROM goreleaser/goreleaser-pro:v1.21.2-pro
    SAVE ARTIFACT /usr/bin/goreleaser

builder-image:
    FROM +base-image
    RUN apk update && apk add go=1.20.10-r0 git curl make pkgconfig bash docker jq
    ENV GOPATH /go
    ENV PATH $PATH:$GOPATH/bin
    ENV CGO_ENABLED=0
    RUN curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin v1.53.3
    RUN --mount=type=cache,id=gomod,target=${GOPATH}/pkg/mod \
        --mount=type=cache,id=gobuild,target=/root/.cache/go-build \
        go install github.com/euank/gotmpl/cmd/gotmpl@latest
    COPY (+goreleaser/*) /usr/bin/goreleaser

deployer-image:
    FROM +base-image
    RUN apk update && \
        apk add --repository=http://dl-cdn.alpinelinux.org/alpine/edge/community kubectl && \
        apk add --repository=http://dl-cdn.alpinelinux.org/alpine/edge/community kustomize && \
        apk add helm git jq
    ARG --required KUBE_TOKEN
    ARG --required KUBE_APISERVER
    RUN kubectl config set clusters.default.server ${KUBE_APISERVER}
    RUN kubectl config set clusters.default.insecure-skip-tls-verify true
    RUN kubectl config set-credentials default --token=${KUBE_TOKEN}
    RUN kubectl config set-context default --cluster=default --user=default
    RUN kubectl config use-context default

final-image:
    FROM +base-image
    RUN apk update && apk add ca-certificates curl

GO_TESTS:
    COMMAND
    ARG GOPROXY
    ARG component
    RUN --mount type=cache,id=go-$component,target=${GOPATH}/pkg/mod \
        --mount=type=cache,id=gobuild,target=/root/.cache/go-build \
        go test ./...

GO_COMPILE:
    COMMAND
    ARG GOPROXY
    ARG VERSION=latest
    ARG EARTHLY_BUILD_SHA
    ARG component
    ENV GIT_PATH "$(head -1 go.mod | cut -d\\  -f2)"
    RUN --mount type=cache,id=go-$component,target=${GOPATH}/pkg/mod \
        --mount=type=cache,id=gobuild,target=/root/.cache/go-build \
        go build -o main \
        -ldflags="-X ${GIT_PATH}/cmd.Version=${VERSION} \
        -X ${GIT_PATH}/cmd.BuildDate=$(date +%s) \
        -X ${GIT_PATH}/cmd.Commit=${EARTHLY_BUILD_SHA}" ./
    SAVE ARTIFACT main

GO_INSTALL:
    COMMAND
    ARG package
    ARG GOPROXY
    ARG component
    RUN --mount type=cache,id=go-$component,target=${GOPATH}/pkg/mod \
        --mount=type=cache,id=gobuild,target=/root/.cache/go-build \
        go install ${package}

SAVE_IMAGE:
    COMMAND
    ARG TAG=latest
    ARG --required COMPONENT
    ARG REPOSITORY=ghcr.io
    ENV OTEL_SERVICE_NAME ${COMPONENT}
    SAVE IMAGE --push ${REPOSITORY}/formancehq/${COMPONENT}:${TAG}

GO_MOD_TIDY:
    COMMAND
    ARG GOPROXY
    ARG component
    RUN --mount type=cache,id=go-$component,target=${GOPATH}/pkg/mod \
        --mount=type=cache,id=gobuild,target=/root/.cache/go-build \
        go mod tidy

GO_INSTALL:
    COMMAND
    DO --pass-args GO_INSTALL

GO_GENERATE:
    COMMAND
    ARG GOPROXY
    ARG component
    RUN --mount type=cache,id=go-$component,target=${GOPATH}/pkg/mod \
        --mount=type=cache,id=gobuild,target=/root/.cache/go-build \
        go generate ./...
