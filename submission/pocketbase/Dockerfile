FROM public.ecr.aws/d3j8x8q7/olympus-base-go:latest

WORKDIR /app

ENV CGO_ENABLED=0
ENV GOFLAGS="-mod=readonly"
ENV GOTOOLCHAIN=auto

RUN go version

COPY go.mod go.sum ./
RUN go mod download && go mod verify

COPY . .

RUN go build ./...

CMD ["/bin/bash"]
