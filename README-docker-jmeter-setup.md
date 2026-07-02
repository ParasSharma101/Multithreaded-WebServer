# Docker + JMeter setup for Multithreaded-WebServer

## Before you use this

1. Confirm the actual port your servers listen on (this setup assumes **8010**
   based on a similarly-structured reference project — check your `Server.java`
   files and fix `EXPOSE` in each Dockerfile + the compose port mappings if different).
2. Confirm the protocol: `telnet localhost <port>` or `nc localhost <port>` and
   send a request manually. If you get back real HTTP (`HTTP/1.1 200 OK`), use
   JMeter's **HTTP Request** sampler when building your test plan. If it's a
   custom text protocol, use the **TCP Sampler** instead.

## File placement

Copy these into your actual repo like so:

```
Multithreaded-WebServer/
├── docker-compose.yml          <- from this package
├── SingleThreaded/
│   ├── Server.java             <- already yours
│   ├── Client.java             <- already yours
│   └── Dockerfile              <- from this package
├── MultiThreaded/
│   ├── Server.java
│   └── Dockerfile              <- from this package
├── ThreadPool/
│   ├── Server.java
│   └── Dockerfile              <- from this package
└── testplan/
    └── plan.jmx                <- you'll build this in JMeter GUI, see below
```

Note: adjust the Dockerfile's `COPY Server.java .` line if your server files
have different names, or if a variant needs more than one .java file compiled.

## Install and run

1. Install Docker Desktop, verify with `docker --version`
2. From the repo root: `docker-compose up --build`
   This builds and starts all three servers on host ports 8081/8082/8083.
3. Sanity check each one from your host machine:
   `curl localhost:8081` (or telnet/nc if not real HTTP)

## Building the JMeter test plan (do this once, reuse for all 3 servers)

Install JMeter locally just to build the plan (you don't need it installed to
*run* tests -- that happens in the container):

1. Open JMeter GUI → right-click Test Plan → Add → Threads → Thread Group
   - Set Number of Threads (concurrent users), Ramp-Up Period, Loop Count
2. Right-click Thread Group → Add → Sampler → **TCP Sampler**
   (or **HTTP Request** if you confirmed real HTTP above)
   - Server Name: `${target_host}`  (so it's overridable from the command line)
   - Port: `${target_port}`
   - For TCP Sampler: set your request text/bytes, and check
     "Re-use connection" off for a fair test of connection overhead
3. Right-click Thread Group → Add → Listener → **Summary Report** and
   **Simple Data Writer** (so results save to a file when run headless)
4. Save as `testplan/plan.jmx`

## Running the load test against each server

```
docker-compose run --rm jmeter -n -t /test/plan.jmx \
  -Jtarget_host=single-threaded -Jtarget_port=8010 \
  -l /test/results-single.jtl -e -o /test/report-single

docker-compose run --rm jmeter -n -t /test/plan.jmx \
  -Jtarget_host=multi-threaded -Jtarget_port=8010 \
  -l /test/results-multi.jtl -e -o /test/report-multi

docker-compose run --rm jmeter -n -t /test/plan.jmx \
  -Jtarget_host=thread-pool -Jtarget_port=8010 \
  -l /test/results-pool.jtl -e -o /test/report-pool
```

Note the target_host values match the service names in docker-compose.yml,
and target_port is the CONTAINER port (8010), not the host-mapped port --
JMeter is running inside the same Docker network as the servers here.

Repeat each run at increasing Thread Group concurrency (e.g. 50 / 200 / 1000
threads) to get your low/medium/high load comparison points. Rename the
output files each time so you don't overwrite results.

## Reading results

Each run produces an HTML report in `testplan/report-*/index.html` on your
host machine (via the mounted volume) -- open it in a browser for latency
graphs, throughput, and error rate, no manual parsing needed.
