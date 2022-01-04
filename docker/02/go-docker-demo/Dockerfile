FROM golang:latest

WORKDIR /app/demo
COPY . .

RUN go build main.go

EXPOSE 8080
ENTRYPOINT ["./main"]
