## How to set up a lemmy instance in a cloud server

You can find Lemmy's official documentation at the following site:
https://join-lemmy.org/docs/en/about/about.html

(1) Choose the name of your demain and register it with a domain registrar (example: namecheap.com). Approximate cost: between 1 and 10 euros for one year.

(3) Rent a cloud server running Ubuntu (20.04 x64) which you are able to access via SSH (example: serverspace.io), cost: between 5 and 20 euros per month. I recommend you to start with a modest configuration and you can upgrade the server later (for example, start with: 1 GB ram, 30 GB disk, 10 Mbps network).
Write down your server's IP address.  

(4) Go back to the website where you registered your domain and find the DNS settings. Add an "A record" for "@" pointing at your server's ip address, as well as an "A record" for "www" pointing at that same IP. This will redirect your IP address to the server. Note: Changes to the DNS settings can take several minutes to become effective.

(5) Access your server using SSH in the terminal (or, if using Windows, use putty). 

```
ssh root@server-ip-address
```

Once inside the server,  update and upgrade apt and install vim (or your favorite text editor), so that you can modify the configuration files.

```

apt -y update && apt -y upgrade

apt -y install vim
```


(6) Install docker and docker compose. Docker is an application that allows one to create virtual containers automatically by using a set of instructions described in a file. The developers have written these files for Lemmy, so we will use this program later on to build the instance.

The instructions on how to install docker are documented here:
https://docs.docker.com/engine/install/ubuntu/

And for docker compose, here:
https://docs.docker.com/compose/install/

These are the commands that you will have to run to install docker and docker compose in your server:

```
apt -y remove docker docker-engine docker.io containerd runc

apt -y install ca-certificates curl gnupg lsb-release
 
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg 
 
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null 
 
 
apt -y update
 
apt -y install docker-ce


curl -L "https://github.com/docker/compose/releases/download/v2.2.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

chmod +x /usr/local/bin/docker-compose


```

(7) Install NGinx. When someone connects to your server via the browser, this is the engine that specificies how the connection will be handled . [https://nginx.org/en/]

```
apt -y install nginx
```

To test that the engine is working properly, try the following:

```
sudo systemctl enable nginx

sudo systemctl start nginx

sudosystemctl status nginx
```

There is a chance that you see an error similar to the following:

```
nginx: [emerg] socket() [::]:80 failed (97: Address family not supported by protocol)
```

This happens because the default server tries to listen for a connection through an IPv6 address, but your server only has an IPv4 IP address. We can solve this by changing the configuration file of the default server.

Open the file:

```
/etc/nginx/sites-enabled/default
```

And comment the following line:

```
#       listen [::]:80 default_server;
```

Now try to start nginx again by running:

```
sudo systemctl start nginx
```

Hopefully this returns no error. If you still see an error, you can try to run the status command once again and look up the error in a search engine.

```
sudo systemctl status nginx
```
 


Once the nginx is running, you use the browser to go ito your domain. You should see the following page, indicating that the connection to the server is being handled appropriately:

![Pasted image 20211211002801](https://user-images.githubusercontent.com/81911574/145698065-7d0bee26-ff54-4d39-8755-dea60d78f867.png)

If you don't see this page:
- The DNS could be setup incorrectly.
- The A records might not have been updated yet. You can try to ping the domain and see if it pings the server's IP address. You can also try deleting your browser's cache or opening a new anonymous window, and try again.
- You may need to manually configure the port forwarding. If you are self-hosting at home, port forwarding will be necessary. For cloud servers the port forwarding is usually not necessary 

(8) We now need to install certbot. This program will allow us to generate the SSL certificates, which are necessary to establish a secure HTTPS connection. This step is necessary to enable federation between instances later on.

Certbot is not available via apt, so we must install the snap package manager before installing it.

```
apt -y install snapd
snap install --classic certbot 
```

(9) We will now generate the SSL certificates.

To ensure that everything is properly configured for the certification, first attempt a dry run using the following command. Make sure to replace "your-domain.xyz" with your domain.

```
certbot certonly --nginx --dry-run -d your-domain.xyz,www.your-domain.xyz  
```

 If the dry run is succesfull,  you may run the command again with the ¨--dry-run¨ option removed to actually generate the certificates.

```
certbot certonly --nginx -d your-domain.xyz,www.your-domain.xyz  
```

This should generate the SSL certificates, which will be saved by default at:

/etc/letsencrypt/live/your-domain.xyz


(9) We must now configure nginx so that connecting to our server via the browser will connect the client to the lemmy instance.

Move to the "sites-enabled" folder of nginx, which contains the server configuration of the different sites. Any file saved in this location will be appended to the default "/etc/nginx/nginx.conf " configuration.

```
cd /etc/nginx/sites-enabled
```

An example of an nginx server configuration can be found in Github at the following URL: 
https://github.com/LemmyNet/lemmy-ansible/blob/main/templates/nginx.conf#L63
 
 We will download this configuration file and rename it as "lemmy.conf" using the following command:

```
wget https://raw.githubusercontent.com/LemmyNet/lemmy-ansible/main/templates/nginx.conf -O lemmy.conf
```

Open the file with a text editor (such as vim) 
```
vim lemmy.conf
```

We must replace the variables that are found in double brackets {{}}. The variable including the brackets should be replaced. Remember to replace your-domain.xyz with your actual domain.

```
{{domain}} => your-domain.xyz
{{lemmy_port}} => 8536
{{lemmy_ui_port}} => 1235
```

If you don't want to do this manually, you can also replace the variables automatically using sed by running the following command, again replacing your-domain.xyz with your domain.

```
sed 's/{{domain}}/your-domain.xyz/g;s/{{lemmy_ui_port}}/1235/g;s/{{lemmy_port}}/8536/g' lemmy.conf -i
```


If during the nginx configuration you had an issue because of the lack of IPv6 address, you also need to comment these two lines:
```
#    listen [::]:80;

#    listen [::]:443 ssl http2;


```

Save the changes.

Try restarting the nginx engine by using the following command. 

```
systemctl restart nginx
```

There should be no output when you do this. If you see an error output, run 

```
systemctl status nginx
```

to troubleshoot.

(10) Setting up Lemmy

You will now create the folder in which Lemmy will reside. For example, you can do so at /var/www/your-domain, and move into it

```
mkdir /var/www/your-domain


cd /var/www/your-domain

```


Lemmy's github contains a folder called "docker" (https://github.com/LemmyNet/lemmy/tree/main/docker). This folder contains an example configuration file called "lemmy.hjson".

There are also several sub-folders that contain the files that contain specific instructions for docker to install Lemmy. I recommend you to install the  current "production" version of lemmy, which is stored in the "prod" folder.  The file that you will need to download from this folder is called "docker-compose.yml"

Download these two files by running these two commands:

```

wget https://raw.githubusercontent.com/LemmyNet/lemmy/main/docker/lemmy.hjson

wget https://raw.githubusercontent.com/LemmyNet/lemmy/main/docker/prod/docker-compose.yml


```

You also need to create a folder that will hold the database and the images that are uploaded to your instance. You need to change the ownership of the 'pictrs' folder so that users are able to upload and interact with these files via their browser.

```
mkdir -p volumes/pictrs

chown -R 991:991 volumes/pictrs

```


In docker-compose.yml, you should change "localhost" to your domain name in the following line:
 
LEMMY_EXTERNAL_HOST=localhost:8536 => LEMMY_EXTERNAL_HOST=your-domiain.xyz:8536

You can change the username, name, and password for the postgres database, and control the version of lemmy and lemmy-ui that is installed (you can change this file later to update lemmy as new version come out).

Save the changes that you have made to this file, and open "lemmy.hjson"

In this file you can setup the administrator account, change "hostname" to "your-domain.xyz", and make sure to change the postgres dabase details so that they match the details that you configured in the docker-compose.yml file.


To enable federation, add this block above the  "slur_filter":

```
federation: {
        enabled: true
  }
```

This would enable indiscriminate federation. If you would like to have more control over who can interact with your instance, take a look at the official documentation on [Federation](https://join-lemmy.org/docs/en/administration/federation_getting_started.html)

Save the changes to the file. 

You should now be able to run your instance by running:

```
docker-compose up -d
```

Use the browser to go to your domain and enjoy your newly created Lemmy instance!

To stop the instance, you can use:

```
docker-compose down
```
