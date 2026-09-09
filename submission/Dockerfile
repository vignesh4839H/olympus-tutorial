FROM public.ecr.aws/d3j8x8q7/olympus-base-go:latest

WORKDIR /app

COPY . .

ENV CGO_ENABLED=0
ENV GOFLAGS="-mod=readonly"
# go.mod asks for a newer toolchain than some base images ship, so let the go
# command fetch it here while there is still a network.
ENV GOTOOLCHAIN=auto

# The runtime container is offline, so every module has to be in the module
# cache before the build finishes.
RUN go mod download

RUN go build ./...

CMD ["/bin/bash"]
