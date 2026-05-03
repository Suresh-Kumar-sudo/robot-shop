#---------- Build Stage ----------

FROM golang:1.23 AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o dispatch

#------------ Run Stage --------------

FROM alpine:3.19

WORKDIR /app

RUN apk add --no-cache ca-certificates

RUN adduser -D appuser

COPY --from=builder /app/dispatch .

USER appuser

EXPOSE 8080

CMD ["/app/dispatch"]