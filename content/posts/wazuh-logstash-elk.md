---
title: "From Agent to Dashboard: Shipping Wazuh Alerts into ELK with Logstash"
date: 2026-09-30
description: "A full walkthrough of a client build: Wazuh agents feeding a Wazuh server, then Logstash shipping every alert into Elasticsearch and Kibana. Every tool explained, every step shown, ending with a tuned SOC dashboard."
tags: ["wazuh", "elk", "logstash", "elasticsearch", "kibana", "siem", "soc"]
categories: ["cybersecurity"]
image: "images/wazuh-elk/architecture.png"
draft: true
---

Hey 👋

This one is long. On purpose.

Recently I built a log pipeline for a client: **Wazuh** collecting and analysing security events from their machines, and **Logstash** shipping every Wazuh alert into an **Elasticsearch + Kibana** (ELK) stack, where we built the dashboard their team actually looks at.

When I was researching it, I found plenty of docs that each cover *one piece*. What I didn't find was one place that walks through the whole thing, end to end, explaining **what each tool is doing and why**. So this is that post.

> *Client details, hostnames, and IPs are changed or removed. Everything below uses example values like `10.0.0.10`. Swap in your own.*

---

## What we're building

Before touching a terminal, here's the full picture:

![Wazuh + ELK architecture](/images/wazuh-elk/architecture.svg)

There are two paths in this diagram:

- **The blue path is Wazuh's own pipeline.** Agents send events to the Wazuh server, the server turns them into alerts, and Wazuh stores and shows them in its own indexer and dashboard. This works out of the box.
- **The orange path is what we add.** Logstash sits on the Wazuh server, reads every alert the moment it's written, and ships it to a separate Elasticsearch cluster, where Kibana visualises it.

Why both? Wazuh is excellent at **detection**: collecting logs, decoding them, and matching them against thousands of rules. But in this project the requirement was to have the alerts **in Elasticsearch and Kibana as well**, alongside the other data there, with a custom dashboard on top. Logstash is the bridge between the two worlds.

---

## Meet the tools (what each one actually does)

If you're new to this, the names get confusing fast. Here's each tool in plain English, then in technical terms.

### 🛰️ Wazuh agent
**Plain English:** a small program you install on every machine you want to watch. It's the security camera on each computer.

**Technical:** a lightweight daemon that reads log files (`/var/log/auth.log`, Windows Event Logs, etc.), monitors file integrity, checks configuration against security baselines, and inventories installed software. It sends everything to the Wazuh server over an encrypted channel on **port 1514/TCP**.

### 🧠 Wazuh server (the "manager")
**Plain English:** the brain. It receives everything from the agents and decides what's suspicious.

**Technical:** the `wazuh-manager` service runs an analysis engine (`wazuh-analysisd`). Every incoming event goes through **decoders** (which pull out fields like source IP, username, and program) and then **rules** (thousands of them, each with a severity level from 0 to 15 and often a MITRE ATT&CK mapping). When a rule matches at level 3 or above, the server writes an **alert**, as one JSON line, to `/var/ossec/logs/alerts/alerts.json`. **That file is the most important file in this entire post.**

### 🚚 Filebeat
**Plain English:** a delivery driver that ships the alerts from the brain to the storage room.

**Technical:** installed automatically with Wazuh. It tails `alerts.json` and sends each alert to the Wazuh indexer. We don't touch it; it keeps Wazuh's own dashboard working.

### 🗄️ Wazuh indexer
**Plain English:** Wazuh's own storage room with a very fast search desk.

**Technical:** a search engine based on OpenSearch. It stores alerts as documents in indices and answers queries on **port 9200**.

### 📺 Wazuh dashboard
**Plain English:** the TV screen for Wazuh's own storage room.

**Technical:** a web UI (based on OpenSearch Dashboards) on **port 443** for managing agents, rules, and viewing alerts.

### 🔀 Logstash
**Plain English:** a mail sorter. It picks up letters from one place, optionally stamps or sorts them, and posts them somewhere else.

**Technical:** a data processing pipeline from Elastic. Every pipeline has three stages: an **input** (where data comes from), optional **filters** (transform, enrich, or drop data), and an **output** (where data goes). We'll use it to read `alerts.json` and send it to Elasticsearch.

### 🔎 Elasticsearch
**Plain English:** a second, very large storage room with a very fast search desk.

**Technical:** a distributed search and analytics engine. It stores JSON documents in indices, and it's what Kibana queries. It also listens on **port 9200** (on a different machine from the Wazuh indexer, which matters!).

### 📊 Kibana
**Plain English:** the TV screen for Elasticsearch.

**Technical:** Elastic's web UI on **port 5601**, used for searching (Discover), building visualisations (Lens), and dashboards.

---

## How one event travels through all of this

The tools make more sense when you follow a single event. Say someone tries to SSH into a server as `root` with the wrong password:

![The journey of one failed SSH login](/images/wazuh-elk/event-journey.svg)

1. Linux writes a line to `/var/log/auth.log`.
2. The **Wazuh agent** on that server reads the new line and sends it (encrypted) to the Wazuh server.
3. The **Wazuh server** decodes it (program `sshd`, source IP `203.0.113.7`, user `root`), matches rule **5760** ("sshd: authentication failed", level 5, mapped to MITRE T1110 Brute Force), and writes an alert to `alerts.json`.
4. **Logstash** notices the new line in `alerts.json`, parses the JSON, and ships it over TLS to Elasticsearch.
5. **Elasticsearch** stores it in today's index, `wazuh-alerts-4.x-2026.09.30`, and within seconds it shows up on the **Kibana** dashboard.

Keep this picture in your head. Every troubleshooting step later on is really just "which of these five hops is broken?"

---

## The plan: machines, sizes, and ports

For this build we used two Ubuntu 24.04 servers.

| Host | Role | What runs on it | Minimum size (small environment) |
|---|---|---|---|
| **Host 1** `10.0.0.10` | Wazuh all-in-one | Wazuh server, indexer, dashboard, **+ Logstash** | 4 vCPU · 8 GB RAM · 50 GB+ disk |
| **Host 2** `10.0.0.20` | Elastic Stack | Elasticsearch + Kibana | 4 vCPU · 8 GB RAM · disk for your retention |
| **Endpoints** | Monitored machines | Wazuh agent | Tiny footprint |

Wazuh's own sizing guide says 4 vCPU / 8 GB / 50 GB covers up to about 25 agents with 90 days of data; bigger fleets need more.

> ⚠️ **Why two hosts?** The Wazuh indexer and Elasticsearch **both want port 9200** and both want lots of RAM. Putting them on separate machines avoids a whole class of problems.

### Firewall rules

This table saved me a lot of "why can't it connect?" time. Open only what's needed:

| From | To | Port | Why |
|---|---|---|---|
| Endpoints | Host 1 | **1514/tcp** | Agents send events |
| Endpoints | Host 1 | **1515/tcp** | Agents enrol (register) the first time |
| Network devices | Host 1 | 514/udp | Optional: syslog from firewalls/switches |
| Admins | Host 1 | **443/tcp** | Wazuh dashboard |
| Host 1 (Logstash) | Host 2 | **9200/tcp** | Logstash → Elasticsearch |
| Admins / SOC | Host 2 | **5601/tcp** | Kibana |

On Ubuntu with `ufw`, for example on Host 1:

```bash
sudo ufw allow 1514/tcp
sudo ufw allow 1515/tcp
sudo ufw allow from 10.0.0.0/24 to any port 443 proto tcp
```

And on Host 2, only allow Elasticsearch from the Wazuh server:

```bash
sudo ufw allow from 10.0.0.10 to any port 9200 proto tcp
sudo ufw allow from 10.0.0.0/24 to any port 5601 proto tcp
```

---

## Part 1: Installing the Wazuh server (Host 1)

### Step 1: Prepare the server

Update the system and set a clear hostname. You'll thank yourself when reading logs later.

```bash
sudo apt update && sudo apt -y upgrade
sudo hostnamectl set-hostname wazuh-server
```

Make sure the clock is right. Security logs with wrong timestamps are worse than useless.

```bash
timedatectl
sudo timedatectl set-ntp true
```

{{< screenshot caption="Terminal: timedatectl showing 'System clock synchronized: yes' on wazuh-server" >}}

### Step 2: Run the Wazuh installation assistant

Wazuh ships an installer that sets up the **server, indexer, and dashboard** on one machine (the "all-in-one" deployment), including generating TLS certificates between them.

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

- `-a` means **all-in-one**.
- It takes a few minutes. Get a coffee ☕

When it finishes, it prints the dashboard URL and the `admin` password. **Copy them somewhere safe right away.**

{{< screenshot caption="Terminal: the end of wazuh-install.sh output, with 'INFO: Installation finished.' and the admin credentials (blur the password!)" >}}

### Step 3: Recover the passwords (if you missed them)

All generated passwords are saved inside an archive next to the script:

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

### Step 4: Check that the three services are running

```bash
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard --no-pager
```

All three should say **active (running)**.

{{< screenshot caption="Terminal: systemctl status showing wazuh-manager, wazuh-indexer and wazuh-dashboard all active (running)" >}}

### Step 5: Log in to the Wazuh dashboard

Open `https://10.0.0.10` in a browser. You'll get a certificate warning because the certificates are self-signed. That's expected; accept it for now. Log in with `admin` and the password from Step 2.

{{< screenshot caption="Browser: the Wazuh dashboard home page right after the first login, with 0 agents" >}}

At this point Wazuh is running but blind. It has no agents yet, so let's give it some eyes.

---

## Part 2: Deploying Wazuh agents

### Step 6: The easy way, using the "Deploy new agent" wizard

In the Wazuh dashboard, go to **Agents management → Summary → Deploy new agent**. You pick the operating system, type in the server address (`10.0.0.10`), optionally name the agent and choose a group, and **the wizard generates the exact install command** for you, with the correct version number.

This is the method I recommend, because the command always matches your server version.

{{< screenshot caption="Browser: the Deploy new agent wizard with OS selected, server address filled in, and the generated command visible" >}}

### Step 7: Installing on Linux (Debian/Ubuntu) by hand

If you prefer to understand every line, here's what the wizard does under the hood.

Add Wazuh's signing key and repository:

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
```

Install the agent, telling it where the server is:

```bash
sudo WAZUH_MANAGER="10.0.0.10" WAZUH_AGENT_NAME="web-01" apt-get install -y wazuh-agent
```

The `WAZUH_MANAGER` variable writes the server address into the agent's config, so the agent **enrols itself** over port 1515 on first start.

Start it and make it start on boot:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
```

Finally, **stop the agent from auto-upgrading** when you run `apt upgrade`. An agent newer than the server can cause problems, so upgrade the server first, then the agents, on your own schedule:

```bash
sudo apt-mark hold wazuh-agent
```

{{< screenshot caption="Terminal: systemctl status wazuh-agent showing active (running) on web-01" >}}

### Step 8: Installing on Windows

On Windows, the wizard gives you a PowerShell one-liner. It downloads the MSI and installs it silently with the server address, and looks like this (use the exact version your wizard shows):

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-<VERSION>-1.msi -OutFile $env:tmp\wazuh-agent.msi
msiexec.exe /i $env:tmp\wazuh-agent.msi /q WAZUH_MANAGER='10.0.0.10' WAZUH_AGENT_NAME='dc-01'
NET START Wazuh
```

Run PowerShell **as Administrator**, or the install silently does nothing.

{{< screenshot caption="PowerShell (Administrator): the install command and 'The Wazuh service was started successfully.'" >}}

### Step 9: Confirm the agents are connected

On the Wazuh server:

```bash
sudo /var/ossec/bin/agent_control -l
```

You should see each agent listed as **Active**.

{{< screenshot caption="Terminal: agent_control -l listing the agents with status Active" >}}

And in the dashboard, the agents now appear with their OS, IP, and status:

{{< screenshot caption="Browser: Wazuh dashboard → Agents list showing all agents as Active" >}}

---

## Part 3: Choosing what the agents collect

Out of the box, agents already read the important system logs. But every environment has its own apps, like web servers, databases, and custom services, and you have to tell the agents about them.

### Step 10: Understand the agent config

Each agent's main config file is:

- Linux: `/var/ossec/etc/ossec.conf`
- Windows: `C:\Program Files (x86)\ossec-agent\ossec.conf`

Logs are collected by `<localfile>` blocks. For example:

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/auth.log</location>
</localfile>
```

### Step 11: Push config centrally with agent groups

Editing `ossec.conf` on 50 machines by hand is a nightmare. Instead, put agents into **groups** and push a shared config from the server.

Create a group and add an agent to it:

```bash
sudo /var/ossec/bin/agent_groups -a -g webservers -q
sudo /var/ossec/bin/agent_groups -a -i 003 -g webservers -q
```

(`003` is the agent ID from `agent_control -l`.)

Then edit the group's shared config on the server, `/var/ossec/etc/shared/webservers/agent.conf`:

```xml
<agent_config>
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/nginx/access.log</location>
  </localfile>
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/nginx/error.log</location>
  </localfile>
</agent_config>
```

Every agent in `webservers` picks this up automatically within a few minutes. No SSH needed.

{{< screenshot caption="Browser: Wazuh dashboard → Agents management → Groups → webservers, showing the agent.conf content" >}}

### Step 12: Test it with a real event

Let's generate the failed-login alert from earlier. From another machine:

```bash
ssh root@10.0.0.30   # type a wrong password 3–4 times
```

Then, on the Wazuh server, watch alerts arrive live:

```bash
sudo tail -f /var/ossec/logs/alerts/alerts.json | grep -i "authentication failed"
```

You'll see JSON lines appear almost instantly. **This is the file Logstash will read later.**

{{< screenshot caption="Terminal: tail -f alerts.json showing the sshd authentication failed alert as JSON" >}}

And in the Wazuh dashboard under **Threat Hunting**, the same alert shows up with rule 5760 and its MITRE mapping:

{{< screenshot caption="Browser: Wazuh Threat Hunting → Events, filtered to rule.id 5760, one alert expanded" >}}

✅ **Checkpoint:** agents → server → alerts works. The blue path is done.

---

## Part 4: Installing Elasticsearch and Kibana (Host 2)

Now the second machine. We install the Elastic Stack from Elastic's official repository.

> 📌 **Versions matter.** Elasticsearch, Kibana, and Logstash should all be the **same version**. Wazuh publishes its Elastic template and dashboards for the 8.x line; if you use 9.x (like the repo below), test in staging first, or change `9.x` to `8.x` in the repo line to stay on the officially documented combination.

### Step 13: Add Elastic's repository

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
sudo apt-get install -y apt-transport-https
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/9.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-9.x.list
sudo apt-get update
```

### Step 14: Install Elasticsearch

```bash
sudo apt-get install -y elasticsearch
```

Pay attention to the output. Elasticsearch **turns on security automatically**: it generates TLS certificates and prints a password for the built-in `elastic` superuser in a block titled *"Security autoconfiguration information"*. **Save that password.**

{{< screenshot caption="Terminal: the 'Security autoconfiguration information' block from the Elasticsearch install (blur the password!)" >}}

Missed it? Reset it any time:

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

### Step 15: Let Elasticsearch listen on the network

Open `/etc/elasticsearch/elasticsearch.yml` and check these lines:

```yaml
cluster.name: soc-elk
node.name: elk-01
network.host: 10.0.0.20
http.port: 9200
```

The security auto-configuration also added a block at the bottom (`xpack.security.*` settings and `http.host`). **Leave that block alone.**

### Step 16: Start Elasticsearch and test it

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now elasticsearch
```

Test it with the auto-generated CA certificate:

```bash
sudo curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic https://10.0.0.20:9200
```

If you get back a JSON block with `"tagline" : "You Know, for Search"`, Elasticsearch is alive. 🎉

{{< screenshot caption="Terminal: curl to https://10.0.0.20:9200 returning the cluster JSON with 'You Know, for Search'" >}}

### Step 17: Install Kibana

```bash
sudo apt-get install -y kibana
```

Edit `/etc/kibana/kibana.yml` so it's reachable from other machines:

```yaml
server.host: "10.0.0.20"
server.port: 5601
server.publicBaseUrl: "http://10.0.0.20:5601"
```

Start it:

```bash
sudo systemctl enable --now kibana
```

### Step 18: Enrol Kibana with Elasticsearch

Kibana needs to trust and log in to Elasticsearch. Elastic makes this easy with an **enrollment token**.

Generate a token on Host 2:

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
```

Open `http://10.0.0.20:5601`, paste the token, and when Kibana asks for a verification code, get it with:

```bash
sudo /usr/share/kibana/bin/kibana-verification-code
```

Then log in as `elastic`.

{{< screenshot caption="Browser: Kibana 'Configure Elastic to get started' screen with the enrollment token pasted" >}}

{{< screenshot caption="Browser: the Kibana home page after the first login" >}}

✅ **Checkpoint:** Elasticsearch and Kibana are running and secured. Now we connect the two worlds.

---

## Part 5: A dedicated user for Logstash

It's tempting to give Logstash the `elastic` superuser password. **Don't.** If the Wazuh server is ever compromised, the attacker gets full control of your Elasticsearch. Give Logstash **only** the permissions it needs.

### Step 19: Create a role that can only write Wazuh alerts

Run this on Host 2 (it asks for the `elastic` password):

```bash
sudo curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic \
  -X POST "https://10.0.0.20:9200/_security/role/wazuh_logstash_writer" \
  -H 'Content-Type: application/json' -d '
{
  "cluster": ["monitor", "manage_index_templates"],
  "indices": [
    {
      "names": ["wazuh-alerts-*"],
      "privileges": ["create_index", "create", "write", "manage"]
    }
  ]
}'
```

What each privilege is for:

- `manage_index_templates`: Logstash uploads the Wazuh index template (Part 6).
- `monitor`: Logstash checks the cluster's health and version.
- `create_index`, `create`, `write`: create the daily index and write alerts into it.
- `manage`: needed for index-level housekeeping (e.g. refreshing and checking the index).

And it can **only** touch indices named `wazuh-alerts-*`. Nothing else.

### Step 20: Create the Logstash user

```bash
sudo curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic \
  -X POST "https://10.0.0.20:9200/_security/user/wazuh_logstash" \
  -H 'Content-Type: application/json' -d '
{
  "password": "CHANGE-ME-to-a-long-random-password",
  "roles": ["wazuh_logstash_writer"],
  "full_name": "Logstash shipper for Wazuh alerts"
}'
```

You can also do both steps by clicking in Kibana under **Stack Management → Security → Roles / Users**.

{{< screenshot caption="Browser: Kibana → Stack Management → Roles → wazuh_logstash_writer showing the index privileges" >}}

---

## Part 6: Installing and configuring Logstash (on Host 1)

Here's the key design decision: **Logstash runs on the Wazuh server**, right next to `alerts.json`. It reads the file locally, so there's no extra network hop on the Wazuh side, and it only needs outbound access to Elasticsearch on 9200.

![Inside Logstash: input, filter, output](/images/wazuh-elk/logstash-pipeline.svg)

### Step 21: Install Logstash

On **Host 1**, add the same Elastic repository as in Step 13, then:

```bash
sudo apt-get update
sudo apt-get install -y logstash
```

Make sure it's the **same version** as your Elasticsearch:

```bash
/usr/share/logstash/bin/logstash --version
```

The Elasticsearch output plugin comes bundled with Logstash. You can confirm or update it with:

```bash
sudo /usr/share/logstash/bin/logstash-plugin install logstash-output-elasticsearch
```

### Step 22: Copy Elasticsearch's CA certificate to Host 1

Logstash talks to Elasticsearch over HTTPS, so it needs to **trust** Elasticsearch's certificate. Copy the CA from Host 2:

```bash
# on Host 1
sudo mkdir -p /etc/logstash/elasticsearch-certs
sudo scp admin@10.0.0.20:/etc/elasticsearch/certs/http_ca.crt /etc/logstash/elasticsearch-certs/root-ca.pem
sudo chmod 644 /etc/logstash/elasticsearch-certs/root-ca.pem
```

(If `scp` can't read the file because it's owned by root, copy it to `/tmp` on Host 2 first with `sudo cp` and `chmod`, then fetch it.)

Test that the certificate and the new user work **before** involving Logstash:

```bash
curl --cacert /etc/logstash/elasticsearch-certs/root-ca.pem -u wazuh_logstash https://10.0.0.20:9200
```

If that works, Logstash will work. If it doesn't, fix it here first. It's much easier to debug with curl.

{{< screenshot caption="Terminal on Host 1: curl with the copied root-ca.pem and wazuh_logstash user returning the cluster JSON" >}}

### Step 23: Download the Wazuh index template

Elasticsearch needs to know the **type** of each field. Is `rule.level` a number? Is `timestamp` a date? Is `GeoLocation.location` a map point? Wazuh publishes a template that answers all of that:

```bash
sudo mkdir -p /etc/logstash/templates
sudo curl -o /etc/logstash/templates/wazuh.json \
  https://packages.wazuh.com/integrations/elastic/4.x-8.x/dashboards/wz-es-4.x-8.x-template.json
```

Without it, Elasticsearch guesses, and it guesses wrong: numbers become text, and your dashboards can't do "level ≥ 12".

### Step 24: Store the credentials in the Logstash keystore

Never put passwords in plain text in a config file. Logstash has a **keystore** for secrets:

```bash
sudo -E /usr/share/logstash/bin/logstash-keystore --path.settings /etc/logstash create
sudo -E /usr/share/logstash/bin/logstash-keystore --path.settings /etc/logstash add ELASTICSEARCH_USERNAME
sudo -E /usr/share/logstash/bin/logstash-keystore --path.settings /etc/logstash add ELASTICSEARCH_PASSWORD
```

Type `wazuh_logstash` for the username and the password from Step 20. The config file then refers to them as `${ELASTICSEARCH_USERNAME}` and `${ELASTICSEARCH_PASSWORD}`.

(`create` may ask about protecting the keystore with a password. For a systemd service, the simple route is to continue without one; the file itself is only readable by root and logstash.)

### Step 25: Write the pipeline

This is the heart of the whole integration. Create `/etc/logstash/conf.d/wazuh-elasticsearch.conf`:

```ruby
input {
  file {
    id => "wazuh_alerts"
    codec => "json"
    start_position => "beginning"
    stat_interval => "1 second"
    path => "/var/ossec/logs/alerts/alerts.json"
    mode => "tail"
    ecs_compatibility => "disabled"
  }
}

output {
  elasticsearch {
    hosts => ["https://10.0.0.20:9200"]
    index => "wazuh-alerts-4.x-%{+YYYY.MM.dd}"
    user => "${ELASTICSEARCH_USERNAME}"
    password => "${ELASTICSEARCH_PASSWORD}"
    ssl_enabled => true
    ssl_certificate_authorities => ["/etc/logstash/elasticsearch-certs/root-ca.pem"]
    template => "/etc/logstash/templates/wazuh.json"
    template_name => "wazuh"
    template_overwrite => true
  }
}
```

Line by line, because this is where most mistakes happen:

**Input:**
- `path`: the alerts file the Wazuh server writes.
- `codec => "json"`: each line is a complete JSON document, so parse it as JSON instead of treating it as plain text.
- `mode => "tail"`: keep watching the file for new lines forever, like `tail -f`.
- `start_position => "beginning"`: the **first** time Logstash sees the file, read it from the top, so existing alerts get shipped too. After that, Logstash remembers its position (in a "sincedb" file) and continues where it left off.
- `stat_interval => "1 second"`: check for new lines every second. That's why alerts show up in Kibana so quickly.
- `ecs_compatibility => "disabled"`: keep Wazuh's own field names (`rule.level`, `agent.name`) instead of renaming them to Elastic Common Schema. The template and dashboards expect Wazuh's names.

**Output:**
- `hosts`: your Elasticsearch, **with `https://`**.
- `index`: one index per day (`wazuh-alerts-4.x-2026.09.30`). Daily indices make retention easy: deleting old data is just deleting old indices.
- `user` / `password`: pulled from the keystore.
- `ssl_enabled` / `ssl_certificate_authorities`: use TLS and trust Elasticsearch's CA.
- `template*`: upload the Wazuh template so fields get the right types.

> 📌 Older guides (and Logstash 7/8 examples) use `ssl => true` and `cacert => "..."`. Those options were **removed in Logstash 9**. The `ssl_enabled` and `ssl_certificate_authorities` names above work on both 8.x and 9.x.

### Step 26 (optional but worth it): Add a filter stage

The pipeline works without filters. But filters are where Logstash earns its place. Here's a filter block worth adding, between `input` and `output`:

```ruby
filter {
  # 1. Add a location to public source IPs (for the world map panel)
  if [data][srcip] {
    geoip {
      source => "[data][srcip]"
      target => "GeoLocation"
      ecs_compatibility => "disabled"
      tag_on_failure => ["_geoip_private_or_unknown"]
    }
  }

  # 2. Tag where the data came from, useful once more sources join the cluster
  mutate {
    add_field => { "[pipeline][source]" => "wazuh-server-01" }
  }
}
```

- The **geoip** filter looks up `data.srcip` in a built-in GeoIP database and writes the country, city, and coordinates into `GeoLocation`, which the Wazuh template already maps as a `geo_point`. Private IPs like `10.x` can't be located, so they just get a tag instead of failing.
- The **mutate** filter adds a field recording which Wazuh server shipped the alert.

### Step 27: Let Logstash read the alerts file

`alerts.json` belongs to the `wazuh` group. The `logstash` user isn't in it, so without this step Logstash silently reads nothing:

```bash
sudo usermod -a -G wazuh logstash
```

### Step 28: Test the config before starting

Logstash can check a config for syntax errors without running it:

```bash
sudo -u logstash /usr/share/logstash/bin/logstash --path.settings /etc/logstash --config.test_and_exit
```

You want to see **`Configuration OK`**.

{{< screenshot caption="Terminal: logstash --config.test_and_exit printing 'Configuration OK'" >}}

### Step 29: Start Logstash

```bash
sudo systemctl enable --now logstash
```

Watch its log while it starts:

```bash
sudo tail -f /var/log/logstash/logstash-plain.log
```

Good signs: `Pipeline started {"pipeline.id"=>"main"}` and a line about installing the `wazuh` template. Bad signs: anything with `401` (wrong credentials), `PKIX` / `certificate` (wrong CA file), or `Permission denied` (Step 27).

{{< screenshot caption="Terminal: logstash-plain.log showing the template installed and 'Pipeline started'" >}}

---

## Part 7: Seeing the data in Elasticsearch and Kibana

### Step 30: Check the index exists

On Host 2:

```bash
sudo curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic "https://10.0.0.20:9200/_cat/indices/wazuh-alerts-*?v"
```

You should see today's index with a growing `docs.count`.

{{< screenshot caption="Terminal: _cat/indices showing wazuh-alerts-4.x-YYYY.MM.dd with health green/yellow and a docs.count" >}}

> On a single-node cluster the index health may show **yellow**. That just means replica copies have nowhere to go. It's normal for one node.

### Step 31: Create a data view in Kibana

Kibana needs to know which indices to look at. Go to **Stack Management → Data Views → Create data view**:

- **Name:** `Wazuh alerts`
- **Index pattern:** `wazuh-alerts-*`
- **Timestamp field:** `timestamp`

{{< screenshot caption="Browser: Kibana Create data view dialog with wazuh-alerts-* and timestamp selected" >}}

### Step 32: Explore in Discover

Open **Discover**, pick the `Wazuh alerts` data view, and set the time range to the last 24 hours. Every Wazuh alert is here now.

Useful columns to add: `agent.name`, `rule.level`, `rule.description`, `data.srcip`.

Try a query in the search bar (this is KQL, Kibana Query Language):

```text
rule.groups : "authentication_failed" and rule.level >= 5
```

{{< screenshot caption="Browser: Kibana Discover with the Wazuh alerts data view, the columns above, and the KQL query applied" >}}

✅ **Checkpoint:** the orange path works. Agent → Wazuh → Logstash → Elasticsearch → Kibana, end to end. 🎉

---

## Part 8: Importing Wazuh's ready-made dashboards

Wazuh publishes a set of Kibana dashboards for this exact integration. They're a great starting point.

### Step 33: Download and import

Download the file to your laptop:

```text
https://packages.wazuh.com/integrations/elastic/4.x-8.x/dashboards/wz-es-4.x-8.x-dashboards.ndjson
```

In Kibana: **Stack Management → Saved Objects → Import**, select the `.ndjson` file, and confirm.

{{< screenshot caption="Browser: Kibana Saved Objects import dialog showing the Wazuh objects imported successfully" >}}

Open **Dashboards** and you'll find the Wazuh ones (security events overview, integrity monitoring, and so on).

{{< screenshot caption="Browser: one of the imported Wazuh dashboards in Kibana, populated with data" >}}

---

## Part 9: Building my own SOC dashboard

The imported dashboards are good. But a SOC analyst starting a shift wants **one screen** that answers: *How bad is it? What's happening? Where? Who?* So I built this:

![SOC Overview dashboard layout](/images/wazuh-elk/dashboard-layout.svg)

All panels are built with **Lens** (Kibana's drag-and-drop editor). Go to **Dashboards → Create dashboard**, then **Create visualization** for each panel below.

### Step 34: Global settings first

- **Time range:** Last 24 hours
- **Refresh every:** 30 seconds
- **Dashboard query:** `rule.level >= 3` (hides informational noise)

Save the dashboard as **"SOC Overview"** with **Store time with dashboard** turned on, so it always opens on the last 24 hours.

### Step 35: Panels 1–4, the headline numbers

Four **Metric** visualisations across the top:

| # | Title | Metric | Filter (KQL) | Colour |
|---|---|---|---|---|
| 1 | Total alerts | Count | *(none)* | Cyan |
| 2 | Critical | Count | `rule.level >= 12` | Red |
| 3 | Auth failures | Count | `rule.groups : "authentication_failed"` | Amber |
| 4 | Active agents | Unique count of `agent.name` | *(none)* | Green |

For the **Critical** tile, set a colour rule: *red when above 0*. An empty red tile is the most calming thing a SOC can see.

{{< screenshot caption="Browser: Lens editor building the 'Critical' metric with the rule.level >= 12 filter and red colour rule" >}}

### Step 36: Panel 5, alerts over time by severity

- **Visualization:** Bar vertical stacked
- **Horizontal axis:** `timestamp` (date histogram, auto interval)
- **Vertical axis:** Count
- **Breakdown:** *Intervals* on `rule.level` with three ranges:
  - `0 → 7` label **Low**
  - `7 → 12` label **Medium**
  - `12 → 16` label **Critical**

Colours: Low = cyan, Medium = amber, Critical = red. A spike of red bars is visible from across the room.

{{< screenshot caption="Browser: Lens stacked bar chart with the three rule.level intervals and custom colours" >}}

### Step 37: Panel 6, top 10 rules

- **Visualization:** Bar horizontal
- **Vertical axis:** Top 10 values of `rule.description`
- **Horizontal axis:** Count

This is the "what's noisy?" panel. When one rule dominates, it's either an attack or a rule that needs tuning. Both are worth knowing.

### Step 38: Panel 7, MITRE ATT&CK tactics

- **Visualization:** Donut
- **Slice by:** Top 8 values of `rule.mitre.tactic`
- **Size by:** Count

It turns raw alerts into attacker language: *Credential Access*, *Persistence*, *Privilege Escalation*. It's great for management reports.

### Step 39: Panel 8, top agents by alerts

- **Visualization:** Bar horizontal
- **Vertical axis:** Top 10 values of `agent.name`
- **Horizontal axis:** Count

If one server suddenly tops this list, go look at that server.

### Step 40: Panel 9, top source IPs attacking logins

- **Visualization:** Table
- **Rows:** Top 10 values of `data.srcip`
- **Metric:** Count
- **Panel filter:** `rule.groups : "authentication_failed"`

This is your block-list candidate table.

### Step 41: Panel 10, latest critical alerts

In **Discover**, filter to `rule.level >= 12`, add the columns `agent.name`, `rule.description`, `data.srcip`, sort by `timestamp` descending, and **Save** it as a search called "Critical alerts". Then on the dashboard use **Add from library** to drop it in as a live table.

### Step 42 (bonus): The world map

Because of the geoip filter from Step 26, public source IPs now have coordinates. Add a **Maps** panel with a *Documents* layer on `GeoLocation.location`, filtered to `rule.groups : "authentication_failed"`. You get a live map of where brute-force attempts come from.

### Step 43: Arrange, save, share

Drag the panels into the layout above. Put the numbers on top, the timeline full width, and details below. Then **Save**.

{{< screenshot caption="Browser: the finished SOC Overview dashboard in Kibana, all panels populated (hide real IPs and hostnames!)" >}}

---

## Part 10: Fine-tuning and hardening

A working pipeline is step one. A pipeline you can **trust for months** needs a bit more.

### Step 44: Retention, so the disk doesn't fill up

With daily indices, retention means "delete indices older than N days". Elasticsearch does this automatically with **Index Lifecycle Management (ILM)**.

Create a policy in **Stack Management → Index Lifecycle Policies**, called `wazuh-alerts-90d`, with only a **Delete** phase at **90 days**.

Then tell the template to use it. Open `/etc/logstash/templates/wazuh.json`, find the `"settings"` block inside `"template"`, and add one line under `"index"`:

```json
"lifecycle": { "name": "wazuh-alerts-90d" },
```

Restart Logstash (`sudo systemctl restart logstash`) so it re-uploads the template. Every **new** daily index now deletes itself after 90 days.

> Why edit the file and not the template in Kibana? Because we set `template_overwrite => true`: Logstash re-uploads its file on every restart and would undo changes made in the UI.

### Step 45: Keep Wazuh's own indexer from filling up too

Remember, the blue path still stores everything in the Wazuh indexer. Set a retention policy there too, under **Indexer management → Index Management → State management policies** in the Wazuh dashboard. Otherwise Host 1's disk fills up while you're only watching Host 2.

### Step 46: Reduce noise at the source, not in the dashboard

If one rule floods everything with harmless alerts, don't just hide it in Kibana. Tune it in Wazuh, so the alert is never generated. Add an override to `/var/ossec/etc/rules/local_rules.xml` on the Wazuh server:

```xml
<group name="local,">
  <!-- Example: silence a known-harmless rule on one noisy host -->
  <rule id="100100" level="0">
    <if_sid>RULE_ID_TO_SILENCE</if_sid>
    <hostname>backup-01</hostname>
    <description>Known noise from backup-01, silenced.</description>
  </rule>
</group>
```

Then `sudo systemctl restart wazuh-manager`. Level 0 means "never alert".

### Step 47: Monitor the pipeline itself

A silent pipeline looks exactly like a quiet network. Quick health checks:

```bash
# Is Logstash reading and sending? (events in/out per pipeline)
curl -s localhost:9600/_node/stats/pipelines?pretty | grep -A3 '"events"'

# Is alerts.json still growing?
ls -lh /var/ossec/logs/alerts/alerts.json
```

In Kibana, add a simple rule under **Stack Management → Rules**: an *Elasticsearch query* rule on `wazuh-alerts-*` that fires when the document count over the last 15 minutes **is below 1**. If the pipeline dies, we find out in 15 minutes, not 15 days.

---

## Troubleshooting: common problems and fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| Logstash runs, but nothing arrives in Elasticsearch | `logstash` can't read `alerts.json` | Step 27 (`usermod -a -G wazuh logstash`), then restart Logstash |
| `401 Unauthorized` in the Logstash log | Wrong keystore username/password | Re-add them (Step 24); test with curl first (Step 22) |
| `PKIX path building failed` / certificate errors | Logstash doesn't trust Elasticsearch's CA | Copy `http_ca.crt` again; check the path in `ssl_certificate_authorities` |
| `Unknown setting 'ssl'` / `'cacert'` | Logstash 9 removed the old SSL options | Use `ssl_enabled` + `ssl_certificate_authorities` (Step 25) |
| `Limit of total fields exceeded` | Template not applied, so Elasticsearch guessed field types | Check the `template` path; restart Logstash; look for "template installed" in the log |
| Alerts in Kibana show the wrong time | Timezone / clock drift | NTP on every host (Step 1); Kibana → Advanced settings → timezone |
| Agent shows **Never connected** | Ports 1514/1515 blocked, or wrong `WAZUH_MANAGER` | Check the firewall; check `/var/ossec/logs/ossec.log` on the agent |
| Index health **yellow** | Single node, so replicas can't be placed | Normal for one node, or set replicas to 0 |

---

## Final thoughts

Here's what stuck with me from this build.

**Each tool does one job well.** Wazuh detects. Logstash moves and shapes data. Elasticsearch stores and searches. Kibana shows. Once you see it that way, the architecture stops being scary. It's just a relay race, and the baton is one line of JSON.

**`alerts.json` is the centre of the universe.** Almost every problem comes down to one question: *Is the alert in the file? Can Logstash read it? Can Logstash deliver it?*

**Test each hop before the next one.** Agent → server (`agent_control`). Server → file (`tail -f`). Logstash → Elasticsearch (`curl` with the same cert and user). Only then open Kibana. It turns a mysterious "no data" into a five-minute fix.

And finally, **a dashboard is only as good as the questions it answers.** Build it around what the analyst needs to decide in the first five minutes of a shift, not around every field you happen to have.

Anyway, back to the logs 🕵🏽‍♂️

---

*References: [Wazuh documentation](https://documentation.wazuh.com/current/) (installation, agents, Elastic Stack integration) and [Elastic documentation](https://www.elastic.co/docs) (Elasticsearch, Kibana, Logstash).*
