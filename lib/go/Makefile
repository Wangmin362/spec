all: build

########################################################################
##                             GOLANG                                 ##
########################################################################

# If GOPATH isn't defined then set its default location.
ifeq (,$(strip $(GOPATH)))
GOPATH := $(HOME)/go
else
# If GOPATH is already set then update GOPATH to be its own
# first element.
GOPATH := $(word 1,$(subst :, ,$(GOPATH)))
endif
export GOPATH


########################################################################
##                             PROTOC                                 ##
########################################################################

# Only set PROTOC_VER if it has an empty value.
ifeq (,$(strip $(PROTOC_VER)))
PROTOC_VER := 3.9.1
endif

PROTOC_OS := $(shell uname -s)
ifeq (Darwin,$(PROTOC_OS))
PROTOC_OS := osx
endif

PROTOC_ARCH := $(shell uname -m)
ifeq (i386,$(PROTOC_ARCH))
PROTOC_ARCH := x86_32
endif

PROTOC := ./protoc
PROTOC_ZIP := protoc-$(PROTOC_VER)-$(PROTOC_OS)-$(PROTOC_ARCH).zip
PROTOC_URL := https://github.com/google/protobuf/releases/download/v$(PROTOC_VER)/$(PROTOC_ZIP)
PROTOC_TMP_DIR := .protoc
PROTOC_TMP_BIN := $(PROTOC_TMP_DIR)/bin/protoc

$(PROTOC):
	-mkdir -p "$(PROTOC_TMP_DIR)" && \
	  curl -L $(PROTOC_URL) -o "$(PROTOC_TMP_DIR)/$(PROTOC_ZIP)" && \
	  unzip "$(PROTOC_TMP_DIR)/$(PROTOC_ZIP)" -d "$(PROTOC_TMP_DIR)" && \
	  chmod 0755 "$(PROTOC_TMP_BIN)" && \
	  cp -f "$(PROTOC_TMP_BIN)" "$@"
	stat "$@" > /dev/null 2>&1


########################################################################
##                          PROTOC-GEN-GO                             ##
########################################################################

# This is the recipe for getting and installing the go plug-in
# for protoc
PROTOC_GEN_GO_PKG := github.com/golang/protobuf/protoc-gen-go
PROTOC_GEN_GO := protoc-gen-go
$(PROTOC_GEN_GO): PROTOBUF_PKG := $(dir $(PROTOC_GEN_GO_PKG))
$(PROTOC_GEN_GO): PROTOBUF_VERSION := v1.3.2
$(PROTOC_GEN_GO):
	mkdir -p $(dir $(GOPATH)/src/$(PROTOBUF_PKG))
	test -d $(GOPATH)/src/$(PROTOBUF_PKG)/.git || git clone https://$(PROTOBUF_PKG) $(GOPATH)/src/$(PROTOBUF_PKG)
	(cd $(GOPATH)/src/$(PROTOBUF_PKG) && \
		(test "$$(git describe --tags | head -1)" = "$(PROTOBUF_VERSION)" || \
			(git fetch && git checkout tags/$(PROTOBUF_VERSION))))
	(cd $(GOPATH)/src/$(PROTOBUF_PKG) && go get -v -d $$(go list -f '{{ .ImportPath }}' ./...)) && \
	go build -o "$@" $(PROTOC_GEN_GO_PKG)


########################################################################
##                              PATH                                  ##
########################################################################

# Update PATH with the current directory. This enables the protoc
# binary to discover the protoc-gen-go binary, built inside this
# directory.
export PATH := $(shell pwd):$(PATH)


########################################################################
##                              BUILD                                 ##
########################################################################
CSI_PROTO := ../../csi.proto
CSI_PKG_ROOT := github.com/container-storage-interface/spec
CSI_PKG_SUB := $(shell cat $(CSI_PROTO) | sed -n -e 's/^package.\([^;]*\).v[0-9]\+;$$/\1/p'|tr '.' '/')
CSI_BUILD := $(CSI_PKG_SUB)/.build
CSI_GO := $(CSI_PKG_SUB)/csi.pb.go
CSI_GO_TMP := $(CSI_BUILD)/$(CSI_PKG_ROOT)/csi.pb.go

# This recipe generates the go language bindings to a temp area.
$(CSI_GO_TMP): HERE := $(shell pwd)
$(CSI_GO_TMP): PTYPES_PKG := github.com/golang/protobuf/ptypes
$(CSI_GO_TMP): GO_OUT := plugins=grpc
$(CSI_GO_TMP): GO_OUT := $(GO_OUT),Mgoogle/protobuf/descriptor.proto=github.com/golang/protobuf/protoc-gen-go/descriptor
$(CSI_GO_TMP): GO_OUT := $(GO_OUT),Mgoogle/protobuf/wrappers.proto=$(PTYPES_PKG)/wrappers
$(CSI_GO_TMP): GO_OUT := $(GO_OUT):"$(HERE)/$(CSI_BUILD)"
$(CSI_GO_TMP): INCLUDE := -I$(GOPATH)/src -I$(HERE)/$(PROTOC_TMP_DIR)/include
$(CSI_GO_TMP): $(CSI_PROTO) | $(PROTOC) $(PROTOC_GEN_GO)
	@mkdir -p "$(@D)"
	(cd "$(GOPATH)/src" && \
		$(HERE)/$(PROTOC) $(INCLUDE) --go_out=$(GO_OUT) "$(CSI_PKG_ROOT)/$(<F)")

# The temp language bindings are compared to the ones that are
# versioned. If they are different then it means the language
# bindings were not updated prior to being committed.
$(CSI_GO): $(CSI_GO_TMP)
ifeq (true,$(GITHUB_ACTIONS))
	diff "$@" "$?"
else
	@mkdir -p "$(@D)"
	diff "$@" "$?" > /dev/null 2>&1 || cp -f "$?" "$@"
endif


build: $(CSI_GO)

clean:
	go clean -i ./...
	rm -rf "$(CSI_GO)" "$(CSI_BUILD)"

clobber: clean
	rm -fr "$(PROTOC)" "$(PROTOC_GEN_GO)" "$(CSI_PKG_SUB)"

.PHONY: clean clobber
