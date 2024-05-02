TMPPATH=./temp
LOGNAME="log"
LOGPATH="${TMPPATH}/${LOGNAME}"

define ENSURE_DIR
	@mkdir -p $(TMPPATH)
endef

.PHONY: test check gen clean

test:
	$(ENSURE_DIR)
	go test -v -race raft/raft |tee ${LOGPATH}

gen:
	go run tool/main.go < ${LOGPATH}

check:
	go vet ./raft/...
	staticcheck ./raft/...

clean:
	rm -rf temp