<img width="1247" height="420" alt="image" src="https://github.com/user-attachments/assets/23a7d453-570b-4f25-bf96-db0517b6ffb3" /># Automated-SOC-Triage-Pipeline-Wazuh-Shuffle-TheHive-

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


# 2. Environment Setup
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

### 2. TheHive Server Installation (Docker)
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

**Installing Ngrok** <br>

Ngrok was installed from the official repository using the following method: <br>

curl -sSL https://ngrok-agent.s3.amazonaws.com/ngrok.asc \ <br>
  | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null \ <br>
  && echo "deb https://ngrok-agent.s3.amazonaws.com bookworm main" \ <br>
  | sudo tee /etc/apt/sources.list.d/ngrok.list \ <br>
  && sudo apt update \ <br>
  && sudo apt install ngrok <br>
<br>
  <img width="600" height="350" alt="ngrok" src="https://github.com/user-attachments/assets/d0b2c4f2-6220-455f-bfd9-cd96a473af65" />



This procedure installed the Ngrok package, set up the repository, and added Ngrok's official GPG key. <br>

After installation, the Ngrok agent was authenticated using a personal authtoken: <br>

**ngrok config add-authtoken <YOUR_NGROK_AUTHTOKEN>** <br>
<br>
<img width="600" height="350" alt="ngrokAuth" src="https://github.com/user-attachments/assets/baada72b-dbaa-42bd-864b-42698aff4e2d" />


This allowed Ngrok to generate public tunnels under the authenticated account. Finally, TheHive dashboard was exposed via the following command:

**ngrok http 9000**
<br>

<img width="600" height="360" alt="ngrokHttp" src="https://github.com/user-attachments/assets/b1f9626d-c9d7-465a-81a1-aaf55b5c4ca2" />
<br>

This provided a public HTTPS URL, which was used in the Shuffle workflow and email notifications to SOC analysts. The terminal displayed real-time request logs and the public URL. The access was verified by opening the URL in a browser and logging into TheHive using the analyst and API users.

<img width="600" height="350" alt="ngrokFinal" src="https://github.com/user-attachments/assets/8ce87dc6-6f1a-46ec-a849-809a8a66c1e5" />
<br>

# 3. Shuffle Orchestration
Shuffle is the automation engine that connects Wazuh, VirusTotal, TheHive, and email notifications.

### Workflow Overview (Integrated Here):
- **Detection:** A Mimikatz alert is triggered on a monitored Windows 10 endpoint.
- **Webhook Trigger:** Wazuh captures the alert and sends it to Shuffle via webhook.
- **Extraction:** Shuffle extracts the SHA256 hash from the alert payload using regex.
- **Threat Intelligence Enrichment:** The hash is checked using VirusTotal for file reputation.
- **Incident Creation:** Shuffle sends alert details to TheHive, which creates an incident under the organization The Hive Mind.
- **Notification:** Shuffle sends an email to SOC analysts to begin investigation.

<br>
<img width="600" height="350" alt="shuffler" src="https://github.com/user-attachments/assets/acfb22a9-8dac-4a38-b763-88fa4d3718dd" />
<br>

### Detailed Steps in Shuffle:
- **Regex Capture Group:** Extract SHA256 from Wazuh alert JSON payload.
- **VirusTotal Integration:** Query reputation score and threat classification for the hash.
- **TheHive Integration:** Create incident automatically using API user SOAR.
- **Email Notification:** Compose and send alert summary email to SOC analyst.

# 4. TheHive Configuration
- **Organization Setup and Users:** To centralize and arrange all SOC incidents resulting from automated alerts, TheHive created an organization called The Hive Mind.  Two users were set up in this organization: SOAR, an API user that Shuffle uses only for programmatic incident creation, and Abel, the SOC analyst in charge of manually reviewing and looking into incidents.  With this configuration, analysts could easily monitor and handle alerts, and automated workflows could generate incidents under the appropriate organization. <br>
- **Incident Fields Captured:** Several important fields were recorded in the incident created in TheHive when a Mimikatz alert was generated on the Windows 10 endpoint and handled by Shuffle.  These comprised the endpoint hostname and IP, the suspicious file's SHA256 hash, the alert severity, the detection timestamp, and the VirusTotal reputation information.  By recording these specifics, analysts were guaranteed to have all the data they needed to make prompt and well-informed decisions during triage.<br>
<br>

<img width="800" height="450" alt="TheHiveWeb" src="https://github.com/user-attachments/assets/10387be1-462b-4dc0-8fea-5417d8664216" />
<br>

# 5. VirusTotal Integration
- **Hash Extraction and Enrichment:** Shuffle automatically retrieved the Wazuh alert payload's SHA256 hash and used the VirusTotal API to retrieve threat intelligence.  The file's detection ratio, reputation score, and threat family details were all supplied by the integration.  Without having to manually search for threat intelligence sources, SOC analysts were able to evaluate the risk level of detected files instantly thanks to this enrichment.
- **Impact on Incident Management:** TheHive incident automatically included the VirusTotal data, providing analysts with actionable intelligence within the event.  The time needed to validate questionable files was greatly decreased by this automation, which also increased the SOC workflow's overall effectiveness. <br>
<br>

<img width="600" height="350" alt="virusTotal200" src="https://github.com/user-attachments/assets/8cfeedac-61e6-45f0-8a6a-d58bf4e26e61" />
<br>

# 6. Email Notifications
- **Automation of Analyst Alerts:** Every time a new incident was created in TheHive, Shuffle was set up to automatically send out email notifications.  The SHA256 hash, the VirusTotal reputation score, the impacted endpoint, a thorough alert summary, and a direct link to the incident were all included in each email.  This made it possible for SOC analysts to get the pertinent data right away and start their investigation right away.
- **Benefits to Response Time:** Automated email notifications enabled SOC analysts to stay situationally aware and react quickly to high-priority threats by giving them immediate access to incident details.  During Mimikatz alert simulations, the email integration was successfully tested, demonstrating that alerts were delivered to the analyst instantly.
<br>

<img width="600" height="350" alt="mail" src="https://github.com/user-attachments/assets/eec42cad-6b01-44b6-a5d9-311fe1dcdf27" />
<br>

# 7. Testing the Workflow
- **Controlled Simulation:** Mimikatz was used to simulate malicious activity in the Windows 10 virtual machine.  Wazuh recorded this activity and used a webhook to send Shuffle the alert.  Shuffle used the API user SOAR to create an incident in TheHive after extracting the SHA256 hash and enriching it with VirusTotal.  Simultaneously, the SOC analyst Abel received an email notification.
- **Verification and Validation:** The SOC analyst received the email, accessed TheHive via the Ngrok public URL, and reviewed the incident. All fields, including the SHA256 hash, endpoint details, alert severity, and VirusTotal reputation, were correctly populated. This confirmed that the automated workflow functioned correctly from detection to incident creation and analyst notification.

#### Image Placeholders for Documentation:
- **Windows 10 Execution:** Screenshot showing Mimikatz execution on the Windows 10 VM, demonstrating the initial alert trigger.
<br>

<img width="600" height="350" alt="win10Exec" src="https://github.com/user-attachments/assets/52f9018b-3753-4247-a04b-96443c3d214a" />
<br>
<br>
- **Wazuh Log Verification:** Screenshot of the Wazuh dashboard showing the captured Mimikatz alert and relevant log details.
<br>
<br>
<img width="600" height="350" alt="wazuhDashboard2" src="https://github.com/user-attachments/assets/99fc3a69-3f3a-4fa5-a97b-2d6e53298ed1" />
<br>
<br>

- **Shuffle Workflow Execution:** Screenshots from Shuffle displaying the workflow execution, including SHA256 extraction, VirusTotal enrichment, and incident creation steps.
<br>

<img width="600" height="350" alt="shuffler" src="https://github.com/user-attachments/assets/f69638a7-0d43-40a7-9b75-e89629d465c8" />

<img width="600" height="350" alt="runtime" src="https://github.com/user-attachments/assets/795b4faa-84c7-48ca-acf6-db57c38df679" />

<img width="600" height="350" alt="regex" src="https://github.com/user-attachments/assets/4f3e4d2d-9e0c-4816-a2d7-e2a148cb8318" />

<img width="600" height="350" alt="virustotal" src="https://github.com/user-attachments/assets/987cac32-9ce8-4a24-8880-a142668be09b" />

<img width="600" height="350" alt="emailShuff" src="https://github.com/user-attachments/assets/c2f957a8-6bb9-4ba7-9358-ab3b71a98fa2" />

<img width="600" height="350 " alt="thehive" src="https://github.com/user-attachments/assets/c3e98382-499f-424f-a875-819c78cd0132" />
<br>

<br>

- **Email Notification:** Screenshot of the automated email received by the SOC analyst abel, showing alert summary, endpoint details, hash, and VirusTotal reputation.
<br>
<img width="600" height="350" alt="emailG" src="https://github.com/user-attachments/assets/a5ff4037-d39b-4c2f-962b-26d5bc8702d2" />

<br>

- **TheHive Incident Review:** Screenshot of the TheHive dashboard showing the incident created by Shuffle, with all fields correctly populated and enriched.
<br>

<img width="600" height="350" alt="thehive" src="https://github.com/user-attachments/assets/120b2d31-66e0-4fc4-9032-260569a09f2c" />
<br>
<br>


# 8. Project Benefits
- **Automated Detection and Triage:** By doing away with manual alert triage steps, the system made it possible for alerts to be processed automatically.  Instead of manually collecting data, SOC analysts could concentrate on more important tasks.
- **Threat Intelligence Enrichment:** Making decisions more quickly and intelligently was made possible by the integration with VirusTotal, which guaranteed that every SHA256 hash was enhanced with trustworthy threat intelligence.
- **Incident Management and Tracking:** Making decisions more quickly and intelligently was made possible by the integration with VirusTotal, which guaranteed that every SHA256 hash was enhanced with trustworthy threat intelligence.
- **Email Notification for Real-Time Awareness:** Analysts were notified immediately of new incidents via email, ensuring rapid response times and improved situational awareness
- **Scalable Architecture:** The workflow was made to be easily scalable, allowing for the addition of more endpoints and alert sources without requiring changes to the current procedures.

# 9. Future Enhancements
- **IOC Enrichment Expansion:** The workflow could be extended to include enrichment of additional IOCs such as domains, IP addresses, and URLs from multiple threat intelligence sources.
- **Automated Blocking Integration:** Future developments might incorporate endpoint security platforms or firewall rules to automatically block malicious files or IP addresses based on the alert data.
- **Monitoring and Dashboarding:** Dashboards could be created to visualize SOC operations metrics, track incident response times, and monitor workflow efficiency.
- **Containerization of Wazuh Server:** To simplify deployment and replication across different environments, the Wazuh server could be containerized, enhancing scalability and operational consistency.
