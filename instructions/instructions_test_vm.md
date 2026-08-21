# Instructions for configuring virtual machine for testing

All you need for predictable and reliable testing is to setup Ubuntu Virtual Machine and make sure it is reachable from all your nodes.

## 1. Create Ubuntu Server VM

## 2. Add the PerfSONAR repository

```bash
curl -s https://downloads.perfsonar.net/install | sudo bash -s -- testpoint --auto-updates --tunings
sudo apt update
```

## 3. Install the PerfSONAR toolkit

```bash
sudo apt install perfsonar-testpoint
```

PerfSONAR/pScheduler installation pulls in the required services and tools, including the web server and test tools.

## 4. Verify pScheduler and the installed test tools

```bash
pscheduler --version
pscheduler tests
sudo systemctl --type=service | grep -i pscheduler
```

## 5. Check if iperf3 was downloaded with pscheduler

```bash
iperf3 --version
```

## 6. Start iperf3 as daemon

Create the iperf3 Service

```bash
sudo nano /etc/systemd/system/iperf3.service
```

Add:

```ini
[Unit]
Description=iperf3 server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/iperf3 -s
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Reload systemd

```bash
sudo systemctl daemon-reload
```

Enable and Start iperf3

```bash
sudo systemctl enable --now iperf3
```

Verify the Service

```bash
sudo systemctl status iperf3
```

The iperf3 server should be listening on the default port:

```text
5201
```

You can verify this with:

```bash
ss -lntp | grep 5201
```

## 7. Make sure you can connect to apache server with http://vm_ip

You can later use it for http tests
