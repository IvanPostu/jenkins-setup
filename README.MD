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
- <img src="./_forReadme/image copy 7.png" />
- Create a helloworld Freestyle project and make sure it was run on agent
- <img src="./_forReadme/image copy 8.png" />

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
1. [Jenkins War Packages (to download war archives)](https://get.jenkins.io/war-stable/)
1. [What Are The Steps To Install Jenkins On Linux Using A War File?](https://manage.accuwebhosting.com/knowledgebase/5378/What-Are-The-Steps-To-Install-Jenkins-On-Linux-Using-A-War-File.html)
1. <https://github.com/devopsjourney1/jenkins-101>
