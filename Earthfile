VERSION 0.8

IMPORT github.com/formancehq/stack/libs/go-libs:feat/monorepo AS libs
IMPORT github.com/formancehq/stack/libs/core:feat/monorepo AS core
IMPORT github.com/formancehq/stack/releases:feat/monorepo AS releases

sources:
    FROM +base-image
    ARG --required LOCATION
    COPY ${LOCATION} out
    SAVE ARTIFACT out

base-image:
    FROM alpine:3.20

sources-goreleaser:
    FROM goreleaser/goreleaser-pro:v2.1.0-pro
    SAVE ARTIFACT /usr/bin/goreleaser

golangci-lint:
    FROM golangci/golangci-lint:v1.60.3
    SAVE ARTIFACT /usr/bin/golangci-lint

sources-syft:
    FROM anchore/syft:v0.103.1
    SAVE ARTIFACT /syft

sources-speakeasy:
    FROM +base-image
    RUN apk update && apk add yarn jq unzip curl
    ARG VERSION=v1.351.0
    ARG TARGETARCH
    RUN echo $VERSION
    RUN curl -fsSL https://github.com/speakeasy-api/speakeasy/releases/download/${VERSION}/speakeasy_linux_$TARGETARCH.zip -o /tmp/speakeasy_linux_$TARGETARCH.zip
    RUN unzip /tmp/speakeasy_linux_$TARGETARCH.zip speakeasy
    RUN chmod +x speakeasy
    SAVE ARTIFACT speakeasy

builder-image:
    FROM +base-image
    RUN apk update && apk add go git curl make pkgconfig bash docker jq
    ENV GOPATH /go
    ENV GOTOOLCHAIN=local
    ARG GOCACHE=/go-cache
    ARG GOMODCACHE=/go-mod-cache
    ENV PATH $GOPATH/bin:/usr/local/go/bin:$PATH
    ENV CGO_ENABLED=0
    COPY (+sources-golangci-lint/*) /usr/bin/golangci-lint
    COPY (+sources-goreleaser/*) /usr/bin/goreleaser
    COPY (+sources-syft/*) /usr/bin/syft

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
        --version v0.19.7 \
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

grpc-base:
    FROM +builder-image
    RUN apk add --no-cache protobuf git protobuf-dev && \
        go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.28 && \
        go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.2

base-argocd:
    FROM --pass-args +base-image
    RUN apk add curl
    RUN curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64 && chmod 555 /usr/local/bin/argocd

# Should rename as `application-set`
# Deprecated stack #1479
deployer-module:
    FROM --pass-args +base-argocd 

    ARG WITH_SYNC=true
    ARG --required ARGS
    LET APPLICATION=staging-eu-west-1-hosting-regions
    LET SERVER=argocd.internal.formance.cloud
    
    RUN --secret AUTH_TOKEN \
        argocd app set $APPLICATION \ 
        $ARGS \
        --auth-token=$AUTH_TOKEN --server=$SERVER --grpc-web

    IF [ "$WITH_SYNC" = "true" ]
        BUILD +deploy-staging --APPLICATION=staging-eu-west-1-hosting-regions
    END


# Should rename as `application-sync`
# Deprecated stack #1479
deploy-staging:
    FROM --pass-args +base-argocd
    
    ARG --required APPLICATION
    LET SERVER=argocd.internal.formance.cloud
    
    RUN --secret AUTH_TOKEN \
        argocd app sync $APPLICATION --auth-token $AUTH_TOKEN --server $SERVER --grpc-web


application-set:
    FROM --pass-args +base-argocd 

    ARG --required ARGS
    ARG WITH_SYNC=false
    LET APPLICATION=staging-eu-west-1-hosting-regions
    LET SERVER=argocd.internal.formance.cloud
    
    RUN --secret AUTH_TOKEN \
        argocd app set $APPLICATION \ 
        $ARGS \
        --auth-token=$AUTH_TOKEN --server=$SERVER --grpc-web

    IF [ "$WITH_SYNC" = "true" ]
        BUILD +application-sync --APPLICATION=staging-eu-west-1-hosting-regions
    END

application-sync:
    FROM --pass-args +base-argocd
    
    ARG --required APPLICATION
    LET SERVER=argocd.internal.formance.cloud
    
    RUN --secret AUTH_TOKEN \
        argocd app sync $APPLICATION --auth-token $AUTH_TOKEN --server $SERVER --grpc-web

GORELEASER:
    FUNCTION
    WORKDIR /src
    ARG mode=local
    LET buildArgs = --clean
    IF [ "$mode" = "local" ]
        SET buildArgs = --nightly --skip=publish --clean
    ELSE IF [ "$mode" = "ci" ]
        SET buildArgs = --nightly --clean
    END
    IF [ "$mode" != "local" ]
        WITH DOCKER
            RUN --secret GITHUB_TOKEN echo $GITHUB_TOKEN | docker login ghcr.io -u NumaryBot --password-stdin
        END
    END
    WITH DOCKER
        RUN --mount=type=cache,id=gomod,target=${GOPATH}/pkg/mod \
            --mount=type=cache,id=gobuild,target=/root/.cache/go-build \
            --secret GORELEASER_KEY \
            --secret GITHUB_TOKEN \
            --secret SPEAKEASY_API_KEY \
            --secret FURY_TOKEN \
            goreleaser release -f .goreleaser.yml $buildArgs
    END

GRPC_GEN:
    FUNCTION
    RUN mkdir generated
    ARG --required protoName
    COPY $protoName .
    RUN protoc --go_out=generated --go_opt=paths=source_relative --go-grpc_out=generated --go-grpc_opt=paths=source_relative $protoName
    SAVE ARTIFACT generated

GO_TESTS:
    FUNCTION
    ARG GOPROXY
    ARG ADDITIONAL_ARGUMENTS
    RUN --mount type=cache,id=gopkgcache,target=${GOPATH}/pkg/mod \
        --mount type=cache,id=gobuildcache,target=/root/.cache/go-build \
        go test ./... -coverprofile=./cover.out -covermode=atomic -coverpkg=./... ${ADDITIONAL_ARGUMENTS}

GO_COVERAGE:
    FUNCTION
    ARG EARTHLY_BUILD_SHA
    RUN --secret CODECOV_TOKEN codecov -t $CODECOV_TOKEN -f cover.out -F ledger -C $EARTHLY_BUILD_SHA

GO_LINT:
    FUNCTION
    COPY (+sources/out --LOCATION=.golangci.yml) .golangci.yml
    RUN golangci-lint run --fix ./...
    ARG GOPROXY
    ARG GOCACHE=/go-cache
    ARG GOMODCACHE=/go-mod-cache
    ARG ADDITIONAL_ARGUMENTS
    RUN --mount=type=cache,id=golangci,target=/root/.cache/golangci-lint \
        golangci-lint run --fix ${ADDITIONAL_ARGUMENTS} ./...

GO_COMPILE:
    FUNCTION
    ARG GOPROXY
    ARG VERSION=latest
    ARG EARTHLY_BUILD_SHA
    LET GIT_PATH=$(head -1 go.mod | cut -d\\  -f2)
    ARG ADDITIONAL_ARGUMENTS
    RUN --mount type=cache,id=gopkgcache,target=${GOPATH}/pkg/mod \
        --mount type=cache,id=gobuildcache,target=/root/.cache/go-build \
        go build -o main \
        -ldflags="-X ${GIT_PATH}/cmd.Version=${VERSION} \
        -X ${GIT_PATH}/cmd.BuildDate=$(date +%s) \
        -X ${GIT_PATH}/cmd.Commit=${EARTHLY_BUILD_SHA}" ${ADDITIONAL_ARGUMENTS} ./
    SAVE ARTIFACT main

GO_INSTALL:
    FUNCTION
    ARG package
    ARG GOPROXY
    ARG ADDITIONAL_ARGUMENTS
    RUN --mount type=cache,id=gopkgcache,target=${GOPATH}/pkg/mod \
        --mount type=cache,id=gobuild,target=/root/.cache/go-build \
        go install ${ADDITIONAL_ARGUMENTS} ${package}

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
    ARG ADDITIONAL_ARGUMENTS
    RUN --mount type=cache,id=gopkgcache,target=${GOPATH}/pkg/mod \
        --mount type=cache,id=gobuildcache,target=/root/.cache/go-build \
        go mod tidy ${ADDITIONAL_ARGUMENTS}

GO_GENERATE:
    FUNCTION
    ARG GOPROXY
    ARG ADDITIONAL_ARGUMENTS
    RUN --mount type=cache,id=gopkgcache,target=${GOPATH}/pkg/mod \
        --mount type=cache,id=gobuildcache,target=/root/.cache/go-build \
        go generate ${ADDITIONAL_ARGUMENTS} ./...

HELM_VALIDATE:
    FUNCTION
    ARG additionalArgs
    RUN helm lint ./ --strict $additionalArgs
    RUN helm template ./ $additionalArgs

HELM_PUBLISH:
    FUNCTION
    ARG --required path
    WITH DOCKER
        RUN --secret GITHUB_TOKEN echo $GITHUB_TOKEN | docker login ghcr.io -u NumaryBot --password-stdin
    END
    WITH DOCKER
        RUN helm push ${path} oci://ghcr.io/formancehq/helm
    END

INCLURE_SDK_GO:
    FUNCTION
    ARG --required LOCATION
    COPY (releases+sdk-generate/go) ${LOCATION}

INCLUDE_GO_LIBS:
    FUNCTION
    ARG --required LOCATION
    COPY (libs+sources/src --LOCATION=libs/go-libs) ${LOCATION}

INCLUDE_CORE_LIBS:
    FUNCTION
    ARG --required LOCATION
    COPY (core+sources/src --LOCATION=libs/core) ${LOCATION}

GO_LINT:
    FUNCTION
    COPY (+sources/out --LOCATION=.golangci.yml) .golangci.yml
    ARG GOPROXY
    RUN --mount=type=cache,id=gomod,target=${GOPATH}/pkg/mod \
        --mount=type=cache,id=gobuild,target=/root/.cache/go-build \
        --mount=type=cache,id=golangci,target=/root/.cache/golangci-lint \
        golangci-lint run --fix ./...

GO_TIDY:
    FUNCTION
    ARG GOPROXY
    RUN --mount=type=cache,id=gomod,target=${GOPATH}/pkg/mod \
        --mount=type=cache,id=gobuild,target=/root/.cache/go-build \
        go mod tidy
    SAVE ARTIFACT go.* AS LOCAL .

SDK_GO:
    FUNCTION
    COPY (+sources-speakeasy/speakeasy) /bin/speakeasy
    COPY ./sdk/go.gen.yaml /src/sdks/gen.yaml
    WORKDIR /src/sdks
    RUN --secret SPEAKEASY_API_KEY speakeasy generate sdk -s ./../openapi.yaml -o ./ -l go
    SAVE ARTIFACT /src/releases/sdks/go AS LOCAL ./sdks/go


OPENAPI:
    FUNCTION
    openapi:
    FROM node:20-alpine
    RUN apk update && apk add yq
    RUN npm install -g openapi-merge-cli
    WORKDIR /src
    COPY --dir openapi openapi
    COPY (+sources/out/base.yaml --LOCATION=sdk/) ./openapi/base.yaml
    RUN openapi-merge-cli --config ./openapi/openapi-merge.json
    RUN yq -oy ./openapi.json > openapi.yaml
    SAVE ARTIFACT ./openapi.yaml AS LOCAL ./openapi.yaml