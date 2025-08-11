# Jenkins CI/CD

## 1. Build Automation Concepts

**Goal of CI/CD automation after pushing to a remote Git repository:**
1. Checkout source code.
2. Build the application.
3. Run automated tests.
4. Build artifacts (e.g., JAR, Docker image).
5. **Publish artifacts** to registry/repository (**CI**).
6. **Deploy artifacts** to target (**CD**).
7. Optional: send notifications, trigger downstream jobs, etc.

**Popular build servers/tools:**
- [Jenkins](https://www.jenkins.io/) (self‑hosted, extensible via plugins).
- [Travis CI](https://www.travis-ci.com/), [GitLab CI/CD](https://about.gitlab.com/), [Bamboo](https://www.atlassian.com/software/bamboo), [TeamCity](https://www.jetbrains.com/teamcity), etc.

***

## 2. Installing Jenkins in Docker

**DigitalOcean Setup**
- Create **Droplet** (min. 1 GB RAM, recommended 4 GB for stability).
- Add firewall rules:
  - Port **22** (SSH) from your IP.
  - Port **8080** (Custom TCP) from all IPs.

**Install Docker and run Jenkins container**:
```bash
apt update
apt install docker.io

docker run -p 8080:8080 -p 50000:50000 \
  -d -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
```
- Port **50000**: master–agent communication.

**Retrieve initial admin password**:
```bash
# Inside container:
docker exec -it  bash
cat /var/jenkins_home/secrets/initialAdminPassword

# Or on host:
docker volume inspect jenkins_home  # get Mountpoint
cat /var/lib/docker/volumes/jenkins_home/_data/secrets/initialAdminPassword
```
- Access Jenkins UI at `http://:8080`, install **Suggested Plugins**, create admin user.

***

## 3. Jenkins UI Basics

- **Administrators** → "Manage Jenkins": plugins, credentials, cluster setup, backups.
- **Users** → "New Item": create jobs (Freestyle, Pipeline, Multibranch, etc.).

***

## 4. Configuring Build Tools

**Options to make tools available:**
1. Install via Jenkins plugin (UI).
2. Install manually inside server/container.

**Example: Maven Plugin**
- Already preinstalled in LTS Jenkins.
- Configure in **Manage Jenkins > Global Tool Configuration**:
  - Add Maven installation (e.g., name: `maven-3.6`).

**Example: Install Node.js and npm in Jenkins container**
```bash
docker exec -u 0 -it  bash
apt update && apt install curl
cat /etc/issue # identify distro
curl -sL https://deb.nodesource.com/setup_10.x -o nodesource_setup.sh
bash nodesource_setup.sh
apt install nodejs
```

***

## 5. Freestyle Job Basics

- Create via **New Item > Freestyle Project**.
- Configure **Build Steps**:  
  - Shell Command: `npm --version`
  - Maven Goal: `--version` (select configured Maven).

- **Plugin Configuration Steps**:
  - Install `NodeJS` plugin.
  - Configure in **Global Tool Configuration**.

- **Git Repository Integration**:
  - **Source Code Management > Git** → set repo URL, credentials.
  - Add credentials in Jenkins via "Username with password" form.

- **Jenkins file structure**:
  - Logs: `/var/jenkins_home/jobs//`.
  - Checked-out code: `/var/jenkins_home/workspace//`.

- **Executing Scripts from Repo**:
  - Ensure execute permission: `chmod +x script.sh`.

- **Java Build Example**:
  - Maven goals: `test` → `package`
  - Artifact found under: `workspace//target/`.

***

## 6. Using Docker from Jenkins

**Grant Jenkins access to Docker CLI (Docker-outside-of-Docker)**:
```bash
docker run -p 8080:8080 -p 50000:50000 -d \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(which docker):/usr/bin/docker \
  jenkins/jenkins:lts
```
- Then, inside container as root:
```bash
docker exec -u 0 -it  bash
chmod 666 /var/run/docker.sock
```

**Build & Push Docker Image in Jenkins Job**:
```bash
docker build -t my-app:1.0 .
```
**Push to Docker Hub**:
- Add Docker Hub creds (`Global Credentials`).
- Bind credentials to env vars in Build Environment.
- Push:
```bash
echo $DOCKER_HUB_PASSWORD | docker login -u $DOCKER_HUB_USERNAME --password-stdin
docker push :
```

**Push to Nexus Docker Repo** (HTTP):
- Add insecure registry to `/etc/docker/daemon.json`:
```json
{
  "insecure-registries": [":8083"]
}
```
- Restart docker, restart Jenkins container, reapply socket permissions.

***

## 7. From Freestyle to Pipelines

- Freestyle chaining possible via *Post-build actions* but pipelines are cleaner.
- Pipelines as code (Groovy) → `Jenkinsfile` in repo.

***

## 8. Jenkinsfile Basics

Two styles:
- **Scripted**:
```groovy
node { ... }
```
- **Declarative**:
```groovy
pipeline {
  agent any
  stages {
    stage("build") { steps { ... } }
    stage("test") { steps { ... } }
  }
}
```

**Key features**:
- `post { always / success / failure }`
- `when { expression { ... } }`
- Environment vars (`env.VAR`), credentials binding, `tools {}`, parameters, external script loading.
- `input {}` for manual approvals.

***

## 9. Example Full Pipeline

```groovy
pipeline {
  agent any
  tools { maven 'maven-3.9' }
  stages {
    stage("build jar") { steps { sh 'mvn package' } }
    stage("build image") {
      steps {
        withCredentials([usernamePassword(credentialsId: 'docker-creds', usernameVariable: 'U', passwordVariable: 'P')]) {
          sh 'docker build -t repo/app:${BUILD_NUMBER} .'
          sh "echo $P | docker login -u $U --password-stdin"
          sh 'docker push repo/app:${BUILD_NUMBER}'
        }
      }
    }
    stage("deploy") { steps { echo 'deploying...' } }
  }
}
```

***

## 10. Multibranch Pipelines

- Auto-detect branches with `Jenkinsfile`.
- Branch-specific logic using `env.BRANCH_NAME`.

***

## 11. Credentials Management

**Scopes**:
- **System**: server-only.
- **Global**: visible to all jobs.
- **Multibranch Pipeline**: visible only within that folder/project.

***

## 12. Shared Libraries

**Structure**:
```
vars/   -> Groovy functions, 1 per file
src/    -> Classes (Serializable), helper code
resources/
```
- Make available in **Global Pipeline Libraries** or per-job via `@Library`.
- Can have parameters, reuse logic via classes.

***

## 13. Webhooks & Auto-Triggers

- Install **GitLab Build Trigger** plugin.
- Configure GitLab -> Jenkins integration (under Integrations/Webhooks).
- For multibranch: **Multibranch Scan Webhook Trigger** plugin; configure webhook URL with token.

***

## 14. Auto Versioning in Pipelines

**Increment Maven Version**:
```bash
mvn build-helper:parse-version versions:set \
  -DnewVersion=${parsedVersion.majorVersion}.${parsedVersion.minorVersion}.${parsedVersion.nextIncrementalVersion} \
  versions:commit
```
**Use in Jenkinsfile**:
- Read new version, save to `env.IMAGE_VERSION`.
- Replace hardcoded JAR names in Dockerfile with wildcard (`*.jar`).
- Use `mvn clean package` to ensure single artifact.

***

## 15. Commit Version Back to Git

```groovy
stage('Commit Version Update') {
  steps {
    withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'U', passwordVariable: 'P')]) {
      sh "git remote set-url origin https://${U}:${P}@github.com/user/repo.git"
      sh 'git add .'
      sh 'git commit -m "ci: version bump"'
      sh 'git push origin HEAD:main'
    }
  }
}
```
- Configure Git user inside Jenkins container:
```bash
git config --global user.email "jenkins@example.com"
git config --global user.name "jenkins"
```
- Avoid build loops by ignoring commits from Jenkins user (via *Ignore Committer Strategy* plugin or Git settings).

***

## Best Practices
- Keep Jenkinsfiles in repo under version control.
- Use credentials bindings instead of hardcoding secrets.
- Run Jenkins in Docker for easy updates/portability.
- Use pipelines for all but simplest jobs.
- Clean workspaces regularly.
- Manage build artifacts in registries like Nexus, Artifactory, or Docker Hub.
- Avoid committing from CI unless necessary.

***
