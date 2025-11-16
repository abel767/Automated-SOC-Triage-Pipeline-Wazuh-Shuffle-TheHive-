# Automated-SOC-Triage-Pipeline-Wazuh-Shuffle-TheHive-

This project implements an automated SOC (Security Operations Center) workflow that detects suspicious activity in endpoints, enriches alerts with threat intelligence, and generates actionable incidents for SOC analysts. The system is designed for rapid detection, triage, and investigation of malware activity, using a combination of Wazuh, Shuffle, TheHive, and VirusTotal.
<br>
The primary focus is to detect Mimikatz activity on a Windows endpoint, automate its processing through a SOC workflow, and notify analysts for further investigation.
<br>
<br>
# 1. System Architecture
![Home Lab diagramF](https://github.com/user-attachments/assets/df0c9af7-de57-4888-ae77-0b1587910dd6)
<br>
Three primary parts of the architecture are installed in virtual machines:
<br>

**Endpoint VM:**
- Windows 10 environment.
- Used for testing malware activity, specifically Mimikatz.
- Monitored by Wazuh agent.<br>


**Wazuh Server VM:**
- Ubuntu server with manual installation of Wazuh 4.7+.
- Manages agent communications, rules, and alert indexing.
- Custom index (wazuh-archives-**) created for SOC alerts.<br>


**TheHive Server VM:**
- Ubuntu server running Docker with TheHive, Cassandra, and Elasticsearch.
- Ngrok is used to expose TheHive’s web interface (port 9000) externally.
- Organization: The Hive Mind.<br>

**Shuffle Orchestration:**
- Receives Wazuh alerts via webhook.
- Extracts relevant IOCs (SHA256 hash).
- Queries VirusTotal for reputation.
- Creates TheHive incidents.
- Sends email notifications to SOC analysts.<br>

<img width="600" height="350" alt="shuffler" src="https://github.com/user-attachments/assets/acfb22a9-8dac-4a38-b763-88fa4d3718dd" />

# Environment Setup
### 1. Wazuh Server Installation

Wazuh creates security alerts and keeps an eye on endpoints.<br>

**Installation Steps:** <br>

curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh <br>
sudo bash ./wazuh-install.sh -a <br>
sudo tar -xvf wazuh-install-files.tar <br>
<br>

<img width="600" height="350" alt="wazuh" src="https://github.com/user-attachments/assets/f25d1eef-78bf-462d-9f04-7c609268e1fd" />
<br>
<br>

**Configuration Highlights:** <br>
- Custom index for alerts: wazuh-archives-**
- Wazuh agent installed on Windows 10 VM
- Alerts configured to trigger webhooks to Shuffle

<img width="600" height="350" alt="wazuhDashbpard" src="https://github.com/user-attachments/assets/23d915ca-fe7c-44ec-b53c-b3f26c41b917" />
<br>
<br>

# 2. TheHive Server Installation (Docker)
TheHive is used for incident management and alert tracking.<br>

**Dependencies:** <br>

sudo apt update <br>
sudo apt install wget gnupg apt-transport-https git ca-certificates curl software-properties-common python3-pip lsb-release <br>

<img width="600" height="350" alt="theHive" src="https://github.com/user-attachments/assets/7529832f-cfdf-4de9-a91e-b5909bcc1b9b" />
<br>

**Java (Amazon Corretto 17):** 
<br>
wget -qO- https://apt.corretto.aws/corretto.key | sudo gpg --dearmor -o /usr/share/keyrings/corretto.gpg <br>
echo "deb [signed-by=/usr/share/keyrings/corretto.gpg] https://apt.corretto.aws stable main" | sudo tee /etc/apt/sources.list.d/corretto.sources.list <br>
sudo apt update <br>
sudo apt install java-17-amazon-corretto-jdk <br>
echo 'JAVA_HOME="/usr/lib/jvm/java-17-amazon-corretto"' | sudo tee -a /etc/environment <br>
export JAVA_HOME="/usr/lib/jvm/java-17-amazon-corretto" <br>
<br>
<br>
<img width="600" height="350" alt="thehiveJava" src="https://github.com/user-attachments/assets/8e1e3242-de7a-4968-bf3f-680ec5af91e4" />
<br>
<br>

**Cassandra Database:** <br>
wget -qO - https://downloads.apache.org/cassandra/KEYS | sudo gpg --dearmor -o /usr/share/keyrings/cassandra-archive.gpg <br>
echo "deb [signed-by=/usr/share/keyrings/cassandra-archive.gpg] https://debian.cassandra.apache.org 40x main" | sudo tee /etc/apt/sources.list.d/cassandra.sources.list <br>
sudo apt update <br>
sudo apt install cassandra <br>
sudo systemctl enable cassandra <br>
sudo systemctl start cassandra <br>
<br>
<img width="600" height="350" alt="casasndra" src="https://github.com/user-attachments/assets/ae7c7d98-5b5f-4b4a-b140-ea56e1d09590" /> <br>
<br>

**Elasticsearch (Latest 8.x):** <br>
<br>
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg <br>
sudo apt install apt-transport-https <br>
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list <br>
sudo apt update <br>
sudo apt install elasticsearch <br>
sudo systemctl enable elasticsearch <br>
sudo systemctl start elasticsearch <br>
<br>
<img width="600" height="350" alt="elastic" src="https://github.com/user-attachments/assets/d9862606-1051-4a96-b2ea-28014d6b7003" /> <br>
<br>

**Docker Deployment of TheHive:** <br>
<br>
sudo apt install docker.io docker-compose -y <br>
sudo docker pull thehiveproject/thehive:4.2.4-1 <br>
sudo docker run -d --name thehive -p 9000:9000 thehiveproject/thehive:4.2.4-1 <br>
<br>
<img width="600" height="350" alt="dockerHive" src="https://github.com/user-attachments/assets/d8746cc8-02b4-43e6-87f8-ef4694cf1985" /> <br>
<br>

**Ngrok for external access:** <br>
Ngrok was set up and installed on the TheHive server to enable external access to the TheHive instance running in Docker.  To enable the Shuffle workflow and SOC analysts to access the dashboard without directly exposing the internal virtual machine to the network, Ngrok established a secure public tunnel to port 9000.<br>

**1. Installing Ngrok** <br>

Ngrok was installed from the official repository using the following method: <br>

curl -sSL https://ngrok-agent.s3.amazonaws.com/ngrok.asc \ <br>
  | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null \ <br>
  && echo "deb https://ngrok-agent.s3.amazonaws.com bookworm main" \ <br>
  | sudo tee /etc/apt/sources.list.d/ngrok.list \ <br>
  && sudo apt update \ <br>
  && sudo apt install ngrok <br>

This procedure installed the Ngrok package, set up the repository, and added Ngrok's official GPG key. <br>

After installation, the Ngrok agent was authenticated using a personal authtoken: <br>

**ngrok config add-authtoken <YOUR_NGROK_AUTHTOKEN>** <br>

This allowed Ngrok to generate public tunnels under the authenticated account. Finally, TheHive dashboard was exposed via the following command:

**3.Exposing TheHive via Ngrok** <br>

ngrok http 9000
