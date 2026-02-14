# Jenkins Setup

## Docker master/slave

```sh
docker network create jenkins-network
docker network rm jenkins-network

docker pull jenkins/jenkins:2.550-jdk21

docker run -d --name jenkins-master \
    --restart=always \
    -p 8080:8080 -p 50000:50000 \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v $(which docker):/usr/bin/docker \
    -v jenkins_home:/var/jenkins_home \
    -u root \
    -e DOCKER_GID=$(getent group docker | cut -d: -f3) \
    --network jenkins-network \
    jenkins/jenkins:2.550-jdk21

docker logs jenkins-master

# Install plugins: Docker and Docker pipeline
# Create node

# setup slave container
docker pull jenkins/inbound-agent:alpine3.23-jdk21

docker run -d --name docker-agent1 \
    --network jenkins-network \
    --restart=always \
    -e JENKINS_AGENT_NAME="docker-agent1" \
    -e JENKINS_SECRET="a13f72722d8e2e4af59d3166bed290b60dabf9749ff3911a7f45ba1dbadc9501" \
    -e JENKINS_URL="http://jenkins-master:8080" \
    -u root \
    jenkins/inbound-agent:alpine3.23-jdk21

# connect to the agent
docker exec -it docker-agent1 bash
apk update

# option for regular image/container
curl -sO http://jenkins-master:8080/jnlpJars/agent.jar
java -jar agent.jar -url http://jenkins-master:8080/ -secret "$JENKINS_SECRET" -name "docker-agent1" -webSocket -workDir "/home/jenkins/agent"
```

- <img src="./_forReadme/image copy 3.png" />
- Stop using master for builds
- <img src="./_forReadme/image copy 4.png" />
- Add new agent
- <img src="./_forReadme/image copy 5.png" />
- <img src="./_forReadme/image copy 6.png" />
- <img src="./_forReadme/image.png" />
- Create a helloworld Freestyle project and make sure it was run on agent
- <img src="./_forReadme/image copy 8.png" />

## Pipeline example

- Create a new pipeline
- Copy/paste `Jenkinsfile.template`
- Run and open `Pipeline overview`
- <img src="./_forReadme/image copy.png" />

## Pipeline from github example

- create new pipeline
- <img src="./_forReadme/image copy 2.png" />
- <img src="./_forReadme/image copy 7.png" />
- to avoid the error below:
- <img src="./_forReadme/image copy 9.png" />
- fix (on master and slaves): `ssh-keyscan -t ed25519 github.com >> ~/.ssh/known_hosts`
- <img src="./_forReadme/image copy 10.png" />

## Generate ssh key for github

- <https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent>

## Docker Containers as Jenkins Build Agents

Expose docker to the same network as jenkins, change docker service

```sh
"unix:///var/run/docker.sock"
"tcp://127.0.0.1:4243" # or, will need additional setup for container to see it on host's host's
# "tcp://0.0.0.0:4243" # don't forget to adjust firewall settings
```

- Test: `curl http://127.0.0.1:4243/version`
- Test: `wget -qO- http://127.0.0.1:4243/version`
- Test: `curl --unix-socket /var/run/docker.sock http://localhost/version`
- In jenkins install `Docker` plugin
- <img src="./_forReadme/image copy 11.png" />

- Access docker api from container:

```sh
# doesn't work on rootless docker!!!
# docker run --rm -it --name my-alpine \
#     --privileged \
#     --user root \
#     --network jenkins-network \
#     -v /var/run/docker.sock:/var/run/docker.sock \
#     -e DOCKER_HOST=unix:///var/run/docker.sock \
#     alpine:3.23 sh

# ls -n /var/run/docker.sock
# apk update
# apk add curl
# curl --unix-socket /var/run/docker.sock http://localhost/version

docker pull docker:29.2.1-dind
docker run --privileged --name my-dind-container -d docker:29.2.1-dind

# use ip got via: ip addr
docker exec -it \
  -e DOCKER_HOST="tcp://192.168.0.3:4243" \
  my-dind-container sh

# if you run inside the container : docker ps, it works
```

- setup ssh <img src="./_forReadme/image copy 12.png" />
- setup JNLP
  - image: `jenkinsci/jnlp-slave:4.3-1` (runs java 8, got error due to too old java version)
  - image: `jenkins/inbound-agent:alpine3.23-jdk21` (WORKED!!!)
  - label: `docker-jnlp-slave`
  - <img src="./_forReadme/image copy 13.png" />
  - <img src="./_forReadme/image copy 14.png" />
  - <img src="./_forReadme/image copy 15.png" />
  - <img src="./_forReadme/image copy 16.png" />

```sh
docker pull jenkinsci/jnlp-slave:4.3-1 # java 8 incompatible with jenkins's newer version of java, use jenkins/inbound-agent:alpine3.23-jdk21 instead
```

## Manual setup

```sh
docker pull alpine:3.23
docker network create jenkins
docker run --rm -it --name my-alpine \
    --network jenkins \
    -p 9090:9090 alpine:3.23 sh

# docker exec -it --user root db9a4c4d530a sh

apk update

apk search -v openjdk

# coreutils to run sha256sum
# fontconfig ttf-dejavu fonts needed for jenkins

apk add --no-cache coreutils curl fontconfig ttf-dejavu git openjdk21-jre

# apk del openjdk21-jre-headless

adduser -D myuser
su myuser
cd "/home/myuser"

mkdir -p "/home/myuser/jenkins_home"
export JENKINS_HOME="/home/myuser/jenkins_home"

# -f - fail on request error
# -L follow redirect
# -O save as file
curl -fLO https://get.jenkins.io/war-stable/2.541.1/jenkins.war

# hash check
# see https://get.jenkins.io/war-stable/2.541.1/jenkins.war.sha256
EXPECTED="8b12678aa6f9550b06825b006f9096186b17a7b857e9b68ad3f980d9f430fc94"
ACTUAL=$(sha256sum jenkins.war | awk '{print $1}')

if [ "$ACTUAL" = "$EXPECTED" ]; then
  echo "Checksum OK"
else
  echo "Checksum FAILED"
  exit 1
fi
# hash check

java -jar jenkins.war --httpPort=9090
# initial password is located here:
# /home/myuser/.jenkins/secrets/initialAdminPassword
```

## References

1. <https://www.youtube.com/watch?v=YGAvqJFr_yM>
1. <https://www.youtube.com/watch?v=6YZvp2GwT0A>
1. [Jenkins War Packages (to download war archives)](https://get.jenkins.io/war-stable/)
1. [What Are The Steps To Install Jenkins On Linux Using A War File?](https://manage.accuwebhosting.com/knowledgebase/5378/What-Are-The-Steps-To-Install-Jenkins-On-Linux-Using-A-War-File.html)
1. <https://github.com/devopsjourney1/jenkins-101>
1. <https://dev.to/behainguyen/cicd-06-jenkins-accessing-private-github-repos-using-ssh-keys-313b>
1. <https://www.youtube.com/watch?v=yb6DodK6mbg>
1. <https://medium.com/@moshedana058/understanding-docker-in-docker-dind-method-a-comprehensive-guide-45ab416c1a53>
