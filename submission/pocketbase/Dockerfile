FROM public.ecr.aws/d3j8x8q7/olympus-base-go:latest

WORKDIR /app

COPY . .

ENV CGO_ENABLED=0
ENV GOFLAGS="-mod=readonly"
ENV GOTOOLCHAIN=auto

RUN go version

RUN test -f go.mod && test -f go.sum

RUN go mod download

RUN go mod verify

RUN go build ./...

CMD ["/bin/bash"]
