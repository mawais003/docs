# Here is a detailed step-by-step guide to install and configure the ELK stack (Elasticsearch, Logstash, Kibana) on Ubuntu 20.04 to receive logs from Filebeat on a remote server:

**Prerequisites**

- Two Ubuntu 20.04 servers: one for the ELK stack and one remote server running Filebeat to send logs to the ELK server.
- Ensure both servers have at least 4GB RAM and 2 CPU cores.
- Open the required ports between the servers:
- Filebeat to Logstash: 5044
- Elasticsearch: 9200
- Kibana: 5601

## Install Elasticsearch

1. Import the Elasticsearch GPG key:

`wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -`

Output Prompt: `OK`

2. Add the Elasticsearch repository:

`echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list`

Output Prompt: `deb https://artifacts.elastic.co/packages/7.x/apt stable main`

3. Install Elasticsearch:

`sudo apt update && sudo apt install elasticsearch`

4. Enable and start Elasticsearch:

`sudo systemctl enable elasticsearch`
`sudo systemctl start elasticsearch`

5. Verify Elasticsearch is running:

`curl -X GET "localhost:9200"`

Output Prompt:

```
{
  "name" : "elk-stack",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "xxx",
  "version" : {
    "number" : "7.17.22",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "38e9ca2e81304a821c50862dafab089ca863944b",
    "build_date" : "2024-06-06T07:35:17.876121680Z",
    "build_snapshot" : false,
    "lucene_version" : "8.11.3",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}

```

## Install Logstash

1. Install Logstash:

`sudo apt install logstash`

2. Create a Logstash configuration file:

`sudo nano /etc/logstash/conf.d/02-beats-input.conf`

- Add the following configuration:

```
input {
  beats {
    port => 5044
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
  }
}
```

3. Enable and start Logstash:

`sudo systemctl enable logstash`
`sudo systemctl start logstash`

## Install Kibana

1. Install Kibana:

`sudo apt install kibana`

2. Enable and start Kibana:

`sudo systemctl enable kibana`
`sudo systemctl start kibana`

3. Changing path of log storage inside "/etc/elasticsearch/elasticsearch.yml"

`path.data: /mnt/lvm/elasticsearch` ---> add this line, or if present update it accordingly.

4. Ensure that Elasticsearch has the necessary permissions to read and write to this directory:

`chown -R elasticsearch:elasticsearch /mnt/lvm/elasticsearch`
`chmod -R 755 /mnt/lvm/elasticsearch`

5. Kibana Configuration

Ensure Kibana is configured to connect to Elasticsearch by modifying the Kibana configuration file (kibana.yml):

`elasticsearch.hosts: ["http://localhost:9200"]` ---> add/modify this line according to requirement

## To configure log rotation and deletion of old logs for Elasticsearch, follow these steps:

1. Install Curator

Curator is a tool from Elastic that helps manage Elasticsearch indices. Install it on your Elasticsearch server.

```
sudo apt-get update
sudo apt-get install python3-pip
sudo pip3 install elasticsearch-curator
```

2. Create Curator Configuration File

Create the Curator configuration file at "/etc/curator/curator.yml"

```
# /etc/curator/curator.yml
client:
  hosts:
    - 127.0.0.1
  port: 9200
  timeout: 30
logging:
  loglevel: INFO
  logfile: /var/log/curator.log
  logformat: default
  blacklist: ['elasticsearch', 'urllib3']

```

3. Create Curator Action File

Create the Curator action file at "/etc/curator/actions.yml"

```
# /etc/curator/actions.yml
actions:
  1:
    action: delete_indices
    description: "Delete indices older than 7 days"
    options:
      ignore_empty_list: True
      timeout_override:
      continue_if_exception: False
      disable_action: False
    filters:
    - filtertype: pattern
      kind: prefix
      value: mail-logs-
    - filtertype: age
      source: name
      direction: older
      timestring: '%Y.%m.%d'
      unit: days
      unit_count: 7

```

4. Schedule Curator to Run Periodically

Create a cron job to run Curator daily.

- Open the crontab editor:
  - `crontab -e`

- Add the following line to run Curator every day at midnight:

  - `0 0 * * * /usr/local/bin/curator --config /etc/curator/curator.yml /etc/curator/actions.yml`

