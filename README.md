# Setting-Up-a-Honeypot-in-Azure


<h2>Description</h2>
In this porject we'll be setting up a honeypot in Microsoft Azure and locating attackers through an API and a Powershell script.

<br>
<br />

> [!NOTE]
>Please setup a VM on Azure before attemping this.




<h2>Utilities and Languages Used</h2>

- <b>Sentinel</b>
- <b>Powershell</b>
- <b>ipgeolocation.io</b>


<h2>Environments Used</h2>

- <b>Microsoft Azure</b>
- <b>Windows 10 (21H2)</b>

## Program Walk-through

### Micrsoft Defender for Cloud:

To begin search for `Micrsoft Defender for Cloud` and enable it, then add your machine to it.
<p align="center">
<img src="https://i.imgur.com/UR1GyXc.png" height="80%" width="80%"/>

Next, navigate to `Environment Settings` on the left and turn on `Servers` and turn off `SQL Servers` then go to `Data collection` on the left and choose `All events`
<p align="center">
<img src="https://i.imgur.com/IbzKsYp.png" height="80%" width="80%"/>


<h2> </h2>

### Log Analytics Workspaces:

Now we have to enable log 
```
nano /etc/elasticsearch/elasticsearch.yml
```
<p align="center">
<img src="https://i.imgur.com/JXoOF9k.png" height="80%" width="80%"/>

Now we have to start and enable ElasticSearch:
```
systemctl start elasticsearch
systemctl enable elasticsearch
```
<h2> </h2>

### Configuring TheHive:

We start by changing the TheHive ownership to the destination directories:
```
chown -R thehive:thehive /opt/thp
```
Next, in the "application.conf" file change all the `hostname` and `application.baseUrl` to your TheHive IP address:

```
nano /etc/thehive/application.conf
```

<p align="center">
<img src="https://i.imgur.com/BJjQZWb.png" height="80%" width="80%"/>

Once again, we have to start and enable TheHive:
```
systemctl start thehive
systemctl enable thehive
```

<h2> </h2> 

### Adding The Agent

In the Wazuh directory, there's a tar file containing the credentials we'll need to log in. The main two are "admin user" and "API user"

```
tar -xvf wazuh-install-files.tar
cd wazuh-install-files
cat wazuh-passwords.txt 
```
Now we can add an agent to start collecting telemetry. In the Wazuh dashboard click on "Add agent" and follow the prompts. Be sure to set your Wazuh public IP address as the server IP address.
<br/>
> [!TIP]
> You'll need to run the terminal with admin privileges when adding the agent
<br>

#### Configuring The Agent

To ingest logs for our telemetry collection, we need to edit the "ossec.conf" file. You'll either find it in `C:\Program Files (x86)\ossec-agent` for Windows OS or in `/var/ossec/etc/` for Linux OS.
<br/>
Inside the file you'll find `<!-- Log Analysis -->`, this is where you can add the location of the logs you want to collect. Make sure to restart Wazuh after editing the conf file.

<p align="center">
<img src="https://i.imgur.com/GPz81qk.png" height="80%" width="80%"/>
<br/>

> [!IMPORTANT]
> It's best practice to create a backup file in case things go wrong!


<h2> </h2> 

### Configuring Wazuh

Next, we'll have to enable Wazuh to ingest these logs. To do so, we need to edit the filebeat.yml in the Wazuh Manager's CLI and enable archiving.
```
sudo nano /etc/filebeat/filebeat.yml
```

<p align="center">
<img src="https://i.imgur.com/8JODF9s.png" height="80%" width="80%"/>

Head back to the Wazuh dahsboard, and click on the top left menu button > Stack management > Index pattern > Create index, and name it "wazuh-archives-**" so we can search everything. In the "Time field" choose "timestamp" and finally, Create index pattern.

<p align="center">
<img src="https://i.imgur.com/kmtg95u.png" height="80%" width="80%"/>

<h2> </h2> 

### Creating Rules

It's time to create our first rule! In this example we want to detect Mimikatz usage in our network.
<br>
In the Wazuh dashboard click on "wazuh." > Management > Rules > Manage rules file. 
<br> 
Search "sysmon" and you can find "0800-sysmon_id_1.xml", click on the eye icon to the right of it to view it then copy any of the rules so you can use it as a template for your first custom rule.

<p align="center">
<img src="https://i.imgur.com/Sw536g2.png" height="80%" width="80%"/>


Click on "custom rules" and you should find "local_rules.xml", click on "edit" and paste in the rule we just copied (Make sure to follow the same indentation).
<br>
Custom rules id start from 100000 and the severity levels range from 1 to 15 (most severe).

<p align="center">
<img src="https://i.imgur.com/aKq3gXV.png" height="80%" width="80%"/>
<br/>

> [!NOTE]
> Rules are case sensitive!

<h2> </h2> 

### Configuring Shuffle

Finally, to tie everything up we'll use [Shuffle](https://shuffler.io) as our SOAR platform. Create an account then navigate to "Workflows" and add a new workflow.
<br>
Next, click on "Triggers" in the bottom left and grab and drop the "Webhook" onto the canvas and name it "Wazuh-Alerts" then copy the "Webhook URI"

<p align="center">
<img src="https://i.imgur.com/jEi0zXO.png" height="80%" width="80%"/>
<br/>

Head back to the Wazuh Manager's CLI and open the ossec.conf file then paste in the "Webhook URI" between the `<global>` and `<alert>` tags (Make sure to follow the same indentation). 
```
sudo nano /var/ossec/etc/ossec.conf
```

<p align="center">
<img src="https://i.imgur.com/vo8N8I3.png" height="80%" width="80%"/>

As always, whenever we change the config file we must restart the service.
```
systemctl restart wazuh-manager.service
```

#### Parsing Data

Since we're collecting lots of information, we want to parse the exact data to make it easier to understand. In this case we'll parse the hash value so we can feed it into VirusTotal:
1. Click on "Change_me" to rename it and set the "Find Actions" to "Regex capture group". 
2. In "input data" click the + icon and choose "hashes".
3. In the "Regex" field enter "SHA256=([0-9A-Fa-f]{64})", this enables us to parse the SHA256 hash.

<p align="center">
<img src="https://i.imgur.com/98k96YO.png" height="80%" width="80%"/>
<br/>

#### Incorporating VirusTotal

Shuffle allows us to use VirusTotal for enrichment. To do so, we'll have to sign up for VirusTotal and copy the API key for our account. Now let's set it up to look up the hash on our Mimikatz file:
1. In Shuffle search "Virustotal" in "Active Apps", then drag and drop it on the canvas.
2. Click on the VirusTotal icon then "Authenticate" and enter your API key.
3. In "Find Actions" choose "Get a hash report".
4. In the "Hash" field click the + icon and choose "SHA256_Regex" then "List"
5. Save and run it

<p align="center">
<img src="https://i.imgur.com/bapWcRC.png" height="80%" width="80%"/>
<br/>

<h2> </h2> 

### Creating Alerts
Log-in to TheHive with the default credentials (provided in the Installation Instruction file), then create a new organization and create 2 accounts within it: a service account and a normal account. 
<br>
For the service account create an API Key and copy it. Next, log-in using the normal account you just created, and head back to Shuffle set up Thehive:
1. Search "TheHive" in "Active Apps", then drag and drop it on the canvas.
2. Click on the TheHive icon then Authenticate and add in your API key, for the url add in your TheHive public IP address along with the port number.
3. In "Find Actions" choose "Create Alert", then scroll down until you find the "Date" field and choose Execution Argument > utcTime
4. Add in the description you want to assist the Analyst with investigating the alert.
5. Set Flag to "false", Pap to 2, Severity to 2, Source to "Wazuh" and Status to "New".
6. In the Tags field, you can add in the Mitre Attack Tag brackets. In this case ["T10003"] stands for credentials dumping, which is what Mimikatz is known for.

<p align="center">
<img src="https://i.imgur.com/hoGujCk.png" align="center" height="80%" width="80%"/>
<br>
<img src="https://i.imgur.com/DLH0OzE.png" height="80%" width="80%"/>
<br/>

<h2> </h2> 

## Video Tutorial
Major shout-out to [MyDFIR](https://www.youtube.com/@MyDFIR) for sharing his knowledge with us. If you prefer to learn through videos you can [find his tutorial here](https://www.youtube.com/watch?v=XR3eamn8ydQ)


</p>
<!--
 ```diff
- text in red
+ text in green
! text in orange
# text in gray
@@ text in purple (and bold)@@
```
--!>
