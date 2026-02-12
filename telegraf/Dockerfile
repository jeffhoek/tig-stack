FROM golang:1.25.5 AS builder

WORKDIR /src
COPY . .

RUN COMMIT=$(git rev-parse --short=8 HEAD) && \
    BRANCH=$(git rev-parse --abbrev-ref HEAD) && \
    VERSION=$(cat build_version.txt) && \
    CGO_ENABLED=0 go build -o /telegraf \
    -ldflags "-s -w \
    -X github.com/influxdata/telegraf/internal.Version=${VERSION}-${COMMIT} \
    -X github.com/influxdata/telegraf/internal.Branch=${BRANCH} \
    -X github.com/influxdata/telegraf/internal.Commit=${COMMIT}" \
    ./cmd/telegraf

FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates && \
    rm -rf /var/lib/apt/lists/* && \
    groupadd -r telegraf && useradd -r -g telegraf telegraf

COPY --from=builder /telegraf /usr/bin/telegraf

USER telegraf
ENTRYPOINT ["/usr/bin/telegraf"]
