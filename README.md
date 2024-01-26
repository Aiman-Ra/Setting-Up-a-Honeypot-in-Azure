# Setting-Up-a-Honeypot-in-Azure


<h2>Description</h2>
In this project, we'll be setting up a honeypot in Microsoft Azure and locating attackers with the use of an API and a Powershell script.

<br>
<br />

> [!NOTE]
>Please setup a VM on Azure before attempting this.


<h2>Utilities and Languages Used</h2>

- <b>Sentinel</b>
- <b>Powershell</b>
- <b>ipgeolocation.io</b>


<h2>Environments Used</h2>

- <b>Microsoft Azure</b>
- <b>Windows 10 (21H2)</b>

## Program Walk-through

### Micrsoft Defender for Cloud
To begin, head to Microsft Azure to search for `Micrsoft Defender for Cloud` and enable it, then add your machine to it and navigate to `Environment Settings` on the left.
<p align="center">
 <img src="https://i.imgur.com/UR1GyXc.png" height="80%" width="80%"/>

Click on your machine and turn on `Servers`, turn off `SQL Servers` then go to `Data collection` on the left and choose `All events`.
<p align="center">
 <img src="https://i.imgur.com/IbzKsYp.png" height="80%" width="80%"/>


<h2> </h2>

### Log Analytics Workspaces
Now we have to add our machine to `Log Analytics Workspaces`, simply search for it and add your machine.
<p align="center">
 <img src="https://i.imgur.com/2UWW3P9.png" height="80%" width="80%"/>

Once you've done that, login to your machine with RDP and disable the firewall. That way, attackers can find your machine and we can get their IP address.
<p align="center">
 <img src="https://i.imgur.com/ccpEMzt.png" height="80%" width="80%"/>

> [!WARNING]
> Never do this in a live production environment!


<h2> </h2>

### Locating the IP Addresses
We'll have to sign up at [ipgeolocation.io](https://www.ipgeolocation.io) to get our free API key so we can locate the attackers that are trying to login to our machine. Afterwards, download or copy the [Powershell script](https://github.com/joshmadakor1/Sentinel-Lab/blob/main/Custom_Security_Log_Exporter.ps1) on your VM and open it with `Powershell ISE` then add your API key and run it.
<p align="center">
 <img src="https://i.imgur.com/KM0quPJ.png" height="80%" width="80%"/>

#### How it works
The script looks for IP addresses found in Windows Event Viewer's `Event ID:4625` which stands for failed logons, and sends them to ipgeolocation to grab their latitude and longitude. Afterwards, it adds them to a log file in `C:\ProgramData\(LOGFILE_NAME)` which we'll ingest into Log Analytics Workspaces. 

### Ingesting Logs into LAW
Next, make a new log file on your desktop, then copy and paste the contents created in the log file found in `C:\ProgramData\(LOGFILE_NAME)` on your VM. Head back to Azure LAW and click on `Tables` on the left then create `New Custom Log (MMA-Based)` and add the log file.
<p align="center">
 <img src="https://i.imgur.com/xHJbEOj.png" height="80%" width="80%"/>
<br>

Now click `Logs` on the left and type in the name of the log file we just added, and run it to see if it works.
<p align="center">
 <img src="https://i.imgur.com/1lBgLna.png" height="80%" width="80%"/>



<h2> </h2> 

### Creating the Map
It's finally time to create our map so we can see where the attacks are coming from. Search `Sentinel` and add your machine, then click on `Workbooks` on the left then `Add Workbook`.
<p align="center">
 <img src="https://i.imgur.com/VZRXNIV.png" height="80%" width="80%"/>
<br>

Now we have to parse through the log to get the relevant information we need for the map. Go into the newly created workbook, then click `Add Query` and insert the following:
```
failed_rdp_geo_CL
| parse RawData with * "latitude:" Latitude ",longitude:" Longitude ",destinationhost:" DestinationHost ",username:" Username ",sourcehost:" Sourcehost ",state:" State ", country:" Country ",label:" Label ",timestamp:" Timestamp 
| summarize event_count=count() by Timestamp, Label, Country, State, Sourcehost, Username, Longitude, Latitude
```

Lastly, click on `Visualization` and choose `Map` then `Map settings` and configure it as follows:
<p align="center">
 <img src="https://i.imgur.com/0gOW323.png" height="80%" width="80%"/>
<br>

<h2> </h2>

### The Map
<img src="https://i.imgur.com/jbnV3cZ.png">

<h2> </h2>

### Video Tutorial
I have to give credit to [Josh Madakor](https://github.com/joshmadakor1) for coming up with the idea and creating the Powershell script. If you prefer to learn through videos, you can [find his tutorial here.](https://www.youtube.com/watch?v=RoZeVbbZ0o0)
