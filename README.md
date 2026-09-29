# M6-NEXUS-REPOSITORY

## Setting up cloud hosted repository, building and pushing artifacts to it

### Nexus Repository

### EXERCISE 1: Install Nexus on a server

Created a new droplet with 4vCPU - 8 GB of RAM - 160 GB SSD. Used pre-existing firewall and ssh key for efficiency.

Commands used:

```bash
ssh roo@<DROPLET_IP>
apt install openjdk-25-jre-headless -y
cd /opt
wget https://download.sonatype.com/nexus/3/nexus-3.96.3-01-linux-x86_64.tar.gz
tar -xf nexus-3.96.3-01-linux-x86_64.tar.gz
adduser nexus 
New password: ********************
Retype new password: *****************
chown -R nexus:nexus nexus-3.96.3-01/
chown -R nexus:nexus sonatype-work/
echo "run_as_user='nexus'" >> nexus-3.96.3-01/bin/nexus.rc
su - nexus
/opt/nexus-3.96.3-01/bin/nexus start
ss -tulpn # shows listening on port 8081
cat /opt/sonatype-work/nexus3/admin.password # saved for later
```
Allowed port 8081 inbound from my home IP. Tested successfully

### EXERCISE 2: Create npm hosted repository

Heavy UI work. Blob Store > "Create Blob Store" > Type = "File" > Name npm-hosted > save

Settings > Repositories > "Create repository" > npm (hosted) > select newly created blob store

###  EXERCISE 3: Create user for team 1

More UI work.

1. **CREATE ROLE** Settings > Security > Roles > Create Role > Type = Nexus role > Role ID = nx-npm > applied privileges = nx-repository-view-npm-*-* 

> [!IMPORTANT]
> I realize above privileges would give full access to all npm repos. In production I would lock it down to only the repo the user needs access to.

2. **CREATE USER** Settings > Security > Users > "Create local user" > ID = npm-user > Roles = nx-npm

This satisfies the reqs

### EXERCISE 4. Build and publish npm tar

Recreated most of this locally to codify my steps

Commands used: 

```bash
cd node-app/

npm login --http://localhost:8081/repository/npm-hosted/ # entered user credentials from last step

npm publish --http://localhost:8081/repository/npm-hosted/ # packages and pushes tar to repo in one step
```

### EXERCISE 5. Create maven hosted repository

> [!NOTE]
> This step is almost identical to the previous exercise so I will not retype it. I acknowledge the only difference is selecting maven2 as the repository format

### EXERCISE 6. Create user for team 2

> [!NOTE]
> This step is almost identical to the previous exercise so I will not retype it. I acknowledge the only difference is creating a role with privileges to the newly create maven repository and assigning them to the new user

### EXERCISE 7. Build and publish jar file

Edited build.gradle file with 

```md
repositories {
        maven {
            name = "nexus"
            url = "http://{nexus-ip}:8081/repository/{repo-name}/"
            allowInsecureProtocol = true
            credentials {
                username project.repoUser
                password project.repoPassword
            }
```
Added aforementioned variable definitions to gradle.properties file

```bash
gradle build
gradle publish
```

### EXERCISE 8: Download from Nexus and start application

- Created a new user with privileges to BOTH of the created repositories
- Fetched artifact with `curl -u {user}:{password} -X GET 'http://{nexus-ip}:8081/service/rest/v1/components?repository={repo-name}&sort=version'
- copied download URL and ran wget command to download file
- java -jar java-app-1.0.jar

### EXERCISE 9: Automate

I have pasted my automation script to this below:

```bash
#!/bin/bash
# using curl to query api server. grepping for http line and extracting url with awk engineering skills. assigning url to var
#art_url=$(curl -s -u app-tester:apptester123 -X GET "http://67.205.156.216:8081/service/rest/v1/components?repository=npm-snapshots" | grep http | awk -F '"' '{ print $4 }')

# new version more clean...using jq to output everything in the query to the artifact.json file.
curl -s -u  app-tester:apptester123 -X GET 'http://67.205.156.216:8081/service/rest/v1/components?repository=npm-snapshots&sort=version' | jq "." > artifact.json

# using jq -r to output raw which removes the quotes for this query so I get a clean url. select endswith calls out to stop output when it finds a .tgz file.
download_url=$(jq -r '.items[0].assets[0].downloadUrl | select(endswith(".tgz"))' artifact.json)

# assigned file name to variable using basename to strip all but file name
snapshot=$(basename "$download_url")

echo $snapshot

sleep 1

# wget command using credentials to access rest api in order to download the file used snapshot var to save it as a file that matches the artifact name
wget --http-user=app-tester --http-password=apptester123 "$download_url" -O "$snapshot"

# simple tar extraction using snapshot var
tar -xvf "$snapshot"

# switch to package dir
cd package/

# build
npm install
# run
node app/server.js &

```

