## Jenkins

### Integration with OpenShift

These commands would need to be executed against each project Jenkins needed to access.

```
oc create serviceaccount jenkins -n kavishm9-dev
oc policy add-role-to-user edit system:serviceaccount:kavishm9-dev:jenkins -n kavishm9-dev

# the commands below adds a cluster role which gives edit rights to access to all projects
oc adm policy add-cluster-role-to-user edit system:serviceaccount:kavishm9-dev:jenkins

oc serviceaccounts get-token jenkins -n kavishm9-dev
```

The token is then used under Configure Clouds section of Jenkins

### Installation in OpenShift

Jenkins standard authentication

```
oc new-app -e JENKINS_PASSWORD=<password> quay.io/openshift/origin-jenkins:4.10.0
```



### Fix CSS Loading Issue with HTML Publisher

```
System.setProperty("hudson.model.DirectoryBrowserSupport.CSP", "")
```





## OpenShift

### Login

```
Internal DNS
DNS VIP IP: 10.168.0.250
DNS IP : 10.168.0.251

https://jenkins-okd-infra-jenkins.apps.okd.lab.mez9.local/
https://console-openshift-console.apps.okd.lab.mez9.local/
http://ansible-openshift-infra.apps.okd.lab.mez9.local/
https://mez9-local-gitea-okd-infra-gitea.apps.okd.lab.mez9.local/

oc login --token=sha256~jnmI_BuiZcsi-YwQmcnJMnx8x2CVZIEamiFqx7cFn0M --server=https://api.okd.lab.mez9.local:6443

oc login -u kavish -p P@ssw0rd https://api.okd.lab.mez9.local:6443
```

### Import Image

```
oc import-image 10.168.0.62/docker-kavish/db:1.0.3-ci.9
```

### Import Image to new ImageStream

Create push secret beforehand

```
```



```
oc import-image ose-jenkins-agent-maven:v4.9.0 --from=10.168.0.76:9002/dependency-registry/ose-jenkins-agent-maven:v4.9.0 --confirm
```



### Push Image to Internal Registry

ImageStreams need to be created beforehand, or --create --confirm flags can be used

```
oc whoami
oc whoami --show-token

docker login -u kavishm9 -p $(oc whoami --show-token) default-route-openshift-image-registry.apps.sandbox.x8i5.p1.openshiftapps.com/kavishm9-dev
docker push docker push default-route-openshift-image-registry.apps.sandbox.x8i5.p1.openshiftapps.com/kavishm9-dev/ose-jenkins-agent-base:v4.8.0
```



### Add insecure registries in OpenShift

```
oc edit image.config.openshift.io/cluster
```

Sample configuration

```yaml
apiVersion: config.openshift.io/v1
kind: Image
metadata:
  annotations:
    include.release.openshift.io/self-managed-high-availability: "true"
    include.release.openshift.io/single-node-developer: "true"
    release.openshift.io/create-only: "true"
  creationTimestamp: "2021-09-17T12:41:03Z"
  generation: 4
  name: cluster
  resourceVersion: "10080941"
  uid: 84471703-f4fb-4cea-bbe9-7017fa82d49e
spec:
  additionalTrustedCA:
    name: registry-config
  registrySources:
    allowedRegistries:
    - quay.io
    - docker.io
    - 10.168.0.62:8082
    - 10.168.0.62
    insecureRegistries:
    - 10.168.0.62:8082
    - 10.168.0.62
```

Verify using command below in one of the worker nodes

```
cat /etc/containers/registries.conf
```

### Simple DeploymentConfig with Private Registry

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
  labels:
    app: words-db
spec:
  selector:
    matchLabels:
      app: words-db
  template:
    metadata:
      labels:
        app: words-db
    spec:
      containers:
      - name: db
        image: 10.168.0.62/docker-kavish/test-image:1.0
        ports:
        - containerPort: 5432
          name: db
      imagePullSecrets:
      - name: artifactory-secret
```

### DeploymentConfig for postgresql

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: db-kavish
    name: db-kavish
  namespace: postgres
spec:
  selector:
    matchLabels:
      app: db-kavish
  template:
    metadata:
      labels:
        app: db-kavish
    spec:
      containers:
        - env:
          - name: POSTGRES_DB
            value: postgres
          - name: POSTGRES_USER
            value: postgres
          - name: POSTGRES_PASSWORD
            value: postgres
          - name: PGDATA
            value: /temp/data
      image: 10.168.0.62/docker-kavish/test-image:1.3
      name: postgres
      imagePullSecrets:
      - name: artifactory-secret
    ports:
      - containerPort: 5432
        protocol: TCP
---

apiVersion: v1
kind: Service
metadata:
  labels:
    app: db-kavish
  name: db-kavish
  namespace: test-project
spec:
  ports:
    - name: http
      port: 5432
      protocol: TCP
  selector:
    app: db-kavish
```

### Add External Registry Secret

```
export DOCKER_REGISTRY_SERVER=http://10.168.0.62:8082
export DOCKER_USER=admin
export DOCKER_PASSWORD=P@ssw0rd
export DOCKER_EMAIL=kavish.punchoo@gmail.com

oc create secret docker-registry artifactory-secret --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL

oc create secret docker-registry artifactory-secret --docker-server=http://10.168.0.62:8082 --docker-username=admin --docker-password=P@ssw0rd

oc create secret docker-registry nexus-secret --docker-server=http://10.168.0.76:9000 --docker-username=admin --docker-password=P@ssw0rd --docker-email=kavish.punchoo@gmail.com
```

### Add External Registry Secret to Builder Service Account

Only required if **oc new-build** command is used.

```
oc edit sa builder
```

```
apiVersion: v1
imagePullSecrets:
- name: builder-dockercfg-cc2t7
kind: ServiceAccount
metadata:
  creationTimestamp: "2021-11-01T07:55:51Z"
  name: builder
  namespace: test-project
  resourceVersion: "11236043"
  uid: 467b5dc9-7c50-45e6-8d7e-c7a3be382347
secrets:
- name: builder-token-dv8bq
- name: builder-dockercfg-cc2t7
- name: artifactory-secret
```

### Start Build from BuildConfig and wait for error code

```
oc start-build kavish-test -w
```

### Service Account Creation for Jenkins

Refer to Jenkins Section



### Copy File into Running Pod

```
oc rsync ./local/dir <pod-name>:/remote/dir

oc rsync C:\Users\Kavish\Documents\Temp newman-pod-test-7-b3rjw-r83lt-l6jkp:/opt/tests --strategy='tar' --container='sel'
```





## Docker

###  Build from Dockerfile

```
docker build -t test-image .
```

### Tag and Push

```
docker tag test-image 10.168.0.62/docker-kavish/test-image:1.0
docker push 10.168.0.62/docker-kavish/test-image:1.0
```



## SonarQube

### Maven Example

```
mvn clean package
sonar-scanner -Dsonar.projectKey=kwsp-webapp -Dsonar.sources=. -Dsonar.host.url=http://10.168.0.66:9000 -Dsonar.login=b8670e9d500e2c403edc9f550233dd6a3a3e0327 -Dsonar.java.binaries=target/classes -Dsonar.qualitygate.wait=true
```

### Go Example

Start sonar-scanner and push results to server

```
sonar-scanner -Dsonar.projectKey=app-go-web -Dsonar.sources=. -Dsonar.host.url=http://10.168.0.66:9000 -Dsonar.login=33e0b35978bfcc4685ce3d0a87dc6bf04efa9aca
```


## Nexus 

Upload raw file using curl

```
curl -v -u admin:P@ssw0rd --upload-file dispatcher.go http://10.168.0.76:8081/repository/dev-raw/
```

Push Docker image to Nexus

```
docker push 10.168.0.76:9000/dev-registry/web:0.1.0-alpha.12
```



## Skopeo

Copy image from one registry to another

```
skopeo copy docker://10.168.0.64:8082/dev-registry/kavish-test:latest  docker://10.168.0.64:8083/prod-registry/kavish-test:latest --src-tls-verify=false --dest-tls-verify=false

skopeo copy docker://10.168.0.76:9000/dev-registry/iris-app:latest  docker://10.168.0.76:9001/test-registry/iris-app:latest --src-tls-verify=false --dest-tls-verify=false
```



## CRC OpenShift

### Access Internal Registry

1. Add default-route-openshift-image-registry.apps-crc.testing in the ìnsecure-registries` list in Docker Desktop configuration
2. Run `docker run -it --rm --privileged --pid=host dockerpinata/nsenter-dockerd`
3. Inside this shell, run `echo "192.168.1.235 default-route-openshift-image-registry.apps-crc.testing" >> /etc/hosts`. 192.168.1.235 is the result of `nslookup host.docker.internal`.

You will be able to login with `docker login -u kubeadmin -p $(oc whoami -t) default-route-openshift-image-registry.apps-crc.testing`

Each time you restart Docker you will have to run step 2 and 3 again.



## IBM EWM/RTC

### scm command line

Login to CCM

```
lscm login -r https://10.168.0.74/ccm -u Administrator -P P@ssw0rd -n ccm
```

List Streams

```
lscm list streams -r ccm
```

Create Workspace from Stream

```
lscm create workspace -r ccm -s "OpenShiftDemo (Change Management) Stream" CICDWorkspace
```

Load Workspace

```
lscm load -r ccm --all --include-root CICDWorkspace
```

Unload Workspace

```
lscm unload --ignore-uncommitted --delete --workspace CICDWorkspace
```

Delete Workspace

```
lscm delete workspace CICDWorkspace -r ccm
```

To Make and Deliver Changes

https://jazz.net/library/article/620



## IBM UCD

### Required UCD Plugins

```
Docker Source Plugin
Docker Automation Plugin
Kubernetes Automation Plugin
```

### Required UCD Templates

```
https://github.com/IBM-UrbanCode/Templates-UCD/blob/master/Docker/componenttemplates/Docker%2BTemplate.json
https://github.com/IBM-UrbanCode/Templates-UCD/blob/master/Kubernetes/Tutorial/KubernetesComponentTemplate.json
https://github.com/IBM-UrbanCode/Templates-UCD/blob/master/Kubernetes/Tutorial/KubernetesApplicationTemplate.json
```



### Get Resources

```
udclient -weburl https://10.168.0.65:8443 -username admin -password admin getResources
```



### Request Application Process Execution

This process is asynchronous.

For processes in progress, result will be NONE

For processes that failed result will be FAULTED

For process that passed, result will be SUCCEEDED

```
udclient -weburl https://10.168.0.65:8443 -username admin -password admin requestApplicationProcess runProcess.json
```

runProcess.json

```
{
  "application": "JPetStore",
  "description": "Requesting deployment",
  "applicationProcess": "Deploy JPetStore",
  "environment": "Tutorial environment 1",
  "onlyChanged": "false",
  "properties": {
    "Prop1": "value1"
  },
  "versions": [
    {
      "version": "1.0",
      "component": "JPetStore-APP"
    },
    {
      "version": "1.0",
      "component": "JPetStore-WEB"
    },
    {
      "version": "1.0",
      "component": "JPetStore-DB"
    }
  ]
}
```



### Get Application Process Request

```
udclient -weburl https://10.168.0.65:8443 -username admin -password admin getApplicationProcessRequest -requestID <id>
```





## Maven

### Run Maven JUnit Test with Report

```
mvn test surefire-report:report
```



## Gradle

```
gradle build
```





## Minikube

### Expose Minikube API and dashboard

http://10.168.0.64:8001/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy

```
su - minikube
kubectl proxy --address='0.0.0.0' --disable-filter=true
```



## JFrog CLI

List Config

```
jfrog config show
```

Add Config

```
jfrog config add --insecure-tls=true
```

Add Config in CI

```
jfrog config add --insecure-tls=true --artifactory-url="http://10.168.0.62:8082/artifactory" --xray-url="http://10.168.0.62:8082/xray" --user=admin --password="P@ssw0rd" --interactive=false jfrog-platform
```

### Maven Build

Configure JFrog before Maven build

```
jfrog rt mvn-config --server-id-deploy=jfrog-platform --repo-deploy-releases=maven-kavish --repo-deploy-snapshots=maven-kavish

# new syntax
jfrog mvnc --server-id-deploy=jfrog-platform --repo-deploy-releases=maven-kavish --repo-deploy-snapshots=maven-kavish
```

Build Maven and Collect Build Info

```
jfrog rt mvn clean test package --build-name=kwsp-webapp-java --build-number=39
```

Send Build Info

```
jfrog rt bp kwsp-webapp-java ${env.BUILD_NUMBER} --build-url=http://10.168.0.60:8080/job/iAkaun/46/ --server-id=jfrog-platform
```



### Docker Build and Publish

```
jfrog rt docker-push 10.168.0.62:8082/docker-kavish/kwsp-webapp:1.0 docker-kavish --build-name=kwsp-webapp --build-number=39 --server-id=jfrog-platform
jfrog rt bp kwsp-webapp 39 --server-id=jfrog-platform
```



### Xray Scan of Build

```
jfrog bs <build name> <build number>
jfrog bs devops-test-build 1
```





## Postman / Newman

### Setup

```
npm install -g newman
npm install -g newman-reporter-html
npm install -g newman-reporter-htmlextra
```

### Run Collection with HTML Report

```
newman run collection.json --reporters cli,html --reporter-html-export newman_report.html
```

### Run Collection with HTMLExtra

```
newman run collection.json -r cli,htmlextra --reporter-htmlextra-export newman_report.html --reporter-htmlextra-title "API Test Report" --reporter-htmlextra-browserTitle "API Test Report"
```



## OWASP ZAP

### Start ZAP container

```
docker pull owasp/zap2docker-stable
docker run --rm owasp/zap2docker-stable zap.sh -daemon -host 0.0.0.0 -port 8080 -config api.addrs.addr.name=.* -config api.addrs.addr.regex=true -config api.disablekey=true -dir /zap/scan
```

### Use zap-cli

```
zap-cli start -o "-config api.key=12345"

zap-cli --api-key 12345 active-scan http://110.30.182.32:3000

zap-cli --api-key 12345 shutdown
```



### Scan and Generate Report

```
zap-cli quick-scan https://reqres.in/
zap-cli -v report -o report.html -f html
```



## Xcode

Requires Team for Signing and Capabilities of Target

### List Projects, Workspace Targets, Build Configuration and Schemes

cd into Xcode project folder

```
xcodebuild -list
```

### Build Project

```
xcodebuild -target BullsEye
```

### List Available Destinations

```
xcrun xctrace list devices
```

### Run Tests

```
xcodebuild test -scheme BullsEye -destination 'platform=iOS Simulator,OS=14.4,name=iPod touch (7th generation)'
```

### Run Tests and export JUnit format output

```
xcodebuild -derivedDataPath DerivedData -scheme BullsEye -destination 'platform=iOS Simulator,OS=14.4,name=iPod touch (7th generation)' test | xcpretty --report junit
```

### Validate and Upload ipa file

```
xcrun altool --validate-app --file "$IPA_PATH" --username "$APP_STORE_USERNAME" --password @keychain:"Application Loader: $APP_STORE_USERNAME"

xcrun altool --upload-app --file "$IPA_PATH" --username "$APP_STORE_USERNAME" --password @keychain:"Application Loader: $APP_STORE_USERNAME"
```

### Archive App

```
xcodebuild -project BullsEye.xcodeproj -scheme BullsEye -archivePath BullsEye.xcarchive archive
```

### Export Archive to ipa

https://medium.com/passei-direto-product-and-technology/from-xcode-to-testflight-using-command-line-288c3a85bd93

exportOptions.plist

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>development</string>
    <key>teamID</key>
    <string>ZWWMLG2G55</string>
    <key>teamName</key>
    <string>Kavish Punchoo (Personal Team)</string>
</dict>
</plist>
```

```
xcodebuild -exportArchive -archivePath BullsEye.xcarchive -exportPath BuildOutput -exportOptionsPlist exportOptions.plist
```



## Linux

### Repeat Command until output equals to

```bash
found=0
until [[ \$found == 1 ]]
do
    if ! oc get pods -l app=words-app -o json | grep '"items": \\[\\]'; then
    	found=1
    fi
	sleep 1
done
```



### Save Command Output to variable and Output to terminal at the same time

```
var=$(echo "hello" | tee /dev/tty)
```





## Git

### Generic

```
git branch -d <branch_name>
git push origin --delete <branch_name>

git checkout -b feature-1 develop

git push -u origin feature-1
```



## MSSQL

### Check Top Queries in MSSQL 19

```
Select 
     st.[text] AS [Query Text],          
     wt.last_execution_time AS [Last Execution Time],
     wt.execution_count AS [Execution Count],
     wt.total_worker_time/1000000 AS [Total CPU Time(second)],
     wt.total_worker_time/wt.execution_count/1000 AS [Average CPU Time(milisecond)],
     qp.query_plan,
     DB_NAME(st.dbid) AS [Database Name]
from 
    (select top 50 
          qs.last_execution_time,
          qs.execution_count,
   qs.plan_handle, 
          qs.total_worker_time
    from sys.dm_exec_query_stats qs
    order by qs.total_worker_time desc) wt
cross apply sys.dm_exec_sql_text(plan_handle) st
cross apply sys.dm_exec_query_plan(plan_handle) qp
order by wt.total_worker_time desc
```



## Android ADB

### adb shell into specific device

```
adb -s 'emulator-5554' shell
```



### Check AppPackage and AppActivity for Appium

run in adb shell

```
dumpsys window windows | grep -E 'mCurrentFocus'
```

