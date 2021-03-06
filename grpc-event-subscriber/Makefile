RELEASE_VERSION  =v0.10.0
SERVICE_NAME    ?=grpc-event-subscriber
DOCKER_USERNAME ?=$(DOCKER_USER)

.PHONY: tidy test debug build run jsonevent xmlevent binevent image lint clean tag
all: help

tidy: ## Updates the go modules and vendors all dependencies 
	go mod tidy
	go mod vendor

test: tidy ## Tests the entire project 
	go test -count=1 -race ./...

debug: tidy ## Runs uncompiled code in Dapr
	dapr run \
        --app-id $(SERVICE_NAME) \
        --app-port 60002 \
        --app-protocol grpc \
        --dapr-http-port 3500 \
        --components-path ./config \
        go run main.go

build: tidy ## Builds local release binary
	CGO_ENABLED=0 go build -a -tags netgo -mod vendor -o bin/$(SERVICE_NAME) .

run: build ## Builds binary and runs it in Dapr
	dapr run \
        --app-id $(SERVICE_NAME) \
        --app-port 60002 \
        --app-protocol grpc \
        --dapr-http-port 3500 \
        --components-path ./config \
        bin/$(SERVICE_NAME) 

jsonevent: ## Publishes sample JSON message to Dapr pubsub API 
	curl -d '{ "from": "John", "to": "Lary", "message": "hi" }' \
     -H "Content-type: application/json" \
     "http://localhost:3500/v1.0/publish/grpc-events/messages"

xmlevent: ## Publishes sample XML message to Dapr pubsub API 
	curl -d '<message><from>John</from><to>Lary</to></message>' \
     -H "Content-type: application/xml" \
     "http://localhost:3500/v1.0/publish/grpc-events/messages"

binevent: ## Publishes sample binary message to Dapr pubsub API 
	curl -d '0x18, 0x2d, 0x44, 0x54, 0xfb, 0x21, 0x09, 0x40' \
     -H "Content-type: application/octet-stream" \
     "http://localhost:3500/v1.0/publish/grpc-events/messages"

image: tidy ## Builds and publish docker image 
	docker build -t "$(DOCKER_USERNAME)/$(SERVICE_NAME):$(RELEASE_VERSION)" .
	docker push "$(DOCKER_USERNAME)/$(SERVICE_NAME):$(RELEASE_VERSION)"

deploy: ## Deploys prebuild image to k8s using currently selected context 
	kubectl apply -f k8s/component.yaml
	kubectl apply -f k8s/deployment.yaml
	kubectl rollout restart deployment/grpc-event-subscriber
	kubectl rollout restart deployment/nginx-ingress-nginx-controller
	kubectl rollout status deployment/nginx-ingress-nginx-controller

event: ## Publishes sample JSON message to Dapr pubsub API 
	$(eval API_TOKEN=$(shell kubectl get secret dapr-api-token -o jsonpath="{.data.token}" | base64 --decode))
	curl -d '{ "from": "John", "to": "Lary", "message": "hi" }' \
     -H "Content-type: application/json" \
     -H "dapr-api-token: $(API_TOKEN)" \
     "https://api.cloudylabs.dev/v1.0/publish/grpc-events/messages"

lint: ## Lints the entire project 
	golangci-lint run --timeout=3m

tag: ## Creates release tag 
	git tag $(RELEASE_VERSION)
	git push origin $(RELEASE_VERSION)

clean: ## Cleans up generated files 
	go clean
	rm -fr ./bin
	rm -fr ./vendor

help: ## Display available commands
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk \
		'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'
