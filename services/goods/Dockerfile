### Build stage ###
FROM golang:1.25-alpine AS builder

WORKDIR /app

COPY go.mod  ./
RUN go mod download

COPY . .
RUN go build -o app ./...


### Final stage ###
FROM alpine:3.19

WORKDIR /app

COPY --from=builder /app/app .

EXPOSE 8080
CMD ["./app"]