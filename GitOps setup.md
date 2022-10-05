# Introduction 
This repository is used to onboard OpenShift applications. 

If your team is fresh and new consumers, review the "Onboarding section" below to make
sure that you have the needed tools and fundations to consume the platform.

# Getting Started
You need to have a:
- system-ID which is registered in the CMDB
- gitlab repository which contains the OpenShift objects you want to deploy

### Clone the repository and create a merge request
Start by cloning the repository and creating a new branch
```
git clone https://git-tos.intern.folksam.se/cloud-and-container/public/openshift-onboarding/apps-tieto.git
cd apps-tieto/
git checkout -b my-app
```
Modify the values.yaml file and provide your application's info:
```yaml
  - name: example
    system_id: a01
    url: https://git-tos.intern.folksam.se/example-group/example-app.git
    path: .
    targetRevision: main
    project: a01-example
    env:
      - name: ci-cd
      - name: dev
        egress:
          - ip_range: 10.205.10.1/32
          - hostname: example.hostname.com
      - name: stst
      - name: atst
      - name: prod  
```

Commit your changes and push:
```
git commit -a -m "Added my-app"
git push --set-upstream origin my-app
```
After a successful push you can create a merge request [here](https://git-tos.intern.folksam.se/cloud-and-container/public/openshift-onboarding/apps-tieto/-/merge_requests). Select the branch you just created as the source branch and main as the target. Alternatively, you can use the link that you should have received as output after you successfully pushed your branch.

Once you have created a merge request, the projects pipeline will run and perform some validation by running helm lint and verifying that the system_id you supplied also exists in the TopDesk CMDB. If the validation passes then another job checks to see what kind of changes you made in the values.yaml file. If your change on consists of adding a new application then the pipeline will automatically merge the request. Otherwise, if you have either modified or removed any of the fields name, system_id, or env then the changes will require approval before by the Cloud & Container team before they are merged.

If your branch is merged to main then your application's namespaces will automatically be created by our Argo CD instance and you can begin deploying by adding content in the repo pointed to in the url variable.

NOTE: Currently we are unable to generate ISIM roles automatically to provide access to your namespaces. Therefore for the time being you will still have to contact the Cloud & Container team and request the ISIM roles to be created. We will automate this step as soon as there is an API available to generate roles in ISIM.

# Parameters

|Parameter|required|Type|Default|Note|
|---------|:------:|:--:|-------|----|
|name|Y|string||The name of your application.|
|system_id|Y|string||Your application's system-ID as listed in TopDesk|
|url|Y|string||The URL of the git repo that you want to use to deploy to OpenShift. Must end with .git |
|path|N|string|root / "."|The directory that you want to use in your git repo|
|targetRevision|N|string|main|The revision (branch, tag, commit) you want to use in your git repo|
|project|Y|string||The Argo CD project that your application will be placed in. Since it must be unique we recommend using system_id+name|
|env|Y|YAML list||A YAML formatted list of the environments you wish to create for your application. Each entry will generate a namespace in OpenShift with the format system_id-name-env|
|env.name.egress|N|YAML list||A YAML formatted list of the IP ranges or hostnames you wish to allow in the EgressNetworkPolicy for that environment|

# Onboarding

## Artifactory, create and consume your images correctly.

To build and consume images correctly on Folksam your application and team should have your own space in Artifactory. No build images or runtime images
should be pulled or used directly from the internet, but rather to be pulled and used via Artifactory.

If images from internet registries like dockerhub is used in builds and your application it can be pulled via Artifactory and pullthrough. This is a seamless
operation for you as consumer, but the win is huge, we get security scanning on all images we use, we get a local cache of images so we dont max out external
pull ratios, and a unified way of working across all teams.

For enable:ing this:

* Order your own Artifactory space from the faf team in Topdesk [here](https://folksam.topdesk.net/tas/public/ssp/content/detail/service?unid=954fda82b2f9426b84142ade5658065c)
* Tick in the checkbox Yes on "Do you require a service account with access to the service(s)?"
* If you have an upstream provider rather then dockerhub order an external pulltrough registry, provide auth information for it if nessasary.
* When you have used our onboarding and required your project in OpenShift. Create a pullsecret for your project:

```yaml
oc create secret docker-registry folksam-artifactory \
    --docker-server=<DEVTEAM>-virtual.repo-tos.intern.folksam.se \
    --docker-username=<DEVTEAM>-artifactory \
    --docker-password=xxxxxxxxx \
    --docker-email=<DEVTEAM>-artifactory@folksam.se

oc secrets link default folksam-artifactory --for=pull

```
* Point out you image to use in your build or deployment, like: <DEVTEAM>-virtual.repo-tos.intern.folksam.se/<image_url_to_use>

## Gitlab, your git space & CI/CD automation for your application.

Your team should have your own Gitlab space for your sourcecode and to house your pipelines and automation to build and deploy your application for OpenShift.
This is where our GitOps automation will check for changes and sync them to the platform.

For enable:ing this:

* Order your own Gitlab space from the faf team in Topdesk [here](https://folksam.topdesk.net/tas/public/ssp/content/detail/service?unid=954fda82b2f9426b84142ade5658065c)
* Add and grant access to your teammembers.
* Review example templates of CI/CD pipelines for your application [here](https://git-tos.intern.folksam.se/ci-templates/templates)

## Your are good to go! but if you still have questions

* Read our onboarding documentation and view our examples [here](https://folksam.sharepoint.com/teams/CloudContainerPlatformTeamver3.0/SitePages/OpenShift.aspx)
* Read RedHats documentation for the platform [here](https://docs.openshift.com/container-platform/4.8/welcome/index.html)
* Contact the Cloud & Container team in Topdesk [here](https://folksam.topdesk.net/tas/public/ssp/content/detail/service?unid=3778e065fdba46bcbd0413508f835d91) Book a consultation or register an incident on problems.

