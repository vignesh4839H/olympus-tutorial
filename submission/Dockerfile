FROM public.ecr.aws/d3j8x8q7/olympus-base-go:latest

WORKDIR /app

COPY . .

ENV CGO_ENABLED=0
# -mod=readonly keeps the build from editing go.mod or go.sum, so the committed
# versions are the ones that get used.
ENV GOFLAGS="-mod=readonly"
# go.mod asks for a newer toolchain than some base images ship. Pin it to the
# exact version go.mod names so the build is reproducible, and fetch it here
# while there is still a network.
ENV GOTOOLCHAIN=go1.25.0

# The runtime container is offline, so every module has to be in the module
# cache before the build finishes.
RUN go mod download

# Check every downloaded module against the hashes committed in go.sum. This
# fails the build if any dependency does not match the pinned version.
RUN go mod verify

RUN go build ./...

CMD ["/bin/bash"]
