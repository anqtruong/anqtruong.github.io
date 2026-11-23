Pre-Project Requirements:
```
A VM in your pool on the course Proxmox cluster, cloned from a course template or installed  
from ISO  DONE
• A Linux server distribution with no desktop environment  DONE - Ubuntu live-server
• All work over SSH from your own computer DONE
```

**SNAPSHOT:**
To snapshot in Proxmox, go to the "Snapshots" page. Select "Take Snapshot", enter the desired name for the snapshot, and save it. The result looks like this:
![](Attachments/Pasted%20image%2020261121160106.png)

**Install Docker Engine (Ubuntu) (https://docs.docker.com/engine/install/ubuntu/)**

Install CA certificates for HTTPS download verification, as well as curl.
```
sudo apt update && sudo apt install -y ca-certificates curl 
```

Create directory for apt to keep third-party signing keys (owner can write, everyone else can read and enter the directory).
``` 
sudo install -m 0755 -d /etc/apt/keyrings  
```

Download Docker GPG to verify that downloads are from Docker.
```
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
-o /etc/apt/keyrings/docker.asc  
```

Add Docker repo to apt sources.
```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
| sudo tee /etc/apt/sources.list.d/docker.list
```

Refresh packages!
```
sudo apt update  
```

Install Docker Engine, Docker CLI, containderd, buildx, and compose.
```
sudo apt install -y docker-ce docker-ce-cli containerd.io \
docker-buildx-plugin docker-compose-plugin
```

**Post-Install Checks**

Do the following commands to check that everything installed successfully:

This command enables docker.
```
sudo systemctl enable --now docker  
```

This command checks if Docker is enabled, and will output "enabled" if so.
```
systemctl is-enabled docker # enabled  
```

This command checks that Docker works. If it does, it will send the message "Hello from Docker! This message shows that your installation appears to be working correctly." in the terminal.
```
sudo docker run --rm hello-world  
```

This command shows the Docker Compose version if it has been installed (the previous step's commands should have).
```
docker compose version
```

Optional: run docker without sudo (log out and back in after)
```
sudo usermod -aG docker $USER
```

These commands ensure Docker starts at boot. This makes sure our app survives *sudo reboot*.
```
sudo systemctl enable docker.service 
sudo systemctl enable containerd.service
```

I did all my file editing using nvim (Neovim, a text editor).

Install nvim:
```
sudo apt update && sudo apt install neovim
```

**Application**

My chosen applications were Uptime Kuma and Gotify for uptime and notifications.

Downloads Uptime Kuma's compose.yaml from the Uptime Kuma GitHub repo (https://github.com/louislam/uptime-kuma):
```
curl -o compose.yaml https://raw.githubusercontent.com/louislam/uptime-kuma/master/compose.yaml
```

Then, I nvim into that compose.yaml and add the config for Gotify. I used the sample docker compose yaml from https://gotify.net/docs/install:
```
gotify:
    image: gotify/server # gotify image
    ports:
      - 8080:80 # opens port 8080
    environment:
      GOTIFY_DEFAULTUSER_PASS: 'REDACTED'
    volumes:
      - './gotify_data:/app/data' # the data volume for the container
```

We're going to make that GOTIFY_DEFAULTUSER_PASS into an environment variable so it's not so publicly exposed.

Nvim into .env and write GOTIFY_PASS=whatever_password_you_like
```
nvim .env
```

Here's the final contents for the compose.yaml:
```
services:
  gotify:
    image: gotify/server
    restart: unless-stopped
    ports:
      - '8080:80'
    environment:
      GOTIFY_DEFAULTUSER_PASS: ${GOTIFY_PASS}
    volumes:
      - './gotify_data:/app/data'
  uptime-kuma:
    image: louislam/uptime-kuma:2
    restart: unless-stopped
    volumes:
      - './uptime-kuma_data:/app/data'
    ports:
      # <Host Port>:<Container Port>
      - '3001:3001'
```

**Securing the VM**

**SSH**
Run the following (with your desired email inserted) to generate an SSH key-value pair. This should be done on the device you are using to SSH into the VM, not on the VM itself.
```
ssh-keygen -t ed25519 -C "your_email@example.com"
```

You will then be prompted to select a location to save the key. I just press Enter to save to the default location.![](Attachments/Pasted%20image%2020261122161226.png)

You will then be prompted to select a passphrase, which acts as a second layer of security on top of your SSH key. I personally left it empty.
![](Attachments/Pasted%20image%2020261122161536.png)

Your identification and public key saved paths will be displayed in the terminal from this point. Additionally, the key's fingerprint and randomart will be displayed (you don't need to save these).

Now, we need to get your public key onto the VM's authorized keys. We can use the following command to do so, where user@hostname is the details of your VM (remote server) (such as sysadmin@10.10.10.10 for example):
```
ssh-copy-id user@hostname
```
The ssh-copy-id command will prompt you to enter the password for the VM. After its completion, if successful, when you try to SSH into your VM you won't need to enter a password anymore!

Next, we will turn off PasswordAuthentication. This makes it so that SSH logins must be done via SSH keys rather than by a typed password. Ensure that the keys from the previous step have been tested in a second terminal before continuing to the next step.

To set PasswordAuthentication to no:

First, navigate to the directory with sshd_config.
```
cd .. # Go to the previous directory until the root directory is reached
cd etc/ # cd into etc/
cd ssh/ # cd into ssh/
```

Then, view sshd_config in a text editor. I use nvim.
```
sudo nvim sshd_config # view sshd_config in a text editor
```

Once in the editor, the "/" character can be used to search for "PasswordAuthentication" in the file. The phrase will be highlighted.
```
/PasswordAuthentication
```

Visual:
![](Attachments/Pasted%20image%2020261122160253.png)

Once it is found, change the "yes" to a "no".
![](Attachments/Pasted%20image%2020261122160403.png)

Then, save and exit.
```
:wq
```

There's another PasswordAuthentication located in the sshd_config.d directory as well. First, we need to go there:
```
cd sshd_config.d
```

Then, we open 50-cloud-init.conf in our text editor and find the second PasswordAuthentication:
```
sudo nvim 50-cloud-init.conf
```
![](Attachments/Pasted%20image%2020261122164658.png)

We edit yes to no, save, and exit.

Now, when we try to ssh into our VM without a public key, we're denied as expected!
![](Attachments/Pasted%20image%2020261122165106.png)

Our regular ssh works perfectly.
![](Attachments/Pasted%20image%2020261122165214.png)


**FIREWALL**

First, we configure the firewall. We need to open ports 22 (SSH), 3001 (Uptime Kuma), and 8080 (Gotify). Aside from those ports, we default to deny incoming traffic.

```
sudo ufw allow 22/tcp
sudo ufw allow 3001/tcp
sudo ufw allow 8080/tcp
sudo ufw default deny incoming
```

Enable firewall with:
```
sudo ufw enable
```

See firewall status:
```
sudo ufw status
```
![](Attachments/Pasted%20image%2020261122203810.png)
**Unattended Upgrades**

Unattended-upgrades was automatically installed with the ubuntu live-server VM. We can check the 20auto-upgrades file at path /etc/apt/apt.conf.d/20auto-upgrades to see if the unattended upgrade is enabled with nvim or any text editor. I referenced https://ubuntu.com/server/docs/how-to/software/automatic-updates/.
![](Attachments/Pasted%20image%2020261122181810.png)
The '1''s indicate that it is enabled.

**Backups**

We want to automate our backups, so we'll do that with a script.

First, navigate to your desired directory (I chose to put the script in where compose.yaml is)
```
cd # return to home directory
cd Docker_Project/ # cd to the project directory w/ yaml file
```

Then, make a new bash script file and a directory to hold backups:
```
touch backup.sh # create new bash script file
mkdir backups # create directory for backups
```

This is the backup script I wrote:
```
#!/bin/bash 

docker compose down # Stop the app 

tar -cvf ./backups/$(date +%Y-%m-%d_%H%M%S)_backup.tar ./uptime-kuma_data ./gotify_data # Create the archive, list the associated files, and use the designated file as the archive. 
#Archives the data for both uptime kuma and gotify into a single archive, with the date as the name. 

docker compose up -d # Start the app 

backups_list=$(ls -t backups/*.tar | tail -n +8) # put the backups that aren't the 7 most recent into backups_list 

for backup in $backups_list; do # loop through the old backups and remove them one at a time 
	rm "$backup" 
	echo "$backup successfully removed" 
done
```

Before we can execute it, we need to grant execute permissions to the script:
```
chmod +x backup.sh
```

Run the backup script!:
```
sudo ./backup.sh
```

Successfully auto-removed the old backup!
![](Attachments/Pasted%20image%2020261122190356.png)

Here's what the backups directory looks like:
![](Attachments/Pasted%20image%2020261122204003.png)

**Proxmox Backup**
Navigate to Proxmox and go to the "Backup" tab for your VM.![](Attachments/Pasted%20image%2020261122190606.png)

Then, select "Backup now" and backup.![](Attachments/Pasted%20image%2020261122190644.png)

**Restore Drill**
First, we add something tangible to our apps (Uptime Kuma and Gotify). 

Navigate to Uptime Kuma at its address:
![](Attachments/Pasted%20image%2020261122192635.png)

On the left, select "Add New Monitor". Then, configure the URL (any URL works, I chose my URL for Gotify), and save at the bottom.![](Attachments/Pasted%20image%2020261122192752.png)

We now have an uptime monitor on our Gotify app.![](Attachments/Pasted%20image%2020261122193935.png)

Navigate to Gotify's address:
![](Attachments/Pasted%20image%2020261122192902.png)

Select the Apps tab, then select "CREATE APPLICATION". Give it any name, I gave it the name "restore test" since we're testing our restoration. Then, create!
![](Attachments/Pasted%20image%2020261122203237.png)

Now, to actually do the restore drill:

This is what our project directory looks like before:
![](Attachments/Pasted%20image%2020261122193330.png)

First, we run our script:
```
sudo ./backup.sh
```

Then, we stop the app:
```
docker compose down
```

We're then going to delete the data for our apps:
```
sudo rm -r gotify_data
sudo rm -r uptime-kuma_data
```

As we can see, we've deleted the data for Uptime Kuma and Gotify.
![](Attachments/Pasted%20image%2020261122193343.png)

Now, we restore our archive!
```
ls -t backups/ # list the backups in our backups directory (newest is first)
sudo tar -xvf ./backups/NameOfBackupHere
```
![](Attachments/Pasted%20image%2020261122193837.png)

Now, we'll start our app again to check if everything was restored properly!
```
docker compose up -d
```

Both of our applications have been properly restored. Thank you for reading!
![](Attachments/Pasted%20image%2020261122194614.png)
![](Attachments/Pasted%20image%2020261122194601.png)

**Reboot Test**

![](Attachments/Pasted%20image%2020261122203611.png)

We're going to reboot the system to see if the apps persist after a reboot.
```
sudo reboot
```

The apps are up and running.
![](Attachments/Pasted%20image%2020261122203714.png)

**Required Screenshots**

**docker compose ps**
![](Attachments/Pasted%20image%2020261122194747.png)

**App in a browser showing username:**
![](Attachments/Pasted%20image%2020261122194836.png)
![](Attachments/Pasted%20image%2020261122194902.png)

**Proxmox Snapshot:**
![](Attachments/Pasted%20image%2020261122194937.png)

**Promox Backup:**
![](Attachments/Pasted%20image%2020261122194951.png)