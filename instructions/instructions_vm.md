# Instructions for setupping main server for managing pssid-nodes

##

### 1. Download Ansible

### 2. Download Docker

### 3. Set up [pssid-gui](https://github.com/UMNET-perfSONAR/pssid-gui2)

### 4. Set up the data analytics stack

You can use this github repo for configuring data pipeline or you can my version.  
[git-hub page](https://github.com/UMNET-perfSONAR/pssid-data-pipeline)

### 4.1. create directory monitoring stack

```bash
mkdir montoring-stack
```

### 4.2. create docker-compose.yml

Create token for grafana-renderer

```bash
openssl rand -hex 32
```

Insert it inside grafana-render AUTH_TOKEn and grafana GF_RENDERING_RENDERER_TOKEN

```yml
        services:
    opensearch:
        image: opensearchproject/opensearch:2
        container_name: opensearch
        environment:
        - discovery.type=single-node
        - OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g
        - DISABLE_SECURITY_PLUGIN=true
        ulimits:
        memlock:
            soft: -1
            hard: -1
        ports:
        - "9200:9200"
        volumes:
        - ./opensearch/data:/usr/share/opensearch/data
        restart: unless-stopped


    logstash:
        image: opensearchproject/logstash-oss-with-opensearch-output-plugin:latest
        container_name: logstash
        depends_on:
        - opensearch
        ports:
        - "9400:9400"
        volumes:
        - ./logstash/pipeline:/usr/share/logstash/pipeline
        restart: unless-stopped

    renderer:
        image: grafana/grafana-image-renderer:latest
        container_name: grafana-renderer
        environment:
                AUTH_TOKEN: TOKEN
        restart: unless-stopped
    grafana:
        image: grafana/grafana:latest
        container_name: grafana
        depends_on:
        - opensearch
        - renderer
        ports:
        - "3000:3000"
        environment:
                GF_RENDERING_SERVER_URL: http://renderer:8081/render
                GF_RENDERING_CALLBACK_URL: http://grafana:3000/
                GF_RENDERING_RENDERER_TOKEN: TOKEN
        volumes:
        - grafana-data:/var/lib/grafana
        restart: unless-stopped

    volumes:
    grafana-data:
```

### 4.3 Create directory Logstash and dir pipeline inside with logstash.conf

Put this as a config or write your own!

```ini
input{
        beats{
                port => 9400
        }
}

filter {
        #Extract Json after "pssid: "
        grok{
                match =>{
                        "message" => "%{TIMESTAMP_ISO8601:log_time} %{DATA:device} pssid: %{GREEDYDATA:json_data}"
                }
        }
        if [json_data]{
                mutate{
                        strip => ["json_data"]
                }
                if [json_data] =~ /^\{/ {

                        json{
                                source => "json_data"
                                target => "pssid"
                        }

                        mutate{
                                add_field => {
                                        "test_type" => "%{[pssid][test][type]}"
                                        "ssid" => "%{[pssid][reference][SSID]}"
                                        "interface" => "%{[pssid][reference][interface]}"
                                }
                        }
                        if [test_type] == "http" {
                                mutate {
                                        add_field =>{
                                                "http_ms" => "%{[pssid][result][time]}"
                                        }
                                }

                                ruby {
                                        code => '
                                        value = event.get("http_ms")
                                        if value
                                                seconds= value.delete_prefix("PT").delete_suffix("S").to_f
                                                event.set("http_ms", seconds * 1000)
                                        else
                                                event.remove("http_ms")
                                        end
                                        '
                                }

                                mutate {
                                        convert => {
                                                "http_ms" => "float"
                                        }
                                }
                        }
                        if [test_type] == "dns" {
                                ruby {
                                        code => '
                                                value = event.get("[pssid][result][time]")
                                                if value
                                                        seconds = value.gsub("PT","").gsub("S","").to_f
                                                        event.set("dns_ms", seconds * 1000)
                                                end
                                        '
                                }
                                mutate {
                                        convert => {
                                                "dns_ms" => "float"
                                        }
                                }
                        }
                        if [test_type] == "mtu" {
                                mutate{
                                        add_field => {
                                                "mtu" => "%{[pssid][result][mtu]}"
                                        }
                                        convert => {
                                                "mtu" => "integer"
                                                }
                                }
                        }
                        if [test_type] == "rtt" {

                                 mutate {
                                        add_field => {
                                                "rtt_ms" => "%{[pssid][result][mean]}"
                                                "packet_loss" => "%{[pssid][result][loss]}"
                                        }
                                }
                                ruby{
                                        code=> '
                                        value = event.get("rtt_ms")
                                        if value && value.start_with?("PT")
                                                seconds = value.delete_prefix("PT").delete_suffix("S").to_f
                                                event.set("rtt_ms", seconds * 1000)
                                        end
                                        '
                                }
                                mutate {
                                        convert=> {
                                                "rtt_ms" => "float"
                                                "packet_loss" => "float"
                                        }
                                }
                        }
                        if [test_type] == "latency" {
                                ruby {
                                        code => '
                                        hist = event.get("[pssid][result][histogram-latency]")

                                        if hist && hist.is_a?(Hash)
                                                latency_histogram = []
                                                values = []

                                                hist.each do |latency, count|
                                                        latency = latency.to_f
                                                        count = count.to_i

                                                        latency_histogram << {
                                                                "latency_ms" => latency,
                                                                "count" => count
                                                        }

                                                        count.times do
                                                                values << latency
                                                        end
                                                end

                                                event.set("latency_histogram", latency_histogram)

                                                unless values.empty?
                                                        event.set("latency_min_ms", values.min)
                                                        event.set("latency_max_ms", values.max)
                                                        event.set("latency_avg_ms", values.sum / values.length)
                                                end
                                        end
                                        '
                                }
                                mutate {
                                        remove_field => [
                                                "[pssid][run][result-full]",
                                                "[pssid][run][result-merged]",
                                                "[pssid][run][limit-diags]",
                                                "[pssid][task][detail][diags]",
                                                "[pssid][result][histogram-latency]",
                                                "[pssid][result][histogram-ttl]",
                                                "[pssid][run][result][histogram-latency]",
                                                "[pssid][run][result][histogram-ttl]"
                                        ]
                                }
                        }
                        if [test_type] == "throughput" {
                                mutate {
                                        add_field =>{
                                                "throughput_source" => "%{[pssid][participants][0]}"
                                                "throughput_destination" => "%{[pssid][participants][1]}"
                                        }
                                }

                                ruby {
                                        code => '
                                                require "json"
                                                diags = event.get("[pssid][run][result-full][0][diags]")
                                                if diags
                                                        start = diags.index("{")
                                                        if start
                                                        iperf = diags[start..-1]
                                                        begin
                                                                data = JSON.parse(iperf)

                                                                sent = data.dig("end","sum_sent","bits_per_second")
                                                                recv = data.dig("end","sum_received","bits_per_second")
                                                                retrans=data.dig("end","sum_sent","retransmits")
                                                                event.set("throughput_sent_mbps", sent.to_f / 1000000) if sent
                                                                event.set("throughput_received_mbps", recv.to_f / 1000000) if recv
                                                                event.set("throughput_retransmits",data["end"]["sum_sent"]["retransmits"]) if retrans

                                                        rescue => e
                                                                event.tag("iperf_json_failed")
                                                        end
                                                end
                                        end
                                        '
                                }

                                mutate {
                                        convert => {
                                                "throughput_sent_mbps" => "float"
                                                "throughput_received_mbps" => "float"
                                                "throughput_retransmits" => "integer"
                                        }
                                }
                        }
                        if [test_type] == "trace" {
                                         mutate {
                                                add_field => {
                                                        "trace_rtt_ms" => "%{[pssid][result][paths][0][0][rtt]}"
                                                }
                                        }
                                        ruby{
                                                code=> '
                                                value = event.get("trace_rtt_ms")
                                                if value && value.start_with?("PT")
                                                        seconds = value.delete_prefix("PT").delete_suffix("S").to_f
                                                        event.set("trace_rtt_ms", seconds * 1000)
                                                end
                                                '
                                        }
                                mutate {
                                        convert=> {
                                                "trace_rtt_ms" => "float"
                                        }
                                }
                        }
                        mutate {
                                remove_field=> [
                                        "[pssid][run]",
                                        "[pssid][task]",
                                        "[pssid][contexts]",
                                        "message",
                                        "event",
                                        "input",
                                        "agent",
                                        "host",
                                        "ecs",
                                        "[pssid][result][intervals]"
                                ]
                        }
                }
        }
}
output{
        opensearch{
                hosts => ["http://opensearch:9200"]
                index => "pssid-%{+YYYY.MM.dd}"
        }
        stdout{
                codec => rubydebug
        }
}
```

### 5. Clone this repo

```bash
git clone https://github.com/UMNET-perfSONAR/ansible-playbook-pssid-daemon.git
```

### 5.1. Change the ansible-playbook-pssid-daemon/inventory/hosts

```ini
[probes]
1rp ansible_user=pdaem
2rp ansible_user=pdaem
3rp ansible_user=pdaem
4rp ansible_user=pdaem
5rp ansible_user=pdaem
6rp ansible_user=pdaem
7rp ansible_user=pdaem
8rp ansible_user=pdaem
```

### 5.2. Install roles

ansible-galaxy install -f -r requirements.yml --ignore-errors

### 5.3. Change filebeat role

Go to ansible-playbook-pssid-daemon/roles/ansible-role-filebeat/tasks

Create file filebeat.yml in this directory

```yml
filebeat.inputs:
    - type: log
      paths:
        - /var/log/pssid.log
      max_bytes: 10485760
output.logstash:
  hosts:
    - "172.30.0.10:9400"

  logging:
    level: debug
    to_files: true
    json: true
    files:
      path: /var/log/filebeat
      name: filebeat.log
      keepfiles: 7
      permissions: 0644
```

Change main.yml to

```yml
- name: Add Elastic GPG key
  apt_key:
    url: https://artifacts.elastic.co/GPG-KEY-elasticsearch
    id: 46095ACC8548582C1A2699A9D27D666CD88E42B4
    state: present

- name: Add Filebeat repository
  apt_repository:
    repo: "deb https://artifacts.elastic.co/packages/7.x/apt stable main"
    state: present
    filename: 'elastic-7.x'
    update_cache: true

- name: Install apt-transport-https and gnupg and Filebeat
  package: # might change to package instead of apt for OS agnostic
    name: "{{ item }}"
    state: present
  loop:
    - apt-transport-https
    - gnupg
    - filebeat

- name: Copy Filebeat configuration
  ansible.builtin.template:
    src: filebeat.yml
    dest: /etc/filebeat/filebeat.yml
    owner: root
    group: root
    mode: 0644
  notify: restart filebeat

- name: Ensure Filebeat is started and enabled at boot.
  service:
    name: filebeat
    state: started
    enabled: true
```

### 5.4. Change VT role

Go to ansible-playbook-pssid-daemon/roles/ansible-role-pssid-VT-tools/templates

change wpa_supplicant.conf.j2 to

```j2
ctrl_interface={{ wpa_supplicant_profiles.global_settings.ctrl_interface }}

network={
 ssid="{{ network.ssid }}"
 key_mgmt={{ network.key_mgmt }}
 proto={{ network.proto }}
 pairwise={{ network.pairwise }}
 group={{ network.group }}
 psk="{{ network.password }}"
}
```

Create a vault with wpa_supplicant_profiles.yml

```bash
ansible-vault create roles/<role_name>/defaults/wpa_supplicant_profiles.yml
```

Inside editor put your wifi config

```yml
wpa_supplicant_profiles:
  global_settings:
    ctrl_interface: /run/wpa_supplicant

  networks:
    - ssid: "WifiName"
      password: "your_wifi_password"
      key_mgmt: WPA-PSK
      proto: RSN
      pairwise: CCMP
      group: CCMP
```

To edit it

```bash
ansible-vault edit roles/<role_name>/defaults/wpa_supplicant_profiles.yml
```

To view it

```bash
ansible-vault view roles/<role_name>/defaults/wpa_supplicant_profiles.yml
```

### 5.5. Check if host is reachable

```bash
ansible <host-pattern> -m ping
```

### 6. Set up the deamon on each of the probes

```bash
ansible-playbook --ask-vault-pass --ask-pass --ask-become-pass --user usernamehere --become --become-user root --become-method su --inventory inventory/ playbook.yml
```

### 7. Connect Grafana to Opensearch

Put ```http://opensearch:9200``` as URL and choose access method as server

Write pssid-* index name

### 8. Grafana Monitoring

For the most of the tests the Lucene query is

```Lucene
test_type:<name_of_the_test> AND device:<device_name>
```

For the throughput test it is necessary to also provide destination_ip

```Lucene
test_type:throughput AND throughput_destination:<dest_ip> AND device:<device_name>
```

For the most of the test, it is more useful to use time series panel so it is easier to see changes over time. But for packet loss and throughput retransmits the choice fell on statistic average.

You can import dashboard from dashboard-example.json

1. Navigate to Dashboards → New → Import
2. Drag and drop JSON file dashboard-example.json

---
