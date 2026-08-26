# Troubleshooting pssid-daemon

## Basic troubleshooting

Basic troubleshooting can be found at this [page](https://umnet-perfsonar.github.io/pSSID-docs/#testing-daemon-troubleshooting)

Usually Layer 2 errors are the most common.
This is the basic solution for Layer 2 error

```bash
sudo systemctl stop pssid-daemon
sudo ip netns delete pssid_wlan0
sudo ip link set wlan0 down
sudo ip link set wlan0 up
sudo systemctl start pssid-daemon
```

## Error bonding wlan0

If you have this error it means that Netplan-Generated WPA Supplicant is still running.
Disable it and delete the socket. Then restart the pssid-daemon.

```bash
sudo systemctl stop netplan-wpa-wlan0.service
sudo rm -f /var/run/wpa_supplicant/wlan0
``
`
This error can occur after rebooting the node. So it's prefererably to monitor nodes for sometime after reboot.


## Finding error in test

If you have troubles with any test and do not receive metrics
use this command to check all runned test

```bash
sudo pscheduler schedule --filter-test test_type -PT2H
```

To seer the result of the test

```bash
sudo pscheduler result https://link-to-result
```

Usually the tests are never started because of the overlapping schedules or for any other reason
If you followed the instructions you shouldn't have problems with any test

## Logstash troubleshooting

To see the logstash logs

```bash
docker logs -f logstash --tail=100
```

Usually the problem is that metrics are truncated by filebeat or rsyslog. Check for JSON parsing errors and try to increase the size of messages in filebeat configuration in nodes.
This particullary happens with throughput and latency tests. Latency test produces a big histogram with all metrics, throughput test just gives the big output in slightly weird format which is still json, but very big.

---
