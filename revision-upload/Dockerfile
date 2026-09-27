FROM public.ecr.aws/d3j8x8q7/olympus-base-go:latest

WORKDIR /app

ENV CGO_ENABLED=0
ENV GOFLAGS="-mod=readonly"
ENV GOTOOLCHAIN=auto
ENV GOPATH=/go
ENV GOMODCACHE=/go/pkg/mod
ENV GOCACHE=/go/cache

RUN go version

COPY go.mod go.sum ./
RUN go mod download && go mod verify

COPY . .

RUN go build ./...

RUN chmod -R a+rwX /app /go

CMD ["/bin/bash"]
