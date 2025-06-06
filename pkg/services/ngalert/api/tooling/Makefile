.DEFAULT_GOAL := openapi

API_DIR = definitions
GO_PKG_FILES = $(shell find $(API_DIR) -name *.go -print)

PATH_DOWN = pkg/services/ngalert/api/tooling
PATH_UP = ../../../../..

GO ?= $(shell which go)
SWAGGER = GODEBUG=gotypesalias=0 $(GO) tool swagger

.DEFAULT_GOAL := all

GENERATED_GO_MATCHERS = ./go/*.go

SED_INPLACE := -i

ifneq ($(OS),Windows_NT)
	# If we're not using the GNU version of sed, we need to specify an argument to -i
	UNAME := $(shell uname)
	isGNU := $(shell sed --help 2>&1 | grep -c GNU)
	ifneq ($(isGNU),0)
		# Do nothing
	else ifeq ($(UNAME),Darwin)
		SED_INPLACE = -i ''
	endif
endif

spec.json spec-stable.json: $(GO_PKG_FILES)
	SWAGGER_GENERATE_EXTENSION=false $(SWAGGER) generate spec -m -w $(API_DIR) -o spec.json && SWAGGER_GENERATE_EXTENSION=false $(SWAGGER) generate spec -m --include-tag=stable -o spec-stable.json

post.json: spec.json
	go run cmd/clean-swagger/main.go -if $(<) -of $@

api.json: spec-stable.json
	go run cmd/clean-swagger/main.go -if $(<) -of $@

validate-stable: spec-stable.json
	$(SWAGGER) validate $(<)

validate: spec.json
	$(SWAGGER) validate $(<)

swagger-codegen-api:
	docker run --rm -v $$(pwd):/local --user $$(id -u):$$(id -g) parsertongue/swagger-codegen-cli:3.0.32 generate \
	-i /local/post.json \
	-l go-server \
	-Dapis \
	-o /local \
	--additional-properties packageName=api \
	-t /local/swagger-codegen/templates \
	# --import-mappings eval.RelativeTimeRange="github.com/grafana/grafana/pkg/services/ngalert/eval" \
	# --type-mappings RelativeTimeRange=eval.RelativeTimeRange

copy-files:
	ls -1 go | xargs -n 1 -I {} mv go/{} ../generated_base_{}

fix:
	sed $(SED_INPLACE) -e 's/apimodels\.\[\]PostableAlert/apimodels.PostableAlerts/' $(GENERATED_GO_MATCHERS)
	sed $(SED_INPLACE) -e 's/apimodels\.\[\]UpdateDashboardACLCommand/apimodels.Permissions/' $(GENERATED_GO_MATCHERS)
	sed $(SED_INPLACE) -e 's/apimodels\.\[\]PostableApiReceiver/apimodels.TestReceiversConfigParams/' $(GENERATED_GO_MATCHERS)
	goimports -w -v $(GENERATED_GO_MATCHERS)

clean: clean-go
	rm -f spec.json
	rm -f spec-stable.json

clean-go:
	rm -rf ./go

serve: post.json
	docker run --rm -p 80:8080 -v $$(pwd):/tmp -e SWAGGER_FILE=/tmp/$(<) swaggerapi/swagger-editor

serve-stable: api.json
	docker run --rm -p 80:8080 -v $$(pwd):/tmp -e SWAGGER_FILE=/tmp/$(<) swaggerapi/swagger-editor

gen: swagger-codegen-api fix copy-files clean-go

all: clean spec.json spec-stable.json post.json api.json gen
