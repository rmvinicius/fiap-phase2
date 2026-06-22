# STAGE BUILD
FROM golang:1.21-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
COPY *.go ./
RUN go mod download && go mod tidy
RUN CGO_ENABLED=0 GOOS=linux go build -o /evaluation-service .

# STAGE RUN
FROM alpine:3.20

RUN apk --no-cache add ca-certificates curl

RUN adduser -D -u 10001 evaluation-service

WORKDIR /app

COPY --from=builder /evaluation-service .

USER evaluation-service

EXPOSE 8004

CMD ["./evaluation-service"]