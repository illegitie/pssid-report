# pSSID DNS Troubleshooting Quick Guide


DNS fails?
    |
    v
Can namespace ping DNS server?
    |
    +-- No
    |      |
    |      +--> Check DHCP/routes
    |
    +-- Yes
           |
           v
     Check /etc/resolv.conf
           |
           +-- nameserver 127.0.0.53
           |        |
           |        +--> systemd-resolved problem
           |
           +-- nameserver 172.30.0.5
                    |
                    +--> DNS should work

1. Check if the namespace has network connectivity by running `sudo ip netns exec pssid_wlan0 ping -c 3 172.30.0.5`.

2. Verify the namespace routing table with `sudo ip netns exec pssid_wlan0 ip route` and make sure a default route exists.

3. Check the current DNS configuration using `sudo ip netns exec pssid_wlan0 cat /etc/resolv.conf`.

4. If the file contains `nameserver 127.0.0.53`, the namespace is using the systemd-resolved stub and DNS will fail.

5. Disable systemd-resolved with `sudo systemctl disable --now systemd-resolved`.

6. Remove the `/etc/resolv.conf` symlink using `sudo rm /etc/resolv.conf`.

7. Create a new `/etc/resolv.conf` file containing the real DNS server: `nameserver 172.30.0.5`.

8. Set normal permissions with `sudo chmod 644 /etc/resolv.conf` and do not make the file immutable with `chattr +i`.

9. Restart the pSSID daemon using `sudo systemctl restart pssid-daemon` to recreate the namespace with the correct DNS configuration.

10. Verify the fix with `sudo ip netns exec pssid_wlan0 nslookup google.com` and confirm that DNS resolution works.