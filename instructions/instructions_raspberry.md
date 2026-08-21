# Raspberry Pi Ubuntu Server Deployment Checklist

## 1. Flash Ubuntu Server

Flash **Ubuntu Server 24.04** onto the Raspberry Pi.

---

## 2. Configure Ubuntu Sources

Add `noble-updates` to:

```bash
/etc/apt/sources.list.d/ubuntu.sources
```

---

## 3. Update the Operating System

```bash
sudo apt update
sudo apt upgrade -y
```

---

## 4. Verify Wi-Fi

Check that the Wi-Fi interface exists:

```bash
ip link
```

The output should contain:

```text
wlan0
```

---

## 5. Configure DNS

### 5.1 Disable `systemd-resolved`

```bash
sudo systemctl disable --now systemd-resolved
```

### 5.2 Remove the Existing `resolv.conf` Symlink

```bash
sudo rm /etc/resolv.conf
```

### 5.3 Create a Static Resolver Configuration

```bash
echo "nameserver 172.30.0.5" | sudo tee /etc/resolv.conf
```

### 5.4 Verify the Configuration

```bash
cat /etc/resolv.conf
```

Expected output:

```text
nameserver 172.30.0.5
```

---

## 6. Check Network Connectivity

Verify that the Raspberry Pi has a working network connection before continuing.

```bash
ip addr
ip route
```

Optionally test connectivity:

```bash
ping -c 3 172.30.0.5
ping -c 3 google.com
```

---

## 7. Stop the Netplan-Generated WPA Supplicant

Stop the service managing the Wi-Fi interface:

```bash
sudo systemctl stop netplan-wpa-wlan0.service
```

---

## 8. Remove the Stale WPA Supplicant Socket

Remove the existing control socket for `wlan0`:

```bash
sudo rm -f /var/run/wpa_supplicant/wlan0
```

---

## 9. Install the pSSID Daemon

Install and configure the **pSSID daemon** according to the deployment requirements.

After installation, verify that the service is running correctly.

---

## 10. Increase rsyslog Maximum Message Size

Increase the maximum message size accepted by rsyslog to prevent large pSSID log messages from being truncated or rejected.

### 10.1 Create the Configuration File

```bash
sudo nano /etc/rsyslog.d/01-max-message-size.conf
```

Add:

```text
global(maxMessageSize="64k")
```

### 10.2 Validate the rsyslog Configuration

```bash
sudo rsyslogd -N1
```

The configuration should pass validation without errors.

### 10.3 Restart rsyslog

```bash
sudo systemctl restart rsyslog
```

---

## 11. Configure iperf3 for Throughput Tests

If the Raspberry Pi will act as the **destination node** for a throughput test, configure an iperf3 server as a systemd service.

### 11.1 Create the iperf3 Service

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

### 11.2 Reload systemd

```bash
sudo systemctl daemon-reload
```

### 11.3 Enable and Start iperf3

```bash
sudo systemctl enable --now iperf3
```

### 11.4 Verify the Service

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

---
