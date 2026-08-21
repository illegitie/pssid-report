# Instructions on test creation

## Test types

There 9 main types of tests: http, dns, rtt, trace, latency, throughput, noop, idle, mtu
6 of them are important, others (noop,idle,mtu) do not provide helpful metrics

### 1. HTTP test

Http test measures http response time
Http test has two fields to specify: destination url and timeout.
For url you can choose any website you want but for predictability is better to use a website hosted on one of your machines.
Timeout specifies how long the HTTP test waits for the HTTP to complete before it is considered as failed. I use PT10S as timeout

### 2. DNS test

DNS test measures DNS transaction time.
It has three fields: nameserver, record, query.
Nameserver is usually an IP of your dns server.
Record is a format of a response (i use 'a')
Query is the website you want to find with dns (i use google.com)

### 3. Rtt test

Rtt test measures the round trip time and related statisctics between hosts.
Returns rtt ms and number of lost packets.
Rtt is similar to http test in terms of fields but instead of timeout it has length (normally 512).

### 4. Trace test

Trace the path between IP hosts.
Return trace ms

### 5. Latency test

It measures how long it takes for a packet to travel from the source node to the destination and back.
The result is usually reported as RTT (Round-Trip Time) in milliseconds.
For latency you only need to specify destination

### 6. Throughput test

The throughput test tells you how much data the network can transfer between two endpoints over a period of time.
Usually reported as bits per second.
The only thing to specify is destination.

---
