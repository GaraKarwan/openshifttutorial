# Instructions
## 1. __Create a namespace__

You will need to have a GitOps enabled namespace for this exercise. You can create one by following the instructions at https://git-tos.intern.folksam.se/cloud-and-container/public/openshift-onboarding/apps-tieto. However, instead of copying the example given in those instructions, you should create an application with this information:
```yaml
  - name: <userid>
    system_id: f36
    url: https://git-tos.intern.folksam.se/<userid>/openshift-course.git
    path: exercise2
    targetRevision: main
    project: f36-<userid>
    env:
      - name: course
```

## 2. __Create an ArgoCD application__

In your repository (https://git-tos.intern.folksam.se/\<userid\>/openshift-course.git) create a file called Chart.yaml in the exercise2 folder:
```yaml
apiVersion: v2
name: helloworld
description: A collection of App of Apps for deploying helloworld
type: application
version: 1.0.0
dependencies:
- name: argo-application
  version: "1.0.2"
  repository: "https://repo-tos.intern.folksam.se/helm-charts/"
```

Then create a file called values.yaml in the exercise2 folder:
```yaml
argo-application:

  source: https://git-tos.intern.folksam.se/<userid>/openshift-course.git
  project: f36-<userid>
  env: gitops

  applications:
    - name: <userid>-course-app-of-helloworld
      source_path: "exercise2/hello-world"
      destination: f36-<userid>-course
```

Once these two files have been created in your repository, ArgoCD will automatically create an ArgoCD application called \<userid\>-course-app-of-helloworld. This ArgoCD application is configured to synchronize any YAML files in the /exercise2/hello-world folder.

## 3. __Create your Hello World application using GitOps__
In your repository, create a folder called hello-world in the exercise2 folder and copy the YAML files from the /exercise1/yaml folder to /exercise2/hello-world. You will need to remove the namespace field from all of your YAML files since it will be controlled by the destination field provided in the values.yaml file.

Once the files have been pushed to your repository then ArgoCD will begin to synchronize the files to OpenShift. Go to https://gitops.apps.intern.folksam.se and login by clicking the "Log in via OpenShift" button. Once logged in you should be able to see two applications, one called f36-\<userid\> and one called gitops-\<userid\>-course-app-of-helloworld. Click the later and you should get an overview of the status of your Hello World application.

ArgoCD will not automatically trigger a build when it synchronizes the buildconfig. Therefore you will have to run a build manually in order to create the hello-world image.

Once you have built an image and your application status in ArgoCD is healthy then You should be able to reach your application at http://\<userid\>-hello-world.apps.intern.folksam.se. Open the URL in your browser and you should get a Hello World! message.

## 4. __Destroy everything__
Congratulations! You have successfully completed this exercise. Before you leave you should delete all the resources you have created. You can do this by creating a merge request to remove your application in the apps-tieto repository (https://git-tos.intern.folksam.se/cloud-and-container/public/openshift-onboarding/apps-tieto).
