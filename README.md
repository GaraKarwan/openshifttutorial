# Instructions
## 1. __Create a gitlab repository__

Go to https://git-tos.intern.folksam.se/ and click the New Project button - choose Create blank project and configure it like so:
![gitlab-project](/exercise1/exercise1-gitlab-project.png)
Replace FTAL06 with your own user ID.

## 2. __Copy the Hello World application__

Create a folder called exercise1 in the root of your repository and copy the contents of this repository's /exercise1/hello-world folder to it. This folder contains a nodejs Hello World application.

## 3. __Create a Dockerfile__

Create a file named Dockerfile with no file extension (such as .txt) in your exercise1 folder.

Paste the following in to the Dockerfile:
```
FROM image-registry.openshift-image-registry.svc:5000/<namespace>/node:18
ENV NODE_ENV=production

WORKDIR /app

COPY ["package.json", "package-lock.json*", "./"]

RUN npm install --production

COPY . .

CMD [ "node", "server.js" ]

```
On the first line replace \<namespace\> with the name of the OpenShift project you are using.

## 4. __Create a buildconfig__

In order to build your image you will need to create a buildconfig. Create a folder called yaml and inside it create a file named bc\_hello-world.yaml and paste the following in to it:
```yaml
kind: BuildConfig
apiVersion: build.openshift.io/v1
metadata:
  name: <userid>-hello-world
  namespace: <namespace>
  labels:
    app: <userid>-hello-world
spec:
  resources:
    requests:
      memory: "1Gi"
      cpu: "4m"
    limits:
      memory: "1Gi"
  source:
    git:
      uri: 'https://git-tos.intern.folksam.se/<userid>/openshift-course.git'
      ref: main
    contextDir: "exercise1"
    type: Git
  output:
    to:
      kind: ImageStreamTag
      name: '<userid>-hello-world:latest'
  strategy:
    type: Docker
    dockerStrategy: {}
```
Replace the \<userid\> and \<namespace\> fields with your own userid and namespace.

You can now create the buildconfig in OpenShift by running the following oc command:
```
oc apply -f bc_hello-world.yaml
```
Make sure you run the command in the /exercise1/yaml folder.

## 5. __Create an imagestream__

Before you can build your image you have to create an imagestream to put it in. Create a file called is\_hello-world.yaml in your yaml folder and paste the following in to it:
```yaml
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: <userid>-hello-world
  namespace: <namespace>
  labels:
    app: <userid>-hello-world
spec:
  lookupPolicy:
    local: true
```
Replace the \<userid\> and \<namespace\> fields with your own userid and namespace.

Create the imagestream by running:
```
oc apply -f is_hello-world.yaml
```

## 6. __Build your image__

You are now ready to build your image. Run this command:
```
oc start-build hello-world
```
Your build should now have started. If you check the build status in the OpenShift portal you should notice that something has gone wrong. In the logs you should see this error:
```
error: failed to fetch requested repository "https://git-tos.intern.folksam.se/<userid>/openshift-course.git" with provided credentials
```

## 7. __Fix your build__

The repository you have created is private and in order for your build to retrieve the source from it you will need to give OpenShift access. You can do this by creating a deploy token in Gitlab. In your Gitlab repository go to: Settings -> Repository -> Deploy tokens and create a new deploy token with the following configuration:
![deploy-token](/exercise1/exercise1-deploy-token.png)

When you click the Create deploy token button you will be shown a generated token. You will need to create a secret in OpenShift with this value. Create a file called secret\_hello-world.yaml in your yaml folder and paste the following in to it:
```yaml
kind: Secret
apiVersion: v1
metadata:
  name: <userid>-hello-world-git-secret
  namespace: <namespace>
  labels:
    app: <userid>-hello-world
stringData:
  password: <the generated token>
  username: openshift-build-hello-world
type: kubernetes.io/basic-auth
```
Then create the secret with the following command:
```
oc apply -f secret_hello-world.yaml
```

You can now update your buildconfig file by adding the sourceSecret field:
```yaml
  source:
    git:
      uri: 'https://git-tos.intern.folksam.se/<userid>/openshift-course.git'
      ref: main
    contextDir: "exercise1"
    type: Git
    sourceSecret:
      name: <userid>-hello-world-git-secret
```
Remember to apply your buildconfig after you update the file.

Start a new build now and it should complete successfully. Verify that an image is created in your imagestream.


## 8. __Deploy your application__

Now you can deploy your application. Create a file called deployment\_hello-world.yaml in your yaml folder and paste the following in to it:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <userid>-hello-world
  namespace: <namespace>
  labels:
    app: <userid>-hello-world
spec:
  selector:
    matchLabels:
      app: <userid>-hello-world
  replicas: 1
  template:
    metadata:
      labels:
        app: <userid>-hello-world
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                      - <userid>-hello-world
                topologyKey: availability_zone
              weight: 100
      priorityClassName: test-low
      containers:
        - name: hello-world
          image: <userid>-hello-world:latest
          ports:
            - containerPort: 8080
          env:
            - name: HELLO_WORLD_PORT
              value: "8080"
          resources:
            requests:
              cpu: 4m
              memory: 30Mi
            limits:
              memory: 30Mi
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 30
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 5
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 30
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 5
```
Check the OpenShift portal and verify that the application is successfully deployed. 

## 9. __Create a service and route__

Now that your application is running we want to be able to use it. In order to make your application available outside of OpenShift we need to create two objects - a service and a route. A service acts as an internal loadbalancer to your pods and a route makes your application available on an external loadbalancer.

Create a file called svc\_hello-world.yaml in your yaml folder and paste the following in to it:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: <userid>-hello-world
  namespace: <namespace>
  labels:
    app: <userid>-hello-world
spec:
  selector:
    app: <userid>-hello-world
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```
also create a file called route\_hello-world.yaml:
```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: <userid>-hello-world
  namespace: <namespace>
  labels:
    app: <userid>-hello-world
spec:
  host: <userid>-hello-world.apps.intern.folksam.se
  to:
    kind: Service
    name: hello-world
  port:
    targetPort: 8080
```

Create both objects in OpenShift by running:
```
oc apply -f svc_hello-world.yaml
oc apply -f route_hello-world.yaml
```

You should now be able to reach your application at http://\<userid\>-hello-world.apps.intern.folksam.se. Open the URL in your browser and you should get a Hello World! message.

## 10. __Destroy everything__

Congratulations! You have successfully completed this exercise. Before you leave you should delete all the resources you have created. You can do this by running:
```
oc delete -f bc_hello_world.yaml
oc delete -f deployment_hello-world.yaml
oc delete -f is_hello_world.yaml
oc delete -f route_hello-world.yaml
oc delete -f secret_hello-world.yaml
oc delete -f svc_hello-world.yaml
```






