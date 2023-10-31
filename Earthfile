VERSION --pass-args --global-cache --arg-scope-and-set 0.7

base-image:
    FROM alpine:3.18
    #CACHE --id=gopkgcache /go/pkg/mod
    #CACHE --id=gobuildcache /root/.cache/go-build

goreleaser:
    FROM goreleaser/goreleaser-pro:v1.21.2-pro
    SAVE ARTIFACT /usr/bin/goreleaser

builder-image:
    FROM +base-image
    RUN apk update && apk add go git curl make pkgconfig bash docker jq
    ENV GOPATH /go
    ARG GOCACHE=/go-cache
    ARG GOMODCACHE=/go-mod-cache
    ENV PATH $PATH:$GOPATH/bin
    ENV CGO_ENABLED=0
    RUN curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin v1.53.3
    COPY (+goreleaser/*) /usr/bin/goreleaser
    RUN curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin v0.94.0

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
    ENV tldCertificatesChartBundled=$(cat tld-certificates-0.6.0.tgz | base64 -w 0)

    RUN echo "user: $user" > /tmp/values.yaml
    RUN --secret tld echo "tld: $tld" >> /tmp/values.yaml

    LET values=$(cat /tmp/values.yaml)
    RUN --secret tld helm upgrade --install $user vcluster \
        --repo https://charts.loft.sh \
        --namespace vcluster-$user \
        --set syncer.extraArgs[0]="--tls-san=kube.$user.$tld" \
        --set syncer.extraArgs[1]="--out-kube-config-server=https://kube.$user.$tld" \
        --set init.helm[0].bundle=$tldCertificatesChartBundled \
        --set init.helm[0].chart.name=tld-certificates \
        --set init.helm[0].chart.version=0.6.0 \
        --set init.helm[0].values="$values" \
        --set init.helm[0].release.name=tld-certificates \
        --set init.helm[0].release.namespace=formance \
        --values values.yaml \
        --repository-config=''
    RUN while ! kubectl -n vcluster-$user get secrets/vc-$user -o jsonpath='{.data.config}'; do sleep 1s; done
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

GO_TESTS:
    COMMAND
    ARG GOPROXY
    RUN --mount type=cache,id=gopkgcache,target=${GOPATH}/pkg/mod \
        --mount type=cache,id=gobuildcache,target=/root/.cache/go-build \
        go test ./...
    CACHE --sharing=shared --id=go_cache $GOCACHE
    CACHE --sharing=shared --id=go_mod_cache $GOMODCACHE

GO_LINT:
    COMMAND
    COPY (+sources/out --LOCATION=.golangci.yml) .golangci.yml
    ARG GOPROXY
    ARG GOCACHE=/go-cache
    ARG GOMODCACHE=/go-mod-cache
    RUN --mount=type=cache,id=golangci,target=/root/.cache/golangci-lint \
        golangci-lint run --fix ./...
    CACHE --sharing=shared --id=go_cache $GOCACHE
    CACHE --sharing=shared --id=go_mod_cache $GOMODCACHE

GO_COMPILE:
    COMMAND
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
    COMMAND
    ARG package
    ARG GOPROXY
    RUN --mount type=cache,id=gopkgcache,target=${GOPATH}/pkg/mod \
        --mount type=cache,id=gobuild,target=/root/.cache/go-build \
        go install ${package}

SAVE_IMAGE:
    COMMAND
    ARG TAG=latest
    ARG --required COMPONENT
    ARG REPOSITORY=ghcr.io
    ENV OTEL_SERVICE_NAME ${COMPONENT}
    # todo(gfyrag): make insecure configurable
    SAVE IMAGE --push --insecure ${REPOSITORY}/formancehq/${COMPONENT}:${TAG}

GO_MOD_TIDY:
    COMMAND
    ARG GOPROXY
    RUN --mount type=cache,id=gopkgcache,target=${GOPATH}/pkg/mod \
        --mount type=cache,id=gobuildcache,target=/root/.cache/go-build \
        go mod tidy
    CACHE --sharing=shared --id=go_cache $GOCACHE
    CACHE --sharing=shared --id=go_mod_cache $GOMODCACHE

GO_INSTALL:
    COMMAND
    DO --pass-args GO_INSTALL

GO_GENERATE:
    COMMAND
    ARG GOPROXY
    RUN --mount type=cache,id=gopkgcache,target=${GOPATH}/pkg/mod \
        --mount type=cache,id=gobuildcache,target=/root/.cache/go-build \
        go generate ./...
