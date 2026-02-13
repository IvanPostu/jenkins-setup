# Jenkins Setup

## Manual setup

```sh
docker pull alpine:3.23
docker run --rm -it --name my-alpine -p 9090:9090 alpine:3.23 sh

# docker exec -it --user root db9a4c4d530a sh

apk update

apk search -v openjdk

apk add --no-cache coreutils \ # to run sha256sum
    curl \
    fontconfig ttf-dejavu \ # fonts needed for jenkins
    git \
    openjdk21-jre

# apk del openjdk21-jre-headless

adduser -D myuser
su myuser

mkdir -p "/home/myuser/jenkins_home"
export JENKINS_HOME="/home/myuser/jenkins_home"

# -f - fail on request error
# -L follow redirect
# -O save as file
curl -fLO https://get.jenkins.io/war-stable/2.528.1/jenkins.war

# hash check
# see https://get.jenkins.io/war-stable/2.528.1/jenkins.war.sha256
EXPECTED="d630dca265f75a8d581f127a9234f1679d4b0800a8f370d03ad4a154ceb7295b"
ACTUAL=$(sha256sum jenkins.war | awk '{print $1}')

if [ "$ACTUAL" = "$EXPECTED" ]; then
  echo "Checksum OK"
else
  echo "Checksum FAILED"
  exit 1
fi
# hash check

apk add --no-cache fontconfig ttf-dejavu
java -jar jenkins.war --httpPort=9090
```

## Progress

- With recommended plugins
- <img src="./_forReadme/image.png" />

### Echo example

- <img src="./_forReadme/image copy.png" />
- Enable: Delete workspace before build starts
- Build: execute shell `echo "Build ID: ${BUILD_ID}"`
- Save
- Build now
- Open successfully build in history
- Open console output

## References

1. [Jenkins War Packages (to download war archives)](https://get.jenkins.io/war-stable/)
1. [What Are The Steps To Install Jenkins On Linux Using A War File?](https://manage.accuwebhosting.com/knowledgebase/5378/What-Are-The-Steps-To-Install-Jenkins-On-Linux-Using-A-War-File.html)
