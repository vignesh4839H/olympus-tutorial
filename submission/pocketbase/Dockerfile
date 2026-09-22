FROM public.ecr.aws/d3j8x8q7/olympus-base-go:latest

WORKDIR /app

ENV CGO_ENABLED=0
ENV GOFLAGS="-mod=readonly"
ENV GOTOOLCHAIN=auto

# the suite runs as an unprivileged uid that has no account and no home
# directory, so the module cache, the build cache and the downloaded toolchain
# have to live somewhere other than $HOME
ENV GOPATH=/go
ENV GOMODCACHE=/go/pkg/mod
ENV GOCACHE=/go/cache

RUN go version

COPY go.mod go.sum ./
RUN go mod download && go mod verify

COPY . .

RUN go build ./...

# several of the existing core tests create their data directory next to their
# own source file and the report is written into the working tree, so the tree
# and the caches both have to stay writable after the container drops out of
# root
RUN chmod -R a+rwX /app /go

CMD ["/bin/bash"]
