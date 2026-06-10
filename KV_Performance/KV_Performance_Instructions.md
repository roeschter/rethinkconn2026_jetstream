# NATS JetStream KV — Performance Test Instructions

These steps stand up a single NATS server with JetStream, create two Key/Value
buckets, fill them with random 1 KB values, and run a random-read benchmark.

Run everything from inside the `KV_Performance/` directory so the JetStream
store lands in `./jetstream`.

**Prerequisites**

- `nats-server` — https://github.com/nats-io/nats-server
- `nats` CLI  (for the `nats bench kv` subcommands) —
  https://github.com/nats-io/natscli
- The `nats-server.conf` in this folder.

Verify versions:

```bash
nats-server --version
nats --version
```

---

## 1. Start the server (GOMEMLIMIT = 4 GB)

`GOMEMLIMIT` is a Go runtime soft memory limit. It uses IEC units, so 4 GB is
written `4GiB`.

```bash
cd KV_Performance
GOMEMLIMIT=4GiB nats-server -c nats-server.conf
```

Leave this running in its own terminal. JetStream data is written to
`./jetstream` (created automatically). Monitoring is at http://localhost:8222.

> The `nats-server.conf` sets `no_auth_user: app`, so the `nats` CLI commands
> below connect to the **APP** account with no credentials. If you prefer to be
> explicit, add `--user app --password app` to any command.

---

## 2. Create the two KV buckets

In a second terminal:

```bash
nats kv add PERF01 --history=1 --storage=file --replicas=1
```

Confirm they exist:

```bash
nats kv ls
```

---

## 3. Fill the buckets with random 1 KB values

`nats bench kv put` writes `--msgs` keys of `--size` bytes into the bucket,
using multiple concurrent clients. `--size=1024` = 1 KB values.

**PERF01 — 1,000,000 keys:**

First we fill all keys with values then we randomize it a bit
```bash
nats bench kv put --bucket=PERF01 --msgs=100000 --size=1024 --randomize=0 --clients=5
nats bench kv put --bucket=PERF01 --msgs=100000 --size=1024 --clients=5 --randomize=100000
```
This was slow on the randomization run? Why?
Due to history=1 a lot of keys needed to be deleted. In a replica=1 stream this is done by flushing the buffer every time (to minimized data loss in case of a crash). FOr replicated streams the flushing is asynchronous. 

We can force this for streams with "--persist_mode=async". But for KV this flag is (not yet) supported in the CLI, so how do we create a KV with this flag? 

```bash
nats kv add PERF01 --history=1 --storage=file --replicas=1 --trace
```
Gives us the command sent:

```bash
$JS.API.STREAM.CREATE.KV_PERF01

{"name":"KV_PERF01","subjects":["$KV.PERF01.\u003e"],"retention":"limits","max_consumers":-1,"max_msgs":-1,"max_bytes":-1,"discard":"new","max_age":0,"max_msgs_per_subject":1,"max_msg_size":-1,"storage":"file","num_replicas":1,"duplicate_window":120000000000,"deny_delete":true,"allow_rollup_hdrs":true,"compression":"none","allow_direct":true,"mirror_direct":false,"consumer_limits":{}}
```

```bash
nats kv del PERF01

nats request '$JS.API.STREAM.CREATE.KV_PERF01' '{"name":"KV_PERF01","subjects":["$KV.PERF01.\u003e"],"retention":"limits","max_consumers":-1,"max_msgs":-1,"max_bytes":-1,"discard":"new","max_age":0,"max_msgs_per_subject":1,"max_msg_size":-1,"storage":"file","num_replicas":1,"duplicate_window":120000000000,"deny_delete":true,"allow_rollup_hdrs":true,"compression":"none","allow_direct":true,"mirror_direct":false,"consumer_limits":{}, "persist_mode":"async"}'
``


**PERF02 — 10,000,000 keys:**

```bash
nats request '$JS.API.STREAM.CREATE.KV_PERF02' '{"name":"KV_PERF02","subjects":["$KV.PERF02.\u003e"],"retention":"limits","max_consumers":-1,"max_msgs":-1,"max_bytes":-1,"discard":"new","max_age":0,"max_msgs_per_subject":1,"max_msg_size":-1,"storage":"file","num_replicas":1,"duplicate_window":120000000000,"deny_delete":true,"allow_rollup_hdrs":true,"compression":"none","allow_direct":true,"mirror_direct":false,"consumer_limits":{}, "persist_mode":"async"}'
``


```bash
nats bench kv put --bucket=PERF02 --msgs=2000000 --size=1024 --clients=5 --randomize=0
nats bench kv put --bucket=PERF02 --msgs=100000 --size=1024 --clients=5 --randomize=2000000
```


---

## 4. Random reads — 100,000 reads over PERF01 (1,000,000 keys)

`nats bench kv get` reads `--msgs` keys chosen at random from the bucket and
reports throughput / latency.

```bash
nats bench kv get --bucket=PERF01 --msgs=100000 --clients=10 --randomize=1000
nats bench kv get --bucket=PERF01 --msgs=100000 --clients=10 --randomize=10000
nats bench kv get --bucket=PERF01 --msgs=100000 --clients=10 --randomize=100000
```

For the larger bucket:

```bash
nats bench kv get --bucket=PERF02 --msgs=100000 --clients=10 --randomize=1000
nats bench kv get --bucket=PERF02 --msgs=100000 --clients=10 --randomize=10000
nats bench kv get --bucket=PERF02 --msgs=100000 --clients=10 --randomize=100000
nats bench kv get --bucket=PERF02 --msgs=100000 --clients=10 --randomize=2000000
```

---

## Notes

