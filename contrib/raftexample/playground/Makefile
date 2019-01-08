all:
	go build go.etcd.io/etcd/contrib/raftexample

dump:
	go build go.etcd.io/etcd/tools/etcd-dump-logs

clean: 
	rm -rf raftexample*

test:
	for x in `seq 12`; do curl -X PUT http://127.0.0.1:12380/mykey -d val$$x; done
