VERSION --pass-args --global-cache --arg-scope-and-set --use-function-keyword 0.7

base-image:
    FROM alpine:3.19

goreleaser:
    FROM goreleaser/goreleaser-pro:v1.23.0-pro
    SAVE ARTIFACT /usr/bin/goreleaser

golangci-lint:
    FROM golangci/golangci-lint:v1.55.2
    SAVE ARTIFACT /usr/bin/golangci-lint

syft:
    FROM anchore/syft:v0.103.1
    SAVE ARTIFACT /syft

builder-image:
    FROM +base-image
    RUN apk update && apk add go git curl make pkgconfig bash docker jq
    ENV GOPATH /go
    ENV GOTOOLCHAIN=local
    ARG GOCACHE=/go-cache
    ARG GOMODCACHE=/go-mod-cache
    ENV PATH $GOPATH/bin:/usr/local/go/bin:$PATH
    ENV CGO_ENABLED=0
    COPY (+golangci-lint/*) /usr/bin/golangci-lint
    COPY (+goreleaser/*) /usr/bin/goreleaser
    COPY (+syft/*) /usr/bin/syft

deployer-image:
    FROM +base-image
    RUN apk update && \
        apk add --repository=http://dl-cdn.alpinelinux.org/alpine/edge/community kubectl && \
        apk add --repository=http://dl-cdn.alpinelinux.org/alpine/edge/community kustomize && \
        apk add helm git jq
    RUN --secret KUBE_APISERVER kubectl config set clusters.default.server ${KUBE_APISERVER}
    RUN kubectl config set clusters.default.insecure-skip-tls-verify true
    RUN --secret KUBE_TOKEN kubectl config set-credentials default --token=${KUBE_TOKEN}
    RUN kubectl config set-context default --cluster=default --user=default
    RUN kubectl config use-context default

vcluster-deployer-image:
    FROM +deployer-image
    COPY --dir ./vcluster/charts/tld-certificates .
    COPY --dir ./vcluster/charts/vcluster-ingress .
    ARG --required user
    RUN --secret tld helm upgrade --install vcluster-$user-ingress ./vcluster-ingress \
        --namespace vcluster-$user \
        --create-namespace \
        --set tld=$tld \
        --set user=$user
    COPY ./vcluster/values.yaml .
    RUN helm package ./tld-certificates
    ENV tldCertificatesChartBundled=$(cat tld-certificates-0.6.1.tgz | base64 -w 0)

    RUN echo "user: $user" > /tmp/values.yaml
    RUN --secret tld echo "tld: $tld" >> /tmp/values.yaml

    LET values=$(cat /tmp/values.yaml)
    RUN --secret tld helm upgrade --install $user vcluster \
        --repo https://charts.loft.sh \
        --namespace vcluster-$user \
        --set syncer.extraArgs[0]="--tls-san=kube.$user.$tld" \
        --set syncer.extraArgs[1]="--out-kube-config-server=https://kube.$user.$tld" \
        --set syncer.extraArgs[2]="--out-kube-config-secret=vc-$user" \
        --set init.helm[0].bundle=$tldCertificatesChartBundled \
        --set init.helm[0].chart.name=tld-certificates \
        --set init.helm[0].chart.version=0.6.1 \
        --set init.helm[0].values="$values" \
        --set init.helm[0].release.name=tld-certificates \
        --set init.helm[0].release.namespace=formance \
        --values values.yaml \
        --repository-config=''
    RUN kubectl -n vcluster-$user get secrets/vc-$user || kubectl -n vcluster-$user create secret generic vc-$user
    RUN while [ "$(kubectl -n vcluster-$user get secrets/vc-$user -o jsonpath='{.data.config}')" == "" ]; do sleep 1s; done
    RUN kubectl -n vcluster-$user get secrets/vc-$user -o jsonpath='{.data.config}' | base64 -d > /root/.kube/vcluster-config
    ENV KUBECONFIG=/root/.kube/vcluster-config
    RUN chmod 0700 /root/.kube/vcluster-config

    SAVE IMAGE vcluster-$user

final-image:
    FROM +base-image
    RUN apk update && apk add ca-certificates curl

run-in-all-vclusters:
    FROM +deployer-image
    ARG --required cmd
    FOR user IN $(helm list --all-namespaces | grep vcluster-0.16.4 | cut -d\  -f1)
        FROM +vcluster-deployer-image --user=$user
        RUN --no-cache $cmd
    END

helm-base:
    FROM +base-image
    RUN apk update && apk add openssl helm


grpc-generate:
    FROM +builder-image
    RUN apk add --no-cache protobuf git protobuf-dev && \
        go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.28 && \
        go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.2
    WORKDIR /src
    RUN mkdir generated
    ARG --required protoName
    COPY $protoName .
    RUN protoc --go_out=generated --go_opt=paths=source_relative --go-grpc_out=generated --go-grpc_opt=paths=source_relative $protoName
    SAVE ARTIFACT generated AS LOCAL internal/generated

GO_TESTS:
    FUNCTION
    ARG GOPROXY
    RUN --mount type=cache,id=gopkgcache,target=${GOPATH}/pkg/mod \
        --mount type=cache,id=gobuildcache,target=/root/.cache/go-build \
        go test ./... -coverprofile=./cover.out -covermode=atomic -coverpkg=./...

GO_COVERAGE:
    FUNCTION
    ARG EARTHLY_BUILD_SHA
    RUN --secret CODECOV_TOKEN codecov -t $CODECOV_TOKEN -f cover.out -F ledger -C $EARTHLY_BUILD_SHA

GO_LINT:
    FUNCTION
    COPY (+sources/out --LOCATION=.golangci.yml) .golangci.yml
    ARG GOPROXY
    ARG GOCACHE=/go-cache
    ARG GOMODCACHE=/go-mod-cache
    RUN --mount=type=cache,id=golangci,target=/root/.cache/golangci-lint \
        golangci-lint run --fix ./...

GO_COMPILE:
    FUNCTION
    ARG GOPROXY
    ARG VERSION=latest
    ARG EARTHLY_BUILD_SHA
    LET GIT_PATH=$(head -1 go.mod | cut -d\\  -f2)
    RUN --mount type=cache,id=gopkgcache,target=${GOPATH}/pkg/mod \
        --mount type=cache,id=gobuildcache,target=/root/.cache/go-build \
        go build -o main \
        -ldflags="-X ${GIT_PATH}/cmd.Version=${VERSION} \
        -X ${GIT_PATH}/cmd.BuildDate=$(date +%s) \
        -X ${GIT_PATH}/cmd.Commit=${EARTHLY_BUILD_SHA}" ./
    SAVE ARTIFACT main

GO_INSTALL:
    FUNCTION
    ARG package
    ARG GOPROXY
    RUN --mount type=cache,id=gopkgcache,target=${GOPATH}/pkg/mod \
        --mount type=cache,id=gobuild,target=/root/.cache/go-build \
        go install ${package}

SAVE_IMAGE:
    FUNCTION
    ARG TAG=latest
    ARG --required COMPONENT
    ARG REPOSITORY=ghcr.io
    ENV OTEL_SERVICE_NAME ${COMPONENT}
    # todo(gfyrag): make insecure configurable
    SAVE IMAGE --push --insecure ${REPOSITORY}/formancehq/${COMPONENT}:${TAG}

GO_MOD_TIDY:
    FUNCTION
    ARG GOPROXY
    RUN --mount type=cache,id=gopkgcache,target=${GOPATH}/pkg/mod \
        --mount type=cache,id=gobuildcache,target=/root/.cache/go-build \
        go mod tidy

GO_INSTALL:
    FUNCTION
    DO --pass-args GO_INSTALL

GO_GENERATE:
    FUNCTION
    ARG GOPROXY
    RUN --mount type=cache,id=gopkgcache,target=${GOPATH}/pkg/mod \
        --mount type=cache,id=gobuildcache,target=/root/.cache/go-build \
        go generate ./...

HELM_VALIDATE:
    FUNCTION
    ARG additionalArgs
    RUN helm lint ./ --strict $additionalArgs
    RUN helm template ./ $additionalArgs
