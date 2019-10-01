# easyTravel-Openshift

![easyTravel Logo](https://github.com/dynatrace-innovationlab/easyTravel-Builder/blob/images/easyTravel-logo.png)

This project builds and deploys the [Dynatrace easyTravel](https://community.dynatrace.com/community/display/DL/Demo+Applications+-+easyTravel) demo application on [Openshift](https://www.openshift.com/), and uses [Ansible Tower](https://www.ansible.com/) to remidiate any issues with the project when it is all setup and running.

What we are going to do through this project is:

* Setup and configure the Dynatrace Dashboard,
* Install the Dynatrace Openshift operator,
* Use that operator to install the Dynatrace OneAgent (which will then automatically monitor our cluster and any running applications),
* Install a web application (made up of a DB, backend, frontend and a web server) called easyTravel
* Install Ansible Tower on Openshift
* Monitor the easyTravel web app, detect any issues and fix them automatically with Ansible Tower.

## Project  Components

| Component          | Description
|:-------------------|:----------------------------------------------------------
| easyTravel         | Dynatrace's containerised travel website demonstration application.ß
| Openshift Container Platform     | Build, deploy and manage your container-based applications consistently across cloud and on-premises infrastructure.
| Dynatrace OneAgent | Dynatrace agent that collects and unifies operational and business performance metrics.
| Ansible Tower      | Software provisioning, configuration management, and application-deployment tool.

## Before we begin

I don't know about you, but I find it too common that project and product documentation assumes you've done this all before, and you are only reading the docs to be reminded of what to do next. I hope to not make you feel that way with this document, though I admit there is one part (coming up soon) where I assume you've already had some experience with Openshift and can be let loose without any hand holding. If you feel I have missed the mark, throw a PR my way.

Now, onto the fun stuff!

Ensure you have access to a Dynatrace dashboard, if not, sign up for a free trial: https://www.dynatrace.com/trial/

Oh, and you will need access to an Openshift Container Platform cluster - make sure you are logged into this cluster (as an appropriate user) before going any further. Setting up a cluster is beyond the scale of this project, but installing one using AWS is very easy:

 - https://docs.openshift.com/container-platform/4.1/installing/installing_aws/installing-aws-default.html

Make sure you have the Openshift CLI installed too:

- https://cloud.redhat.com/openshift/install

And now would be a good time to install Ansible on your local machine:

- https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html

## Install the Dynatrace Operator and OneAgent inside Openshift

Dynatrace provides an operator for Openshift that allows for the monitoring of the full stack (i.e fom the application down the infrastrucure) of your Openshift environment, through the automated installation of OneAgent. If you don't have access to the infrastructure layer, you can also just deploy OneAgent inside your application. This project will utilise the operator for full stack monitoring, and it is left as an exercise to the reader to implement application-only monitoring.

First things first, you will need to obtain two tokens (API_TOKEN and PAAS_TOKEN) from your Dynatrace dashboard:
https://www.dynatrace.com/support/help/reference/dynatrace-concepts/what-is-an-access-token/

You will need these tokens to create a secret for the OneAgent to use, and a project (which we want to call "dynatrace") to install the Dynatrace operator into:

```
oc new-project dynatrace
```

```
oc -n dynatrace create secret generic oneagent --from-literal="apiToken=API_TOKEN" --from-literal="paasToken=PAAS_TOKEN
```

To install the Dynatrace operator, the easist thing to do now is log into the Openshift Container Platform dashboard. Once in:

1. Change into the "dynatrace" project
2. Click on Catalog
3. Click on OperatorHub
4. Locate and install the Dynatrace Operator
5. Click on Installed Operator (though this might take a while to be populated with the Dynatrace OneAgent)
6. Click on Create New (the YAML editor should pop-up)
7. Modify the apiUrl: 'https://ENVIRONMENTID.live.dynatrace.com/api' line to point to your Dynatrace dashboard URL.
8. Click create.

After a while your Dynatrace OneAgent pods will be built and you can log into your Dynatrace dashboard to see information about your environment start to filter in. Whilst you are in your Dynatrace dashboard, now would be a good time to make a couple of configuration changes to ensure Dynatrace is reporting back everything it can on your Openshift pods (and later, easyTravel application).

From the Settings page:

1. Click on Processes and containers
2. Click on Container monitoring
3. Ensure CRI-O containers is enabled (you can choose to select other options here now, though I wouldn't bother with the Windows one for this project)


Must go and change monitoring configuration to include CRI-O and Java technologies.


## easyTravel Application Components

| Component | Description
|:----------|:-----------
| mongodb   | A pre-populated travel database (MongoDB).
| backend   | The easyTravel Business Backend (Java).
| frontend  | The easyTravel Customer Frontend (Java).
| nginx     | A reverse-proxy for the easyTravel Customer Frontend (NGINX).
| loadgen   | A synthetic UEM load generator (Java).

## Run easyTravel in Openshift

The original version of this demo deployed a number of Docker containers, but as we are going to be using Openshift our deployment platform, we need to break things up into seperate dockerfiles (note: the URIs used in the orignal docker-compose file have been changed to work with the internal Openshift DNS). You can do this quickly and easily by converting the existing `docker-compose.yml` file with [Kompose](https://kompose.io/), like so:

```
kompose convert
```

Once that is done, you should have the following files:

| yaml     |
|:----------
| backend-service.yaml,frontend-service.yaml
| mongodb-service.yaml
| www-service.yaml
| backend-deployment.yaml
| frontend-deployment.yaml
| loadgen-deployment.yaml
| mongodb-deployment.yaml
| www-deployment.yaml  

Before we do anything else, we will want to configure our project in Openshift, and install the  Dynatrace operator. First off, lets create a project using the [oc cli](https://cloud.redhat.com/openshift/install):

```
oc new-project easytravel
```
Great, awesome work. Now let's actually deploy and build everyting:

```
oc apply -f backend-service.yaml,frontend-service.yaml,mongodb-service.yaml,www-service.yaml,backend-deployment.yaml,frontend-deployment.yaml,loadgen-deployment.yaml,mongodb-deployment.yaml,www-deployment.yaml
```

Then open up some routes so that the website is accessible to uses:

```
oc expose service/frontend
```

```
oc expose service/www
```

Note: as of writing, additional work is required to get the www pod to run. I will update this with detailed steps ASAP.

## Configure easyTravel in Openshift

Aligning with principles of [12factor apps](http://12factor.net/config), one of them which requires strict separation of configuration from code, easyTravel can be configured at startup time via the following environment variables:

| Component | Environment Variable  | Defaults                       | Description
|:----------|:----------------------|:-------------------------------|:-----------
| backend   | ET_DATABASE_LOCATION  | mongodb:27017       | The location of the database the easyTravel Business Backend shall connect to.
| frontend  | ET_BACKEND_URL        | http://backend:8080 | The URL to easyTravel's Business Backend.
| nginx     | ET_FRONTEND_LOCATION  | frontend:8080       | The location of the Customer Frontend the easyTravel WWW server shall serve via port 80.
| nginx     | ET_BACKEND_LOCATION   | backend:8080        | The location of the Business Backend the easyTravel WWW server shall serve via port 8080.
| loadgen   | ET_WWW_URL            | http://www:80       | The URL to easytravel's Customer Frontend.
| loadgen   | ET_BACKEND_URL        | http://www:8080     | The URL to easyTravel's Business Backend (optional). If provided, the problem patterns provided in `ET_PROBLEMS` will be applied consecutively for a duration of 10 minutes each.
| loadgen   | ET_PROBLEMS           | BadCacheSynchronization,<br/>CPULoad,<br/>DatabaseCleanup,<br/>DatabaseSlowdown,<br/>FetchSizeTooSmall,<br/>JourneySearchError404,<br/>JourneySearchError500,<br/>LoginProblems,<br/>MobileErrors,<br/>TravellersOptionBox | A list of supported problem patterns, see below on how to activate.
| loadgen   | ET_PROBLEMS_DELAY     | 0                              | A delay in seconds. When used with Dynatrace, it is suggested to use a value of 7500 (slightly more than 2 hours) so that Dynatrace can learn from an error-free behavior first.
| loadgen<br/>backend<br/>frontend | ET_APM_SERVER_DEFAULT | Classic | The type of used server. Can be "APM" for Dynatrace and "Classic" for AppMon
| loadgen   | ET_VISIT_NUMBER       | 5                              | The number of visits generated per minute

## Enable easyTravel Problem Patterns

The following problem patterns are supported and triggered through the *loadgen* component, as described above:

| Pattern                 | Description
|:------------------------|:------------
| BadCacheSynchronization | Activating this plugin causes synchronization problems in the customer frontend and uses a lot of CPU by doing an inefficient cache lookup. Activating this plugin should show a class 'CacheLookup' as doing lots of synchronization.
| CPULoad                 | Causes high CPU usage in the business backend process to provoke an unhealthy host health state. The additional CPU time is triggered in 8 separate threads independent of any searching/booking activity.
| DatabaseCleanup         | Cleans out items where we continuously accumulate data in the Database, e.g. Booking and LoginHistory and keeps the last 5000 to avoid filling up the database over time. This is done every 5 minutes at the point where a Journey is searched. Usually this plugin is enabled by default, if you disable it, the database will fill up over time, especially if traffic is generated automatically.
| DatabaseSlowdown        | Plugin which causes queries on locations to take longer.
| FetchSizeTooSmall       | This plugin sets the fetchsize of the Hibernate persistence layer to 1 when executing database queries. This will cause inefficient select statements to show up on databases where otherwise Hibernate is optimizing fetches into bulks.
| JourneySearchError404   | Causes an HTTP 404 error code by returning an image name that does not exist when searching for journeys in the customer frontend web application.
| JourneySearchError500   | Throws an HTTP 500 server error if the journey search parameters are invalid, e.g. toDate is before fromDate
| LargeMemoryLeak         | Causes a large memory leak in the business backend when locations are queried for auto-completion in the search text box in the customer frontend. Note: This will quickly lead to a non-functional Java backend application because of out-of-memory errors.
| LoginProblems           | Simulates an execption when a login is performed in the customer frontend application.
| MobileErrors            | Journey searches and bookings from mobile devices create errors. (no errors created for Tablets)
| TravellersOptionBox     | Causes an 'ArrayIndexOutOfBoundsException' wrapped in an 'InvalidTravellerCostItemException' if in the review-step of the booking flow in the customer frontend, the last option '2 adults+2 kids' is selected in the combo-box for 'travellers'.

## Monitor easyTravel with Dynatrace Openshift Operator

Please refer to the [Red Hat Openshift monitoring](https://www.dynatrace.com/technologies/openshift-monitoring/) page for more information.

Must go and change monitoring configuration to include CRI-O and Java technologies.

## Remidiate easyTrabel with Red Hat Ansible Tower
To really make things swing, we are going to install Red Hat Ansible Tower to keep our application running as expected at all time. If Dynatrace detects a problem with an easyTravel container, Tower will be called to run a [Playbook](https://docs.ansible.com/ansible/latest/user_guide/playbooks.html) to remediate the issue. Before we begin, make sure your Openshift cluster has enough resources to support Tower. This might seem a silly thing to mention, but trust me, go check.

Create a Persitent Volume Claim for postgres - refer to postgres-nfs-pvc.yaml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgresql
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Create the claim:
```
oc create -f postgres-nfs-pvc.yaml
```

Check the claim:
```
oc get pvc
```

Refer to the [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) documentation to install Ansible on your local machine.

Download and untar the  Ainsible Tower installer:
  https://releases.ansible.com/ansible-tower/setup_openshift/

Open that newly created folder and open the "inventory" file. Make changes appropriate for your Openshift installation.

Install Ansible Tower on Openshift

```
./setup_openshift.sh
```

## Problems? Questions? Suggestions?

This offering is [Dynatrace Community Supported](https://community.dynatrace.com/community/display/DL/Support+Levels#SupportLevels-Communitysupported/NotSupportedbyDynatrace(providedbyacommunitymember)). Feel free to share any problems, questions and suggestions with your peers on the Dynatrace Community's [Application Monitoring & UEM Forum](https://answers.dynatrace.com/spaces/146/index.html).

## License

Licensed under the MIT License. See the [LICENSE](https://github.com/djamryl/easyTravel-Docker/blob/master/LICENSE) file for details.
