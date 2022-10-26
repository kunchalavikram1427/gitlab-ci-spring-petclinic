# End to End Gitlab Pipeline with Spring PetClinic Sample Application 

## Follow my GitLab Video Series
```
https://youtube.com/playlist?list=PL8klaCXyIuQ46vZOchy-fSregRq1YdPBF
```
## Youtube Video URL for this End to End Demo
```
https://youtu.be/6i1t7JcEOds
```

## Pre-Requisites
```
DigitalOcean Account: Get free 125 USD DigitalOcean cloud credits to try Kubernetes for 60 days. Sign up using this link: https://m.do.co/c/150b2fde3db0
Install Kubectl on Windows: https://youtu.be/G9MmLUsBd3g
Install Helm on Windows: https://youtu.be/ZKAlKoqlWac
Deploy Nexus on DigitalOcean: https://youtu.be/PzDnj_SGAQM
Deploy SonarQube on DigitalOcean: https://youtu.be/A0L1krD5l3c
Creating Kubernetes Cluster on DigitalOcean: https://youtu.be/_soteDvxaAk
GitLab Account: https://youtu.be/uqTqOwfosbw
```

## Understanding the Spring Petclinic application with a few diagrams
<a href="https://speakerdeck.com/michaelisvy/spring-petclinic-sample-application">See the presentation here</a>

## Running petclinic locally
```
git clone https://github.com/spring-projects/spring-petclinic.git
cd spring-petclinic
mvn package
java -jar target/*.jar
```
You can then access petclinic here: http://localhost:8080/

<img width="1042" alt="petclinic-screenshot" src="https://cloud.githubusercontent.com/assets/838318/19727082/2aee6d6c-9b8e-11e6-81fe-e889a5ddfded.png">

## POM file changes
Adding Nexus URL as a variable
```
<properties>
  <nexus.host.url>http://IP:PORT or DNS</nexus.host.url>
<properties>
```
Adding Distribution Management to distribute the artifacts
```
<repositories>
  . . .
</repositories>
  
<distributionManagement>
  <repository>
    <id>nexus</id>
    <name>Nexus Release Repository</name>
    <url>${nexus.host.url}/repository/maven-hosted-release/</url>
  </repository>
  <snapshotRepository>
    <id>nexus</id>
    <name>Nexus Snapshot Repository</name>
    <url>${nexus.host.url}/repository/maven-hosted-snapshot/</url>
  </snapshotRepository>
</distributionManagement>  
```

## Maven settings.xml file changes
To pass properties using Environment variables
```
<servers>
  <server>
    <id>nexus</id>
    <username>${env.NEXUS_USER}</username>
    <password>${env.NEXUS_PASSWORD}</password>
  </server>
</servers>
```
Pass variables to mvn
```
mvn deploy -DNEXUS_USER=yourusername -DNEXUS_PASSWORD=yourpassword
```

## sonar-project.properties file
```
sonar.projectKey=spring-petclinic
sonar.projectName=spring-petclinic
sonar.projectVersion=1.0
sonar.sources=src/main
sonar.tests=src/test
sonar.java.binaries=target/classes 
sonar.language=java
sonar.sourceEncoding=UTF-8
sonar.java.libraries=target/classes
```

## Building a Container
```
docker build -t kunchalavikram/spring-petclinic:latest .
```

## Running as a Container
```
docker run -d -p 80:8080 --name web kunchalavikram/spring-petclinic:latest
```

## Gitlab Runner Registration
Register runner and assign various jobs via tags
```
https://docs.gitlab.com/runner/install/kubernetes.html
https://docs.gitlab.com/runner/#tags
```

## Using Docker in Docker
```
https://docs.gitlab.com/ee/ci/docker/using_docker_build.html
```

## Using Kaniko to build container images
```
https://docs.gitlab.com/ee/ci/docker/using_kaniko.html
https://github.com/GoogleContainerTools/kaniko/blob/main/examples/pod.yaml
https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#log-in-to-docker
https://gitlab.com/guided-explorations/containers/kaniko-docker-build
```
Set below variables in project settings to push to dockerhub
```
CI_REGISTRY: https://index.docker.io/v1/
CI_REGISTRY_IMAGE: docker.io/youruser/yourrepo
CI_REGISTRY_PASSWORD: yourpassword
CI_REGISTRY_USER: Yourdockeruserid
```
```
dockerize:
  stage: dockerize
  image:
    name: gcr.io/kaniko-project/executor:v1.9.0-debug
    entrypoint: [""]
  script:
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"${CI_REGISTRY}\":{\"auth\":\"$(printf "%s:%s" "${CI_REGISTRY_USER}" "${CI_REGISTRY_PASSWORD}" | base64 | tr -d '\n')\"}}}" > /kaniko/.docker/config.json
    - /kaniko/executor --context "${CI_PROJECT_DIR}" --destination "$USERNAME/$IMAGE_NAME:$TAG"

```
## Install SonarQube
```
helm repo add sonarqube https://SonarSource.github.io/helm-chart-sonarqube
helm repo update
helm pull sonarqube/sonarqube --untar
```
Run
```
helm upgrade --install sonarqube sonarqube/ 
```
Password: admin

## Install Nexus
https://artifacthub.io/packages/helm/sonatype/nexus-repository-manager
```
helm repo add sonatype https://sonatype.github.io/helm3-charts/
helm repo update
helm pull sonatype/nexus-repository-manager --untar
```
Run
```
helm upgrade --install nexus nexus-repository-manager/
```
Get Password
```
kubectl exec -it nexus-nexus-repository-manager-5745766765-5pjcf -- bash
cat /nexus-data/admin.password
```

## GitLab Pipeline Steps Snippets

### Maven Nexus Deploy
```
mvn -s .m2/settings.xml -Dmaven.test.failure.ignore=true deploy
```
### Build Image
```
docker build -t $USERNAME/$IMAGE_NAME:$CI_COMMIT_SHA .
```
### Sonar Stage
https://docs.sonarqube.org/8.5/analysis/gitlab-cicd/
```
sonar-scanner -Dsonar.qualitygate.wait=true
```

## Dockerfile to build petclinic project docker image
```
FROM openjdk:8-jre-alpine
EXPOSE 8080
COPY target/*.war /usr/bin/spring-petclinic.war
ENTRYPOINT ["java","-jar","/usr/bin/spring-petclinic.war","--server.port=8080"]
```

## Dockerfile to build custom image with helm and kubectl cli tools
Image available in Dockerhub as: kunchalavikram/kubectl_helm_cli:latest
```
FROM alpine/helm
RUN curl -LO https://dl.k8s.io/release/v1.25.0/bin/linux/amd64/kubectl \
    && mv kubectl /bin/kubectl \
    && chmod a+x /bin/kubectl
 ```
## Kubernetes Deployment & Service files
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: petclinic
  labels:
    app: petclinic
spec:
  replicas: 1
  selector:
    matchLabels:
      app: petclinic
  template:
    metadata:
      labels:
        app: petclinic
    spec:
      containers:
      - name: petclinic
        image: kunchalavikram/spring-petclinic:latest
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: petclinic
spec:
  type: LoadBalancer
  selector:
    app: petclinic
  ports:
    - port: 80
      targetPort: 8080
```

## References
```
Maven CLI Options: https://maven.apache.org/ref/3.6.3/maven-embedder/cli.html
Maven settings xml: https://maven.apache.org/settings.html#servers
Sonarscan for Gitlab: https://docs.sonarqube.org/8.5/analysis/gitlab-cicd/
```

## Author
- Vikram K (www.youtube.com/c/devopsmadeeasy)
