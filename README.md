# DShield Sensor Log Collection with Elasticsearch
## Introduction
This is a fork and a significant update from the initial publication on the ISC Storm Center website by Scott Jensen as a BACS paper and the scripts published in Github.<br>
https://github.com/fkadriver/Dshield-ELK<br>
https://isc.sans.edu/diary/DShield+Sensor+Monitoring+with+a+Docker+ELK+Stack+Guest+Diary/30118<br>

This is a good reference on how to use DShield-SIEM for analysis: https://isc.sans.edu/diary/30962/

## DShield SIEM Network Flow
This provides an overview of how the log collection with the DShield sensor is done.<br>
https://github.com/bruneaug/DShield-SIEM/blob/main/Troubleshooting/DShield-SIEM-Flow.png

# Why Use This?
This docker is custom built to be used with the [DShield Honeypot](https://isc.sans.edu/tools/honeypot/) to collect and parse the logs and collect the data to make it easy to search for research purposes. The suggested installation is to install the DShield sensor in a Rasperry using PI Raspbian OS or a system running Ubuntu 20.04 LTS either in your network or in the cloud of your choice.<br>

**Note**: This must be installed on a separate server per the instructions below.<br>

- This was tested on Ubuntu 20.04, 22.04 & 24.04 LTS or [Proxmox LXC](https://github.com/bruneaug/DShield-SIEM/blob/main/AddOn/LXC_Container_DShield-SIEM.md) Container <br>
- Step 1 build Ubuntu<br>
- Step 2 install docker and ELK<br>
- Step 3 install and configure Filebeat on DShield Sensor(s)<br>

# Ubuntu Setup
#### Like in an enterprise, a system collecting security logs and monitoring a network should be installed on a separate server. That is, the DShield SIEM should not be installed on the same system as the DShield sensor. 

- Ubuntu 22.04 LTS Live Server 64-Bit<br>
- Minimum 8+ GB - Recommended 10-12 GB RAM<br>
  - If the amount of RAM assigned to each container (see below) is more than 2GB, consider increasing the server RAM capacity.<br>
- 4-8 Cores<br>
- Add 2 partitions, one for the OS, the other for docker<br>
- Minimum 300 GB partition assigned to /var/lib/docker<br>
- After Ubuntu is rebooted, setup the docker partition<br>
- Adding a 300 GB to a VM: https://github.com/bruneaug/DShield-SIEM/blob/main/AddOn/Build_a_Docker_Partition.md
## Elastic Packages Installed
### ELK Current Version: 8.17.3 (Updated 4 Mar 2025)
- Kibana
- Elasticsearch
- Logstash
- Elastic-Agent
- Filebeat
- Heartbeat
- Metricbeat

**Note**: To update ELK server components, follow these steps:<br>
https://github.com/bruneaug/DShield-SIEM/blob/main/README.md#download-github-update

## Install docker
Install docker user user ($) account:
````
sudo apt-get install ca-certificates curl gnupg network-manager txt2html
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
Note: You may need to take the next line, remove the backlash () and put everythin in a single line in Notepad to run this echo.
echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
"$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update && sudo apt upgrade
sudo reboot (if update were applied)
sudo apt-get install -y jq docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin pip
sudo systemctl enable docker
sudo systemctl start docker.service
````
Confirm the docker service is running<br>
````
sudo systemctl status docker.service
````

## Update /etc/hosts
Add the following ELK servername **es01 fleet-server heartbeat logstash filebeat filebeat01**<br>
to /etc/hosts after the IP address and the server name:<br>
````
sudo vi /etc/hosts
````
Add this to /etc/hosts
````
es01 fleet-server heartbeat logstash filebeat filebeat01
````
## Add to sysctl vm.max_map_count
Edit the sysctl.conf file and add the following at the bottom of the file:
````
sudo vi /etc/sysctl.conf
vm.max_map_count=262144
````
Add save it. This is for [Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#docker-cli-run-prod-mode) when using Docker.

## Ensure the Firewall is Off
````
sudo iptables -nL
````
![image](https://github.com/user-attachments/assets/e6d260bd-021f-4dc0-9c27-86878f84d637)

# Configure and install DShield ELK
Using user ($) account, run the following commands:<br>
````
git clone https://github.com/bruneaug/DShield-SIEM.git
mkdir scripts
mv DShield-SIEM/AddOnScripts/*.sh scripts
chmod 754 scripts/*.sh
cd ~/DShield-SIEM
````
**Note**: Before installation, you can edit the **.env** (ls -la to see it) file to make any derided changes.<br>
    - Current _default password_ for elastic is **student**<br>
Memory Limits in **.env** are the most memory that docker will allocate for each of the ELK containers.<br>
Default to 3GB for Elasticsearch but can be expanded if you have the resources<br>
Edit .env file and configure for your own ELK Stack<br>
````
vi .env
````
- Update the following variables in the **.env** file to match your ELK server DNS information, hostname, IP and default elastic password if you want to change it: <br>
  - HOSTNAME="ubuntu"
  - DNS_SERVER="9.9.9.9"
  - IPADDRESS="192.168.25.231"
  - ELASTIC_PASSWORD=student
  - ES_MEM = Elasticsearch -> 3g, KB_MEM = Kibana -> 2g
- If you want to change the default nameserver(s) information and your local private network location for the destination address (i.e. DShield sensor mapping - currently set for Ottawa, Canada), edit to the following directory for the files before loading docker:<br>
````
  cd logstash/pipeline
````
  - logstash-200-filter-cowrie.conf <br>
  - logstash-201-filter-iptables.conf <br>
  - logstash-202-filter-cowrie-webhoneypot.conf <br>
You can keep these default or edit each files and change them.

Edit heartbeat.yml to change the PI IP to your own sensor. Kibana heartbeat check the status of your services and determine whether they are available.<br>
Update this IP and hostname to reflect your sensor. You can copy and add same 5 lines if you have more than one sensor you would like to monitor<br>
This can be viewed Observability -> Uptime Monitors<br>
![image](https://github.com/user-attachments/assets/2877b810-e727-48e6-a6ba-bcb71bd7be88)

Update this line: hosts: ["192.168.25.165"] <br>
````
vi ~/DShield-SIEM/heartbeat/heartbeat.yml
````

The script **change_perms.sh** will configure some of the files before starting the installation of the docker components.<br>
The script will also update the IP address used in the dashboard query to the IP of your ELK stack.<br>
````
~/scripts/change_perms.sh
````
Now execute docker compose to build the ELK server applications. <br>
This will build: Kibana, Elasticsearch, elastic-agent, Logstash and load the Cowrie parsers, configuration files and dashboard.<br>
````
sudo docker compose up -d
````
Installation Completed

![image](https://github.com/user-attachments/assets/4adccef9-a0d2-4084-9f93-71a61a2a8a43)

### Docker Directory Listing
This command shows the list of docker directories in used after all of Elasticsearch components have been installed.<br>
As data gets added to Elasticsearch, you can also monitor either with the command below or within ELK the amount of disk is available for storage.<br>
````
sudo du --human-readable --max-depth 1 --no-dereference --one-file-system /var/lib/docker
````
![image](https://github.com/bruneaug/DShield-SIEM/assets/48228401/2f3f3f6d-94a8-4154-b39e-7edaa4086572)

### The following ELK Services are Setup
Using netstat, these 4 services should now be listening.<br>
```
netstat -an | grep '9200\|9300\|8220\|5601\|5044\|5066\|5067\|5068\|9600'
````
This should be the output you should see:<br>
<pre>
tcp        0      0 0.0.0.0:5601            0.0.0.0:*               LISTEN  ---> Kibana Web Access
tcp        0      0 0.0.0.0:8220            0.0.0.0:*               LISTEN  ---> elastic-agent Service
tcp        0      0 0.0.0.0:9200            0.0.0.0:*               LISTEN  ---> Elasticsearch Service
tcp        0      0 0.0.0.0:9300            0.0.0.0:*               LISTEN  ---> Elasticsearch Cluster Service
tcp        0      0 0.0.0.0:5044            0.0.0.0:*               LISTEN  ---> Logstash Data Collection
tcp        0      0 0.0.0.0:5066            0.0.0.0:*               LISTEN  ---> Metricbeat Stack Node Information
tcp        0      0 0.0.0.0:5067            0.0.0.0:*               LISTEN  ---> Filebeat Stack Node Information
tcp        0      0 0.0.0.0:5068            0.0.0.0:*               LISTEN  ---> Filebeat Stack Node Information (Process ISC Threat Intel to Logstash)  
tcp        0      0 0.0.0.0:9600            0.0.0.0:*               LISTEN  ---> Logstash Stack Node and Pipeline Information
tcp6       0      0 :::5601                 :::*                    LISTEN
tcp6       0      0 :::8220                 :::*                    LISTEN
tcp6       0      0 :::9200                 :::*                    LISTEN
tcp6       0      0 :::9300                 :::*                    LISTEN
tcp6       0      0 :::5044                 :::*                    LISTEN
tcp6       0      0 :::5066                 :::*                    LISTEN
tcp6       0      0 :::5067                 :::*                    LISTEN
tcp6       0      0 :::5068                 :::*                    LISTEN
tcp6       0      0 :::9600                 :::*                    LISTEN
</pre>

# Loading Default Filebeat Templates in Kibana
This will install the default templates and dashboards can be used to store netflow data from remote DShield sensors.<br>
In order for Logstash to start storing data, the following commands must be executed. First step is to login filebeat:<br>
````
sudo docker exec -ti filebeat bash
````
Next, run this Filebeat command to load the templates into Kibana:<br>
````
./filebeat setup -e 
````

# Access Kibana Interface
After docker finish installing all the ELK docker components, now it is time to login the ELK stack using your Ubuntu server IP.<br>
- Login Kibana with username: **elastic** and default password _if it hasn't been changed_: **student**<br>
Web Access: https://serverIP:5601

![image](https://github.com/user-attachments/assets/c51f17d0-4475-4e8e-b272-6fd1ad373fcb)

# Configuring ELK Stack Management
In order to see the ELK stack health, it is necessary to configure the stack monitoring by navigating to: Management -> Stack Monitoring<br>

- This step is only necessary if the installation didn't automatically setup Stack Monitoring<br>
- Select "_Or, set up with self monitoring_"<br>
- Monitoring is currently off -> Select: _Turn on monitoring_<br>
![image](https://github.com/user-attachments/assets/129deb88-a657-4cab-9f44-64d337c5c9c6)


![image](https://github.com/user-attachments/assets/79b7a5c9-ebf7-40e2-8bd6-89d808c4c25b)

# Configuring the fleet-agent
The fleet-agent (this is an elastic-agent running on the ELK stack) will be used to ingest threat intelligence. It can also be used to do other things that won’t be covered here.<br>
This is an example for the format to setup the fleet-server and the elastic-agent:<br>
https://github.com/bruneaug/DShield-SIEM/blob/main/Troubleshooting/fleet-server-examples.txt

- From the dropdown menu, select Management → Fleet →Settings → Edit Outputs (Actions)<br>
- Login server via SSH<br>
- Copy ca.crt certificate to /tmp<br>
````
sudo cp /var/lib/docker/volumes/dshield-elk_certs/_data/ca/ca.crt /tmp
````
- Get a copy of Elasticsearch CA trusted fingerprint<br>
````
sudo openssl x509 -fingerprint -sha256 -noout -in /tmp/ca.crt | awk -F"=" {' print $2 '} | sed s/://g
````
- The output will look like this:<br>
673FB617E15CCCE73F9B647EF99449642A19CFC1D75BF5772047DA99DB950844

- Get Content of Elasticsearch CA Certificate to Apply to Advanced YAML configuration. Type the command because it doesn't copy well<br>
````
sudo cat /tmp/ca.crt | sed -r 's/(.*)/    \1/g'
````
Follow the example from the Troubleshooting fleet-server-examples guide URL above for the correct format.<br>
_sed_ will add the 4 spaces with the previous command against the CA certificate

- Change the Hosts to: https://es01:9200

After adding the certificate information, save and apply these settings.<br>
Followed by Save and deploy

![image](https://github.com/bruneaug/DShield-SIEM/assets/48228401/82d3b29f-5398-4f23-afea-63bf6727e087)
![image](https://github.com/bruneaug/DShield-SIEM/assets/48228401/bc97d11c-b9de-4c26-b24e-17ebed278cd2)

Under Settings, configure the Fleet server hosts by either edit or Add Fleet Server configured as follows:

![image](https://github.com/bruneaug/DShield-SIEM/assets/48228401/8c6abd5d-a7ba-4183-842a-c98410bd21f0)

Next phase is to Select Agent Policy → Add Agent → Enroll in Fleet → Add Fleet Server

- Provide a Name: es01
- Provide URL: https://fleet-server:8220
- Last: Generate Fleet Server policy
- Select: RPM
Refer to this page for an example of _Fleet-Server SSL Configuration_:<br>
https://github.com/bruneaug/DShield-SIEM/blob/main/Troubleshooting/fleet-server-examples.txt
  
![image](https://github.com/bruneaug/DShield-SIEM/assets/48228401/580b3009-0de8-45ba-b297-d4090e2120c9)

We are going to need this information to setup our fleet server.<br>
Login via SSH to the fleet-server and make sure the fleet-server is running before setting up our agent:<br>
````
sudo docker exec -ti fleet-server bash
./elastic-agent status (check it is running)
./elastic-agent restart (if it doesn't appear to be running, force a restart, and recheck the status)
````
This is an example of what need to be copied to the fleet server. Ensure the fleet server es is: https://es01:9200<br>
Add the bold section after port=8220 because are certificates are self-generated. This will ensure the agent takes the update.<br>

The token and fingerprint will be different than this example<br>

Copy the elastic-enrol agent below and eplace these 2 lines taken from your own serve and everything else remain the same.<br>
<pre>
--fleet-server-service-token=AAEAAWVsYXN0aWMvZmxlZXQtc2VydmVyL3Rva2VuLTE3MzEwOTcwODU3MzI6VjMyLU13cmFUOUM1eUFvMUhVUFl5QQ \
--fleet-server-es-ca-trusted-fingerprint=0D9A25F4C147EB3A496253525DF6F039CF3C19776E64A1F77CEFCCD08B76BC61 \
</pre>

While logged in the fleet-agent, enroll the elastic-agent in your fleet-agent:

Use this template to install your certificate and make sure you replace with your own<br>
fleet-server-service-token & fleet-server-es-ca-trusted-fingerprint<br>

````
elastic-agent enroll \
--url=https://fleet-server:8220 \
--fleet-server-es=https://es01:9200 \
--fleet-server-service-token=AAEAAWVsYXN0aWMvZmxlZXQtc2VydmVyL3Rva2VuLTE3MzEwOTcwODU3MzI6VjMyLU13cmFUOUM1eUFvMUhVUFl5QQ \
--fleet-server-policy=fleet-server-policy \
--fleet-server-es-ca=/certs/es01/es01.crt \
--fleet-server-es-ca-trusted-fingerprint=0D9A25F4C147EB3A496253525DF6F039CF3C19776E64A1F77CEFCCD08B76BC61 \
--fleet-server-port=8220 \
--certificate-authorities=/certs/ca/ca.crt \
--fleet-server-cert=/certs/fleet-server/fleet-server.crt \
--fleet-server-cert-key=/certs/fleet-server/fleet-server.key \
--elastic-agent-cert=/certs/fleet-server/fleet-server.crt \
--elastic-agent-cert-key=/certs/fleet-server/fleet-server.key \
--fleet-server-es-cert=/certs/fleet-server/fleet-server.crt \
--fleet-server-es-cert-key=/certs/fleet-server/fleet-server.key

````

This will replace your current settings. Do you want to continue? [Y/n]: Y<br>

![image](https://github.com/user-attachments/assets/dc1844c0-e7ef-428a-81a9-5b7d22089687)

**Successfully enrolled the Elastic Agent.**

From your current location, verify it installed correctly<br>
````
./elastic-agent status
./elastic-agent restart (if you got what appears to be errors, force a restart and check the status)
````
![image](https://github.com/bruneaug/DShield-SIEM/assets/48228401/a1bdb5ec-4cf9-4921-b3dc-ff4be0981bc8)

Now that the Fleet Server is connected, close this Windows and lets proceed to the next step.

![image](https://github.com/bruneaug/DShield-SIEM/assets/48228401/05f1b89e-b081-4ee7-bea6-9c83ae20d350)


In Elastic Management → Fleet, refresh Agents and this is what shows up:

![image](https://github.com/bruneaug/DShield-SIEM/assets/48228401/f2bb9efd-3c3e-463b-bb9a-a5b3e1bacfd0)

The server is now ready to install Threat Intel Agents to be used in Security (SIEM portion) against the honeypot logs.<br>
The next step is to select Agent policies → Fleet Server Policy → Add integration:<br>
Use this Dashboard to view the metrics collected by elastic-agent:<br>
![image](https://github.com/bruneaug/DShield-SIEM/assets/48228401/ebb0441a-36b4-454c-acd3-ae6d8262f7d9)


- Select and Add AlienVault OTX (need an API key)<br>
- Select AbuseCH (no API key needed)<br>
- Select Threat Intelligence Utilities<br>
- Elasticsearch<br>
- Select Kibana<br>
- Select Docker<br>

![image](https://github.com/bruneaug/DShield-SIEM/assets/48228401/64f795cf-e761-42ca-b94a-a79e87b16ee6)

# Fleet Server Policy Example

![image](https://github.com/bruneaug/DShield-SIEM/assets/48228401/9cbd15e0-1c03-4d79-8cf5-4e0bf1467b18)

- In Elastic Management → Installed Integration<br>
- Select each of the installed integration, then select Settings and enable the tab to _keep the policy up to date_:<br>

![image](https://github.com/bruneaug/DShield-SIEM/assets/48228401/f7f8d4d9-f5b3-45b2-849f-cb855bf6b757)

## Configuring Security → Rules

- Select Rules → Detection rules (SIEM) → Add Elastic rules
- Under Search Tags: Rule Type: Threat Intel (add at the minimum those 4 rules)
- Install and enable those 4 rules
  - Threat Intel Indicator Match - Cowrie (Custom rule loaded and enabled when docker load all the ELK stack components)
  - Threat Intel Windows Registry Indicator Match
  - Threat Intel Hash Indicator Match
  - Threat Intel URL Indicator Match
- You can look through the rules and enable those other rules that you want to try against your honeypot data.<br>

![image](https://github.com/user-attachments/assets/31cde450-2179-495e-8b73-7184ed5750b0)


## Review the Activity Captured by the Rules in Alert trend
If some of the threat intel above were added to the elastic-agent, and cowrie* was added to the Management -> Advanced Settings during the initial installation, it can now track rules that match in the alert trends of the SIEM part of the ELK stack.<br>
This is the output from Management → Stack Management → Advanced Settings<br>
 ![image](https://github.com/bruneaug/DShield-SIEM/assets/48228401/6112efdc-954e-4377-8a8e-31c005f54a18)

 If any threat intel match, the SIEM will display the following activity that can now be investigated:<br>
 
![image](https://github.com/bruneaug/DShield-SIEM/assets/48228401/4c2d50c9-5776-432d-8ca3-1fcc2306f2f1)
![image](https://github.com/bruneaug/DShield-SIEM/assets/48228401/09319047-e0bf-4377-a1e9-72b2fb08c893)


# Setup Filebeat on DShield Sensor - Logs to ELK

Next step is to add the Filebeat package to the DShield Sensor to send the logs the Elasticsearch.<br>

Use the following steps to install Filebeat using the following commands taken from this reference [3]: <br>
````
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo apt-get install apt-transport-https<br>
echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list
echo "deb https://artifacts.elastic.co/packages/oss-8.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list
sudo apt-get update && sudo apt-get install filebeat
````
If you want to use Zeek on your DShield sensor, review these installation [steps](https://github.com/bruneaug/DShield-SIEM/blob/main/AddOn/Configure_Zeek_DShield_Sensor.md)

### Transfer all Sensor Files to the DShield Sensor
Follow this URL to download all the files related to the DShield Sensor<br>
https://github.com/bruneaug/DShield-Sensor
````
git clone https://github.com/bruneaug/DShield-Sensor.git
sudo cp ~/DShield-Sensor/filebeat/filebeat.yml /etc/filebeat/filebeat.yml
````
- Edit filebeat.yml and change the IP address or hostname to the logstash parser (192.168.25.23) to match the IP used by Logstash:<br>
````
sudo vi /etc/filebeat/filebeat.yml
````
  output.logstash:<br>
  hosts: ["192.168.25.23:5044"]<br>
  #hosts: ["logstash.com:5044"]<br>

### Troubleshooting Filebeat
````
sudo su -
filebeat test config
````
Expected output: _Config OK<br>_
````
filebeat test output
````
Expected output:
<pre>
logstash: 192.168.25.231:5044...
  connection...
    parse host... OK
    dns lookup... OK
    addresses: 192.168.25.231
    dial up... OK
  TLS... WARN secure connection disabled
  talk to server... OK
</pre>

## Start Filebeat
````
sudo systemctl enable filebeat
sudo systemctl start filebeat
sudo systemctl status filebeat
sudo systemctl enable elastic-agent
sudo systemctl start elastic-agent
````
### Filebeat Tracking File
Filebeat tracks the events it has processed with a file located called **log.json**, if deleted, all the events that were previous sent to Elasticsearch will be reprocessed when filebeat is restarted.<br>
The location of this file:<br>
sudo su -
````
cd /var/lib/filebeat/registry/filebeat
````
If you are planning to resend all the logs because your ELK server got rebuild, _stop filebeat, delete log.json and restart filebeat_.
This will usually take a while to catch up in the ELK Stack dashboard. If you are looking to see if it is sending the logs to the ELK Stack, you could<br>
- tail -f log.json
- Look for the past 30 days in the dashboard while it is catching up

## Want to add Elastic-Agent to other Devices?
Follow this step-by-step documents to install the elastic-agent to the DShield sensor<br>
https://github.com/bruneaug/DShield-SIEM/blob/main/Configure-Elastic-Agent.pdf

# Interface - Logs DShield Sensor Overview
To access the Dashboard select Analytics -> Dashboard -> 	**[Logs DShield Sensor] Overview**<br>

![image](https://github.com/user-attachments/assets/a700729e-b2e4-4c78-833c-759df4ff609a)

## Adding ca.crt to Workstation Browser
Follow the steps in this [Page](https://github.com/bruneaug/DShield-SIEM/blob/main/AddOn/Workstation_Browser_CA_Certificate.md) to add to Kibana the ca.crt used by the ELK server.<br>

![image](https://github.com/user-attachments/assets/7c709b0a-72f9-45a5-a6e4-5a777aa642a3)

# Useful Docker Commands
Refer to this page for a list of useful docker commands.<br>
https://github.com/bruneaug/DShield-SIEM/edit/main/Troubleshooting/docker_useful_commands..md

# Information on Elastic with the Console
These command are run from the Dev Tool -> Console<br>

GET _nodes/http?pretty	(Get a list and information of all the nodes)<br>
GET _security/_authenticate<br>
GET /_cat/nodes?v=true<br>

# References
[1] https://isc.sans.edu/tools/honeypot/<br>
[2] https://www.elastic.co/downloads/beats/filebeat<br>
[3] https://www.elastic.co/guide/en/beats/filebeat/8.8/setup-repositories.html#_apt<br>
[4] https://isc.sans.edu/diary/DShield+Honeypot+Activity+for+May+2023/29932<br>
[5] https://isc.sans.edu/diary/DShield+Sensor+JSON+Log+to+Elasticsearch/29458<br>
[6] https://isc.sans.edu/diary/DShield+Sensor+JSON+Log+Analysis/29412<br>
[7] https://github.com/jslagrew/cowrieprocessor/blob/main/submit_vtfiles.py<br>
[8] https://handlers.sans.edu/gbruneau/elastic.htm<br>
[9] https://www.elastic.co/guide/en/fleet/current/secure-connections.html<br>
[10] https://www.docker.elastic.co/<br>
