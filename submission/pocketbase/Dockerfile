FROM public.ecr.aws/d3j8x8q7/olympus-base-go:latest

WORKDIR /app

COPY . .

# The project uses a pure Go SQLite driver, so nothing here needs a C toolchain.
ENV CGO_ENABLED=0
# -mod=readonly keeps the build from editing go.mod or go.sum, so the versions
# committed in the repository are the ones that get used.
ENV GOFLAGS="-mod=readonly"
# Pin the toolchain to the exact version go.mod names, so the build does not
# drift onto a newer Go release later on.
ENV GOTOOLCHAIN=go1.27.0

# Download and activate that pinned toolchain now, while there is still a
# network, and print the version actually in use so the build fails here rather
# than later if it could not be obtained.
RUN go version

# The runtime container is offline, so every module has to be in the module
# cache before the build finishes.
RUN go mod download

# Check every downloaded module against the hashes in the committed go.sum.
# This fails the build if any dependency does not match its pinned version.
RUN go mod verify

RUN go build ./...

CMD ["/bin/bash"]
