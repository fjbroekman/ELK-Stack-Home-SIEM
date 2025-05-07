# 🛡️ ELK Stack Home SIEM Lab

Welcome to my personal Security Information and Event Management (SIEM) lab built using the ELK Stack (Elasticsearch, Logstash, Kibana) along with Filebeat for log shipping. This project simulates a centralized logging environment for monitoring and analyzing system logs, making it a great practice setup for cybersecurity learning and blue team operations.


---

## 📸 Project Overview

This lab is designed for:

- Practicing log analysis and incident detection
- Understanding how logs flow from endpoints to a SIEM
- Building custom dashboards and visualizations
- Learning how to detect suspicious behavior using Kibana

---

## 🧰 Stack Components

| Component     | Description                               |
|---------------|-------------------------------------------|
| Elasticsearch | Stores and indexes log data               |
| Kibana        | Visualizes data and builds dashboards     |
| Filebeat      | Collects and ships logs from endpoints    |

---

## 🏗️ Architecture


1. Elasticsearch, Kibana and Filebeat is installed on a Linux host
2. Filebeat ships logs to Elasticsearch.
3. Kibana is used to analyze and visualize the collected logs.
4. Kibana is secured using Certificate Authorities.
   
---

## The Setup Begins
This is our first step into our SIEM-building journey. First things first, we need a VM to host our services. We'll be using VirtualBox for just that.

Once VirtualBox is set up, we can install an ISO for our VM; preferably Linux. I am running an **Ubuntu Server VM**


To set up the VM:
1. Open VirtualBox and Select `Machine` then `New`.
2. Give your VM a cool name and select the previously installed ISO image as your chosen OS.
3. Check the box to skip the unattented install. This is so we can tinker everything ourselves.
4. Adjust virtual hardware settings. I have my VM running 4GB of RAM, 2 CPU cores and ~50GB of SSD storage.
5. Start up your VM once everything is ready!

To set up the OS on our VM:
1. Select your language accordingly.
   
<br>

 ![Select a Language](https://github.com/fjbroekman/ELK-Stack-Home-SIEM/blob/main/Images/select-language.png)
 
 <br>
 
2. If you get an "installer update" notice, you can skip it as you wish.
3. <strong>Record the IP shown next to DHCPv4, we will need this!</strong>
4. We don't need a proxy, so we can skip configuring one. We also don't need an alternative mirror for Ubuntu, so leave everything there as is.
5. Continue setting up the storage config for the OS, there is no need to tinker with anything unless you want to
   
<br>

![Storage Conig](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/storage-config.png)

<br>

6. Confirm your user credentials and SSH setup, tick the OpenSSH Server Box. We don't need any featured software on our VM, so continue with the installation.
   
<br>

![Enter your credentials](https://github.com/fjbroekman/ELK-Stack-Home-SIEM/blob/main/Images/profile-setup.png)

<br>

7. Wait for the OS install to complete
   
<br>

![](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/waiting-for-install.png)

<br>

## Installing and Configuring Elasticsearch

Before we even get to starting anything, go into your VirtualBox menu and select the 3 bars next to your VM, then select `Snapshots`. Click `Take` and name the snapshot however you like. Congratulations, you have just implemented a failsafe in case you break your VM somehow. <strong>I strongly recommend you regularly take snapshots of your VM when you reach checkpoints in this guide.</strong>

Once you reboot your system, wait for your log statements to stop, then press Enter. Continue to logging in as your user. When you're in, your screen should look something like this:
![Voila](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/login-complete.png)

The last thing we need to do before getting to real SIEM setup fun is to update our system with the following commands:
```
sudo apt update && sudo apt upgrade -y
sudo apt dist-upgrade -y
sudo apt install zip unzip -y
sudo apt install jq -y
```

From here on out, we will be using SSH on our host machine instead of using VirtualBox.

## SSH Setup 
If you don't have ssh enabled by default, you can and should do the following:
<br>
1. Run `sudo systemctl status ssh` to see if SSH is even live on your system. 
2. If you see an error entailing `ssh.service` couldn't be found. Don't panic. We will simply reinstall OpenSSH so we can continue.
3. Run the following command: `sudo apt-get remove --purge openssh-server&& sudo apt-get update && sudo apt-get install openssh-server`.
4. If all is well, you can run `sudo systemctl status ssh` and see that SSH is currently a dead process. Run `sudo systemctl start ssh` to start it up! You can also run `sudo systemctl enable ssh` to just have it be a startup process.

Because I am such a nice guy, I have left a script on this repo that can do all of this for you :) 

<h2 align="center">Intermission: More SSH Setup</h2>
Fun fact, we can't just SSH and magically run on our VM. No, that'd be too easy. We actually have to do <strong>port forwarding</strong> to achieve this. Simply put, we make an SSH request on a certain port, which then gets sent to our VM at port 22; the port for SSH connections.

To achieve this, I created this rule by going to the `Settings` tab in VirtualBox and going to `Network` then `Port Forwarding`. ![Networking Page](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/network-settings.png).

From here, I make the following Port Forwarding rule:
![Port forwarding in VirtualBox.](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/forwarding-rule.png)
What this basically means is that whenever I send a TCP request to port 2222 on my machine, regardless of the IP used to specify the host, it will be forwarded to port 22 on the Guest IP, which is our DHCP IP address from before. <strong>You can (and should) change the Host IP value to something that isn't blank since that will allow any machine to access the VM effectively. You can specify 127.0.0.1 for the Host IP to only allow localhost/your machine to access the VM.</strong>

Moment of truth, we should be able to `ssh` given any IP address and on port 2222. Run `ssh -p 2222 SERVER_USERNAME@127.0.0.1` and hope for the best!
![This took me like 20-30 minutes.](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/ssh-success.png)

<h2 align="center">Continuing to setup Elasticsearch</h2>
<strong>The horrors persist but so do we, let's continue!</strong> If your VM is not running right now, go and start it up. We'll SSH in just like before and get straight to business.

Firstly, we will be installing `apt-transport-https`, which on its own enables APT transport to be done w/ HTTPS and not HTTP; slightly more secure stuff for our VM which could prove useful for production environments.

Run `sudo apt install apt-transport-https -y` and continue. Next we add the GPG key for Elasticsearch and its repo to our Ubuntu sources.

Run `wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -` to add the GPG key and `echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-7.x.list` to see the repo in our ubuntu sources. 
![GPG and repo added!](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/gpg-repo-success.png)

Success! Update our packages then install elasticsearch. Run `sudo apt update -y && sudo apt install elasticsearch -y`.

Now that Elasticsearch is installed, we need to reload all running daemons and services on our VM. Since we are adding a new service (Elasticsearch), this is necessary. It wasn't needed for SSH since we changed nothing with configuration files, and configuration is already handled by apt (to my knowledge) but for Elasticsearch, we will be modifying our configuration files.

We will want to confirm that Elasticsearch is actually on our system, so run `sudo systemctl status elasticsearch.service`; we want to see a 'dead' process.
![Yes I did a typo.](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/elasticsearch-success.png)

With that all said and done, we are officially finished the installation phase for Elasticsearch and will now get to configuring it.

<h2 align="center">Configuring Elasticsearch and Testing It Out</h2>
This step is entirely for configuring Elasticsearch; telling it how to run, what ports it needs, communication, etc.

Open `/etc/elasticsearch/elasticsearch.yml` in a text editor (neovim da goat). If you are using a terminal editor like me, run `sudo EDITOR_OF_CHOICE ELASTICSEARCH.YML PATH`. Firstly, we will edit the cluster.name property to be the name of your choice for your homelab. 
![Delete those pesky pound symbols.](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/edit-elasticsearch-config.png)

Continue editing the file. The `network.host` property will be our VM's IP address. The `http.port` property will be left alone, but make sure to uncomment it, and lastly, add the `discovery.type` property (this property tells Elasticsearch how it should form a cluster). <strong>Do not forget the colon next to the property name!</strong> 

Once we are done, go ahead and close the YML file. Congrats, our config work here is done (temporarily)! We can now go ahead and start the `elasticsearch.service` service, then check the JSON output of Elasticsearch at our IP address + HTTP port--you'll see what I mean :)

Run `sudo systemctl start elasticsearch` and wait a lil for the command to complete. We can also run `sudo systemctl enable elasticsearch` to automatically start the service on boot of our VM, if you wish. Once the command completes, run `curl --get http://NETWORK_HOST_IP:9200`. NETWORK_HOST_IP is the network.host IP we have in our config file!
![We got something!!!](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/elasticsearch-curl-success.png)

Congratulations, we just set up Elasticsearch and can confirm that a cluster is up and running. Port 9200 is where we will send all of our logged data to; from Beats or Elastic Agents if you go down that route. Our next step is setting up Kibana in the same manner.

If you ever want to edit the `elasticsearch.yml` configuration on your own time, <strong>you will need to run `sudo systemctl restart elasticsearch.service` so that the updated configuarion is read by Elasticsearch.</strong>

<h2 align="center">Configuring Kibana</h2>
Configuring Kibana starts with installing it; run `sudo apt install kibana -y`.

Next we will update the YML configuration of Kibana just like we did with Elasticsearch. In my case of running Vim as my editor, run `sudo vim /etc/kibana/kibana.yml`.

We will leave the `server.port` property alone as port 5601 is the default option and we have no need to edit it. We will edit the `server.host` and `server.name` properties to be our IP address and an appropriate name respectively. One thing you may notice is that you can adjust the credentials for accessing the Kibana server. You can leave them blank for now, as we will change them later on.
![Our edits.](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/kibana-config.png)
![Default credentials are a big nono!](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/kibana-nono-security.png)

Save and exit our changes to the YML file and go ahead and start up the Kibana service. If you're like me and have Elasticsearch not enabled in systemd, you will need to start it up in the following order:
1. Start Elasticsearch w/ `sudo systemctl start elasticsearch` if not already started.
2. Start Kibana w/ `sudo systemctl start kibana`. We can also make Kibana run on boot w/ `sudo systemctl enable kibana`.

Once both Elasticsearch and Kibana are running, we will go ahead and check the status of both services by running `sudo systemctl status elasticsearch && sudo systemctl status kibana`.
![Both services running just fine.](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/elastic-kibana-success.png)

That "legacy OpenSSL" message thing is not an issue. According to ChatGPT (yes you heard that right), this is simply for backwards compatibility measures involving communication with Kibana's Node.js runtime. Basically, we do not care in the slightest. 

Kibana and Elasticsearch are now up and running and all that's left is Beats for ingestion. To quickly go over Beats, we will actually be using <strong>filebeat</strong>, which is a kind of Beat offered by Beats (wowza). [Here is a link for further reading on other beats you can use.](https://www.objectrocket.com/resource/what-are-elasticsearch-beats/) For our purposes, filebeat will be just fine to get going and testing our environment. However, I will personally guide you through installing <strong>Winlogbeat</strong> to create our very own mock attack inspired by <strong>chroma</strong> on crin's server; Sally running PowerShell.

If you're closing down like me for the night, you can run `sudo systemctl stop kibana` and `sudo systemctl stop elasticsearch` <strong>in that order</strong> to safely close down our active services :)

<h2 align="center">Configuring Filebeat</h2>
Install filebeat by running `sudo apt install filebeat -y`. As seen withboth Kibana and Elasticsearch, we will need to edit filebeat's YML file; `/etc/filebeat/filebeat.yml`, which I will go ahead and open with vim as I have been doing thus far.

Here's where things get interesting for us. We can add a TON of different inputs for filebeat, ranging from `journald` logs to yes, `winlog input`. We can actually get Windows event logs using filebeat. Simply put, `Winlogbeat` captures far more log data with Windows operations past just logged events. Event logs are available from `winlog input` but also `Winlogbeat`. For simplicity sake, we will use filebeat instead of Winlogbeat for our mock attack, and you will see why once I get to describing the attack :)

Therefore, we will make the following edits:
1. Add a new configuration to the `filebeat.inputs` field. This `winglog` input will require Script Block Logging on our Windows machine to work effectively, but we will configure that soon enough. The event IDs 4104 (script logging), 4105 (powershell cmd executed) and 4106 (powershell cmd complete) are all going to be very nice to have assuming an attack via PowerShell! <strong>This only works for PowerShell 5.1. If PowerShell is any greater version we need to change the `name` property to `PowerShellCore/operational` in accordance to MS Documentation.</strong>
![Winlog input to add alongside base inputs.](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/winlog-config.png)
2. Configure the Elasticsearch portion of Filebeat. Kibana's config is totally fine. All we need to do is change the IP of the Elasticsearch host.
![Elasticsearch-Filebeat configuration.](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/elastic-filebeat-config.png)

That is all we need to do. Our next step is to literally set up Filebeat in the terminal. This will manage how index management w/ Elasticsearch works, on top of disabling Logstash for log ingest.

Run `sudo filebeat setup --index-management -E output.logstash.enabled=false 'output.elasticsearch.hosts=["ELASTICSEARCH_HOST_IP_ADDRESS:9200"]'`. ELASTICSEARCH_HOST_IP_ADDRESS is the host IP address we chose in our elasticsearch.yml file. Index management manages how Elasticsearch goes about its indexing business, which I will not go over since that'd take 10 krillion years. `-E` overwrites specifc config settings, which we do next by 1) Disabling Logstash and 2) Ensuring we have our Elasticsearch host correctly set because why not.

Once that's over we can actually start everything up in the following order if NOT ALREADY STARTED:
1. Start Elasticsearch: `sudo systemctl start elasticsearch`
2. Start Filebeat: `sudo systemctl start filebeat`
3. Start Kibana: `sudo systemctl start kibana`
![Everything looks good!](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/all-services-running.png)

Assuming nothing blew up, we can now run one last `curl` to actually check if Elasticsearch is getting something from Filebeat. Run `curl --get http://ELASTICSEARCH_HOST_IP_ADDRESS:9200/_cat/indices?v`. This command may look insane, but it's relatively simple:
1. `_cat/indices` is the CAT Indices API. Simply put, it is an API that returns high level information about indices in an Elasticsearch cluster. We are making an API call with this URL.
2. `?v` = verbose. Gives column headers to output and makes it more readable. 

Now if you're a complete dumby like me, and forgot to run `sudo filebeat setup`, then we're DOOMED! Not. Just shutdown filebeat, run the setup command, and restart filebeat with the following order:
1. `sudo systemctl stop filebeat.service`, <strong>this took a while so do NOT Ctrl+C and possibly ruin your day.</strong>
2. `sudo filebeat setup --index-management -E output.logstash.enabled=false 'output.elasticsearch.hosts=["ELASTICSEARCH_HOST_IP_ADDRESS:9200"]'`
3. `sudo systemctl restart filebeat.service` && `sudo systemctl status filebeat.service`
![IT WORKS!???????????????!](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/filebeat-success.png)

That yellow healthcheck notice is nothing to worry about. To my knowledge, it is indicating that replica shards are unassigned, which basically means we are at risk of data loss, which we don't care about in our current state. No only that, we have no primary shards, since we have no indexed data, so <strong>we do not care.</strong>

With that being said, we're done here! To shut everything down, run:
1. `sudo systemctl stop filebeat.service`, <strong>this took a while so, once again, do NOT Ctrl+C.</strong>
2. `sudo systemctl stop kibana.service`
3. `sudo systemctl stop elasticsearch.service`

If you aren't leaving, then we'll get going to the next step; establishing a CA.

<h2 align="center">Establishing Elastic Services IPs and Understanding Certificate Authorities</h2>
Our first step with making a CA is to understand why we are doing it. We are making a CA so we can implement [TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security), thus HTTPS into our Elastic services, and that requires certificates.

Firstly, we need to specify the instances of our services so we can later grant them a certificate. Navigate to `/usr/share/elasticsearch` and create a file called `instances.yml` by running `sudo touch instances.yml` and then opening it in a text editor. This file will hold information regarding what Elastic components we are using and their host IPs.
![Creating the file in the directory.](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/creating-instances-yml.png)

Great stuff, let's add the following text that I will not copy because it is too tiresome:
![instances.yml configuration.](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/instances-yml-config.png)

Replace the `10.0.2.15` with your appropriate host IP. You may be wondering what that `fleet` service is; it is for managing your fleet of Elastic Agents, which themselves collect all sorts of information past log data. Simply put, they are fundamental if you wish to set up Elastic Agents later down the line.

Save the file and exit our text editor. Our next step is to literally create a PKI (Public Key Infrastructure) for ourselves. To do so, we will need keys and certificates, and of course, a certificate authority, which we will make using Elasticsearch's utilities.

<h2 align="center">Creating the CA and Generating Certificates</h2>
Navigate to `/usr/share/elasticsearch` and run the following: `sudo /usr/share/elasticsearch/bin/elasticsearch-certutil ca --pem`. Elasticsearch-certutil will aid use with creating basic [X.509](https://www.techtarget.com/searchsecurity/definition/X509-certificate) certificates as well as signing them. The `ca --pem` bit will 1) Create a new CA and 2) Create the CA certificate and private key in PEM format, as the command output will say. <strong>Leave the option for the output file blank; simply click Enter and continue.</strong>
![Running the command and seeing the output!](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/elastic-certutil-success.png)

Awesome sauce. We now have a zip file of our our CA certificate and private key. Unzip the zip folder with `sudo unzip ./elastic-stack-ca.zip`. You should have a `ca/` directory with a certificate file and private key. Our next step is to generate the certificate to sign for each of our instances.

Generate the certificate with the following command: `sudo /usr/share/elasticsearch/bin/elasticsearch-certutil cert --ca-cert ca/ca.crt --ca-key ca/ca.key --pem --in instances.yml --out cert.zip`. Let me make this shrimple to understand:
1. We use the certutil and assign the CA certificate and private key from our unzipped files.
2. Everything is set to the PEM format, given the `--pem` option.
3. `--in instances.yml` is going to effectively use each instance specified in `instances.yml` and generate its certificate, private key and the CA certificate.
4. Output the final compressed zip with the name `certs.zip`.
![Creating our certificates for each instance.](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/creating-instances-certificates.png)

The command output on its own does a great job giving a ton of documentation, which is awesome. We'll unzip the zip folder and then make a new directory `certs` solely to organize our goodies. Run `sudo unzip certs.zip` then `sudo mkdir certs`. We'll move all of our cert files into this directory with the following commands:
1. `sudo mv /usr/share/elasticsearch/elasticsearch/* certs/`
2. `sudo mv /usr/share/elasticsearch/kibana/* certs/`
3. `sudo mv /usr/share/elasticsearch/fleet/ * certs/`

Next, we will create directories to store the CA certificate and private key for each of our services:
1. `sudo mkdir -p /etc/elasticsearch/certs/ca`
2. `sudo mkdir -p /etc/kibana/certs/ca`
3. `sudo mkdir -p /etc/fleet/certs/ca`

Then we copy the private key and CA certificate:
1. `sudo cp ca/ca.* /etc/elasticsearch/certs/ca`
2. `sudo cp ca/ca.* /etc/kibana/certs/ca`
3. `sudo cp ca/ca.* /etc/fleet/certs/ca`

And finally, copy the service sertificates and keys we generated in the beginning to each `certs` directory:
1. `sudo cp certs/elasticsearch.* /etc/elasticsearch/certs/`
2. `sudo cp certs/kibana.* /etc/kibana/certs/`
3. `sudo cp certs/fleet.* /etc/fleet/certs/`

Last thing to do before moving on is to clean up our now empty directories as we have moved their contents to the above new folders: `sudo rm -r elasticsearch/ kibana/ fleet/`. 

<h2 align="center">Proper Security for SIEM Deployment</h2>
Fun fact, we are using `sudo` a fair bit to access stuff on our system given escalated privileges through authentication. Simply put, we input out password to edit most of our configs, since this is low level stuff yes? 

What if we hate using `sudo` and just want to edit everything? Well, we just use the root account all the time, then we don't need to worry about authenticating! <strong>No, no, no, no!!!!</strong> Bad! We want to employ least privilege on everything. As someone studying cybersecurity and even speaking with current analysts, systems are VERY susceptible to attacks; friggin' [PrintNightmare](https://nvd.nist.gov/vuln/detail/cve-2021-34527) everywhere. 

What we are going to do is just that; employ least privilege on our systems to each of our services to both 1) Enhance our security posture and 2) <strong>Practice the task of assigning least privilege.</strong>

We'll begin by navigating to `/usr/share` and run the following commands:
1. `sudo chown -R elasticsearch:elasticsearch elasticsearch/`, which will 1) Recursively set the ownership of the `elasticsearch` directory to Elasticsearch and 2) Change its group to the `elasticsearch` group, which will further allow us to limit the scope of Elasticsearch's permissions.
2. `sudo chown -R elasticsearch:elasticsearch /etc/elasticsearch/certs/ca`. Same stuff as above but now for Elasticsearch's CA copies, so it can access that directory as well.

<strong>Now I cannot make myself any clear that you must NOT do this `chown` privilege restriction to Kibana FOR ANY REASON. Kibana needs access to its CA files to serve the SIEM frontend in your web browser.</strong>

Now, to ensure our posture is correct and our certificates are valid, we will use the `openssl` command to print certificate information to the console with the following command: `sudo openssl x509 -in /etc/elasticsearch/certs/elasticsearch.crt -text -noout`, which will: 
1. Print information for X509 certificates.
2. `-in` specifies the certificate to check out
3. `-text`  ouputs everything as human-readable.
4. `-noout` suppresses the output of the encoded version of the certificate; only shows the human-readbale output above.
![Success.](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/elastic-chown-success.png)

<h2 align="center">Configuring The Services To Run HTTTPS w/ SSL Certificates</h2>
Now we are effectively ready to put everything together, and enable SSL on all of our services, thereby enabling HTTPS. 

Our first step is to go configure the `kibana.yml` configuration file in `/etc/kibana/` using a text editor, where we will copy the following massive text wall and paste it to the <strong>bottom</strong> of our YML file:

```
server.ssl.enabled: true
server.ssl.certificate: "/etc/kibana/certs/kibana.crt"
server.ssl.key: "/etc/kibana/certs/kibana.key"

elasticsearch.hosts: ["https://10.0.2.15:9200"]
elasticsearch.ssl.certificateAuthorities: ["/etc/kibana/certs/ca/ca.crt"]
elasticsearch.ssl.certificate: "/etc/kibana/certs/kibana.crt"
elasticsearch.ssl.key: "/etc/kibana/certs/kibana.key"

server.publicBaseUrl: "https://10.0.2.15:5601"

xpack.security.enabled: true
xpack.security.session.idleTimeout: "30m"
xpack.encryptedSavedObjects.encryptionKey: "min-32-byte-long-strong-encryption-key"
```

Fun fact, these are all properties inside of our YML file, but they are commented out, and addressing you to edit each and every property would be a pain. Some of these are very self explanatory and others are not the case. The `server.publicBaseUrl` property is our IP and Kibana port number (5601), which specifies where Kibana is available; basically our SIEM UI. The `xpack` bits are all for securing Elasticsearch. The `xpack.encryptedSavedObjects.encryptionKey` property is for creating an encryption key to encrypt and decrypt sensitive Kibana entities like dashboards, alerts, etc. We use the default specified by [Elastic](https://www.elastic.co/guide/en/kibana/current/xpack-security-secure-saved-objects.html), but if you guys find docs on this, PLEASE send them to me because I cannot find many examples for this property.

The rather confusing matter in all of this is specifying the `elasticsearch.ssl.key/certificate` properties with our kibana key/certificate. Initially, I was very confused on why we (the LevelEffect Guide linked in Sources) was using the Kibana certificate and key here, but the answer is LITERALLY in the YML file comments. We are SENDING the Kibana certificate and key TO Elasticsearch to VERIFY our identity as Kibana. That is it. And here I was, scrabbling in the dirt for 10 minutes trying to find an answer xD

Anyways, remember to replace the `10.0.2.15` with your host IPs from before. Save and exit, then open the `elasticsearch.yml` configuration file in `/etc/elasticsearch/` and paste the following to the bottom of your YML file:

```
xpack.security.enabled: true
xpack.security.authc.api_key.enabled: true

xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.key: /etc/elasticsearch/certs/elasticsearch.key
xpack.security.transport.ssl.certificate: /etc/elasticsearch/certs/elasticsearch.crt
xpack.security.transport.ssl.certificate_authorities: ["/etc/elasticsearch/certs/ca/ca.crt"]

xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.verification_mode: certificate
xpack.security.http.ssl.key: /etc/elasticsearch/certs/elasticsearch.key
xpack.security.http.ssl.certificate: /etc/elasticsearch/certs/elasticsearch.crt
xpack.security.http.ssl.certificate_authorities: ["/etc/elasticsearch/certs/ca/ca.crt"]
```

For the sake of length and explanation, I will leave [a link to documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-settings.html#token-service-settings) for you to read. The summary for these settings is that we are enabling a gajillion security features and settings for our setup, particularly to use SSL.

You will 110% notice that we have no quotation marks with each filepath here, and that is, in fact, the correct method to display them here. I believe this is simply a matter of parsing behind the scenes, but it is undeniably a <strong>terrible one.</strong> The `certificate_authorities` list needing quotation marks makes sense, but wadahek Elastic? Were the devs chugging Red Bull while making the backend for this?

Save and exit as usual, and we are ready to load our new configurations to officially enable SSL for our services. If you recall from before, we need to restart our services to enable new configurations by running `sudo systemctl restart <service_name>`. `restart` will both start the system if it is not running and restart it, so we can go ahead and do just that for both Elasticsearch and Kibana:

1. `sudo systemctl restart elasticsearch`
2. `sudo systemctl restart kibana`

If you're not like me and had a typo in their `elasticsearch.yml` config and panicked whem everything blew up in my face for like 10 minutes (I forgot to specify the SSL CERTIFICATE BRO), you should have everything running just fine. Run `sudo systemctl status <elasticsearch or kibana>` if you wanna check the status of either service, though we can be certain they are running at the moment.

We don't need `filebeat` running for this next step. Our last check to ensure everything is in order is to `curl` the HTTPS host for Elasticsearch, which in my case is: `curl --get https://10.0.2.15:9200`. Replace your IP as needed, anddd....
![Erm... What?](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/curl-fail.png)


<strong>Noooooooooooo!</strong> Firstly, don't quit easy like that. Secondly, the reason why `curl` failed like this is because our browser does not trust the certificate just yet, or rather, the CA. We can bypass this temporarily and just add the `--insecure` parameter to end of our `curl` command and pipe the output to `jq` to make it nice and pretty as JSON format: `curl --get https://10.0.2.15:9200 --insecure | jq`.
![Yippee!](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/curl-success.png)

Nice, we at least got something! To summarize this output, we require credential to log into Elasticsearch, which we will generate in the next step of our guide. But for now, go ahead and pat yourself on the back; we've got SSL running and security enabled! One step closer to a real homelab setup. We can stop our services by running `sudo systemctl stop kibana` then `sudo systemctl stop elasticsearch`. 

<h2 align="center">Generating Authentication Credentials For Elasticsearch</h2>
To generate credentials, we will use the [elasticsearch-setup-passwords](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-passwords.html) utility to do so. Documentation is hyperlinked as always for your reading. Simply put, the utility will generate passwords for a cluster and connect via HTTPS using our previously defined `xpack.security.http.ssl` in our `elasticsearch.yml` file. 

Run `sudo /usr/share/elasticsearch/bin/elasticsearch-setup-passwords auto` to randomly generate passwords, set the passwords of select users, and output them to the console. Make sure Elasticsearch is running before you do this or else you will get an error. You will see many passwords and names associated with them. For example, `apm_system` is the [APM or Application Performance Montioring System](https://www.elastic.co/guide/en/observability/current/apm.html), which, per its name, collects performance metrics for services and applications running in real-time, and is built on the Elastic Stack. More information about these users can be collected here by [Elastic's official documentation.](https://www.elastic.co/guide/en/elasticsearch/reference/current/built-in-users.html)

We only need the `kibana_system` password as Kibana authenticates through Elasticsearch and thus grants access to the frontend UI. <strong>However,</strong> we will need to create our own user with prvilieges to login to the SIEM frontend after we initially login using the <strong>`elastic` superuser</strong>. It's credentials will be `elastic` as its username and `<generated_password>` from the previous step when we generated new passwords.

Open the `kibana.yml` file in `/etc/kibana/` with a text editor and find the `elasticsearch.username` and `elasticsearch.password` properties. Change the `elasticsearch.password` to the password we just generated and <strong>DO NOT CHANGE THE `elasticsearch.username` PROPERTY. You will get error logs back when the server attempts to launch to host the frontend!</strong>

Save and exit our YML config file and restart Kibana so that our new configuration settings are read properly with `sudo systemctl restart kibana && sudo systemctl restart elasticsearch`.

And with that, we are ready to log into our SIEM frontend for the first time!

<h2 align="center">Logging into Elastic</h2>
Firstly, start up Elasticsearch and Kibana if you haven't already with `sudo systemctl start elasticsearch`, `sudo systemctl start kibana` and `sudo systemctl start filebeat`. The fun part starts now.

Secondly, we can't actually access our frontend just yet. We actually need to create a port forwarding rule, Fun fact, `10.0.2.15:5601` is hosted on the SERVER, but not publicly accessible to US. We need to redirect, or otherwise, <strong>forward</strong>, the traffic to us so we can recieve it.

Open your Network settings as we did long ago to configure SSH's port forwaring. Create a new rule with the little 'Plus' icon on the right with the following specifications:
- Protocol: TCP
- Host IP: 127.0.0.1 
- Host Port: 5601
- Guest IP: `YOUR_KIBANA_IP`
- Gust Port: 5601

`YOUR_KIBANA_IP` is the IP you are hosting the Kibana server on, which is shown in both Kibana's log files and the `kibana.yml` configuration. Once you've created the rule, you can attempt to head to your frontend in your web browser at `https://127.0.0.1:5601` if your VM is powered on and your services are running.
![Waiting....](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/frontend-loading.png)

The loading may take a while; just be patient...
![A FRONTEND????????????](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/frontend-loaded.png)

Now we login using our <strong>`elastic` superuser credentials</strong> and NOT the `kibana_system` user.
![IT WORKED!](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/elastic-login-success.png)

Many hours, days, a VM restart, and many sweaty, hair-pulling moments have finally led both me and <strong>you</strong> to this moment! We have a frontend!

<h2 align="center">Creating Our Own User</h2>
Click the `Explore on my own` button to continue to the Elastic UI. Once we're in, our sole goal is to create an admin user for ourself, so that we don't need to log in to the `elastic` superuser every time.

Given the steps in the [Official Elastic Docs](https://www.elastic.co/guide/en/kibana/8.17/using-kibana-with-security.html), we will go to the Roles management page and create a new user with the `kibana_admin` role; to grant us admin access to the Kibana; by extension, the SIEM frontend.

Scroll down and click the `Manage Permissions` button. On the sidebar to the left, Click `Users` under `Security`. This is where we want to be.
![User page!](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/users-page-success.png)

Click the `Create User` button, and set up the credentials as needed. <strong>For privileges, select the following roles:</strong>
- beats_admin
- kibana_admin
- superuser, to access Fleet Management

More information about privileges can be found [here in the Elastic Docs!](https://www.elastic.co/guide/en/elasticsearch/reference/current/built-in-roles.html) To ensure that our account actually works, we will attempt to login to it on the Elastic login page. Logout then log back in using our new user.
![Nice!](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/new-user-success.png)

We are now ready to move on! If we ever need to modify our account privileges, we will simply login using the `elastic` superuser again and modify our roles as needed.

<h2 align="center">Understanding Fleet and Creating a Fleet Server</h2>
Open the hamburger menu of the left of our UI and scroll to `Management` and click `Fleet`. 

Fleet management will take some time to load, but once it is, we will configure our fleet server with the following options:
![My fleet config.](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/fleet-config.png)

Click the `Add host` button and then generate our service token. I recommend taking a photo or saving this token SOMEWHERE because yes, you WILL forget it. Our next step is to actually get an Elastic Agent set up with Fleet.

In your VM, `cd` into `/etc/fleet` and run the following command: 

`sudo wget https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-7.17.6-linux-x86_64.tar.gz`

Given that this is a tarball, we will need to extract its contents with the follwoing: 

`sudo tar -xvf ./elastic-agent-7.17.6-linux-x86_64.tar.gz`

Once that's done, we can delete the original tarball if we wish. Return back to our frontend. `cd` into `elastic-agent-7.17.6-linux-x86_64`. Now it's time to run the below commands to start our Fleet server, as shown on the frontend:

```
sudo ./elastic-agent install --url=https://10.0.2.15:8220 \
  --fleet-server-es=https://10.0.2.15:9200 \
  --fleet-server-service-token=YOUR_SERVICE_TOKEN \
  --fleet-server-policy=499b5aa7-d214-5b5d-838b-3cd76469844e \
  --certificate-authorities="/etc/fleet/certs/ca/ca,crt" \
  --fleet-server-es-ca="/etc/elasticsearch/certs/elasticsearch.crt" \
  --fleet-server-cert="/etc/fleet/certs/fleet.crt" \
  --fleet-server-cert-key="/etc/fleet/certs/fleet.key"
```

I recommened pasting that entire command into a separate notepad and editting it; change the IP to match your Fleet server IP that we set earlier as well as the Elasticsearch IP and input your service token from before to replace `YOUR_SERVICE_TOKEN` in the above command. Paste the entire command into your terminal and hit Enter!
![Yatta!](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/elastic-agent-success.png)
![Yippee!](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/fleet-server-success.png)

I had to restore my VM many times to snapshots because I made various typos that invalidated the server setup, but still installed the Agent. Simply put, I couldn't revert anything. <strong>Don't do that :)</strong>

You will notice that we are able to upgrade our Fleet server, however I am not sure if that requires upgrades to our other services at this time. You may go ahead and attempt to upgrade the service but make snapshots and take your time!

Let's continue on now. Click the `Fleet Settings` button in the top right corner of the UI. Change the Elasticsearch host from `localhost` to your Elasticsearch host IP. Save and Apply our changes and we're good to go to our next step of Agent Policies.

If you do wish to shut down your server. You can do so with the following steps:
1. `sudo systemctl stop filebeat.service`
2. `sudo systemctl stop kibana.service`
3. `sudo systemctl stop elasticsearch.service`
4. `sudo systemctl stop elastic-agent`

**I have noticed that sometimes, the `stop` command will timeout and fail when attempting to stop the `elastic-agent` service. I am not sure why this occurs. If anyone does know, do let me know! To my knowledge, it has no impact on anything; nothing has broken *yet*.**

<h2 align="center">Creating Agent Policies To Collect Logs</h2>
Navigate to the `Fleet` menu and click `Agent Policies`. Recall that agent policies will define what logs we want our agents to collect. To my knowledge, the Elastic Agent(s) **work with Filbeat** to get log data specified. Both the Filebeat configurations and agent policies work in tandem for log ingestion.

Click `Create Agent Policy`. For the sake of demonstration, we will create a basic policy encompassing Windows Endpoints, since we will be using a Windows VM down the line for our mock incidents.
![Policy configuration.](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/policy-config.png)

Once your policy is created, click on it and select `Add Integration`. This is where we get to finally have some fun tweaking how our agents will work! What we're interested in is the `Security` section, but feel free to scroll and check out all the options at your disposal. What we **really** want is the `Endpoint Security` integration. Click on it once you've found it.

Click `Add Endpoint Security` and configure the integration as you please. Click `Save and continue` once your finished.
![Integration configuration!](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/integration-config.png)

Once your configuration is saved, you'll get a popup to add Elastic Agents to your host machines. Click `Add Elastic Agent later`; we will handle this soon! Before we continue on, get to **this** menu and click on your `Endpoint Security` integration's name. 
![This lovely thing.](https://github.com/nubbsterr/ELK-SIEM-Setup/blob/main/screenshots/agent-policy-config.png)

Scroll to the bottom and click `Show advanced settings`. Scroll until you find `windows.advanced.elasticsearch.tls.ca_crt`. This property will house the public CA certificate for elasticsearch on our endpoint(s). I have set the directory for this to be `C:\Windows\System32\ca.crt`; we'll handle getting the cert copied from our server VM to the Windows endpoint in a later step and will require us to tinker with our VM settings. For now, simply save and continue.

Next, we will add the `Windows` integration. Yes, that's its name. Add it just as we did before. The integration itself will log all sorts of behaviour but particularly **Script Block Logging** through the `PowerShell Operational` event log channel, which will log any and all commands ran in PowerShell, provided we have it enabled on our endpoint(s); we'll worry about this when it matters.

Click `Save and continue` then click `Add Elastic Agent later` once more. With all that said and done, we are done tweaking our Agents!
