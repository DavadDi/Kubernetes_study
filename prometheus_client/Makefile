docker:
	CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o client main.go
	docker build -t gitub.com/dwh0403/prometheus_client_test:latest .
