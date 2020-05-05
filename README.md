# mygovbc

The structure of this repository is compatible with the build and deployment mechanisms in Red Hat OpenShift (Origin/Enterprise/Container Platform).  

## Pre-requisites

In order to complete the steps below, the following pre-requisites must be fulfilled.

1. Project(s) must exist in which the build and deployment resources can be created
1. You must have edit or admin access to the projects
1. An imagestream must exist in the project that will host the build resrouce for the S2I image that will be used to build the wildfly image.  See below for the steps to do this.  

### Pull and push an image

If no project exists that contains the S2I imagestream needed to build the wildfly project, you can:
 
 * (**RECOMMENDED**) use the script [here](https://github.com/BCDevOps/openshift-tools/blob/master/utils/import-image.sh) to import the image automatically into the OpenShift Docker registry
 
 or...
 
 * import the image manually from DockerHub following the instructions [here](https://github.com/BCDevOps/issues-and-solutions/wiki/Tips-and-Tricks#pushing-an-image-to-openshift-internal-registry)
 
## Initial Setup

There are two initial build/deployment models possible with OpenShift - please see below to determine which is apprpriate for your purposes.

### One Step Build and Deployment (combined build and deployment project)

**Note:** The model decribed here is primarily for validation/testing of the repository contents and structure. For example, use by developers within the Red Hat Container Development Kit  The "multi-project" process described below is the intended model for use within BC Gov.   

- Assuming you have the OpenShift ```oc``` tools installed and you've authenticated against an OpenShift instance, the application in this repository can be built and deployed within your current OpenShift project using the following command line:

```
oc new-app openshift/wildfly-100-centos7~https://github.com/bcgov/mygovbc.git
```

- The command above will result in a BuildConfiguration, DeploymentConfiguration and other supporting resources being created in OpenShift. A build will also be triggered and the application will be deployed once the build completes. The build is performed using the [Wildfly S2I](openshift/wildfly-100-centos7) builder image.

### Multi-project Build and Deployment (separate build and deployment projects)

**Note:** This model is intended to be used for "official" BC Gov projects in OpenShift.  The pattern it uses is separate OpenShift projects for each logical deployment environment (e.g. DEV, TEST, PROD) **plus** and additional "TOOLS" project for build/CI tooling (such as Jenkins) and related OpenShift resources. 

#### Build Configuration

- Assuming you have the OpenShift ```oc``` tools installed and you've authenticated against an OpenShift instance, a BuildConfiguration for this app can be created within your OpenShift "TOOLS" project using the following command line:

```
cd openshift/templates
oc project gcpe-mygovbc-tools
oc process -f ./mygov-wildfly-bc-template.json -v NAME=mygovbc-app -v BUILDER_IMAGESTREAM_TAG=wildfly-100-centos7:latest -v SOURCE_REPOSITORY_URL=https://github.com/bcgov/mygovbc.git | oc create -f -
```

**Note:** There are additional parameters available which can be seen in the "parameters" section of the file in this repo at openshift/templates/mygov-wildfly-bc-template.json

#### Performing Builds

Builds will be performed via an automated GitHub webhook (triggered on every push to the bcgov/mygovbc GitHub repository), or when manually requested the OpenShift web UI, the ```oc``` tools.
 
**Note:** The values of the "-v" parameters in the command above can be adjusted to suit your preferences, as well as the value "gcpe-mygovbc-tools", if you have names your "TOOLS" project differently.

#### Deployment Configuration

To configure a deployment environment (e.g. one of DEV, TEST, PROD) including database, storage, deployment triggers, etc. **within an appropriate OpenShift project** the commands below may be followed, adapting the values as appropriate for your target environment.  The example below is for a "DEV" environment, and the values for APPLICATION_DOMAIN, APP_DEPLOYMENT_TAG, "gcpe-mygovbc-dev" can be adjusted as needed. 

```
cd openshift/templates
oc project gcpe-mygovbc-dev
oc process -f mygov-wildfly-environment-template.json -v APPLICATION_DOMAIN=mygovbc-dev.pathfinder.bcgov -v APP_DEPLOYMENT_TAG=dev -v DATABASE_VOLUME_CAPACITY=1Gi | oc create -f -
``` 

**Note:** There are additional parameters available which can be seen in the "parameters" section of the file in this repo at openshift/templates/mygov-wildfly-environment-template.json

#### Performing Deployments 

Deployments are performed within each environment based on OpenShift "ImageChange triggers" that are activated when a tag associated with a deployment environment is applied to a different image revision.  By convention, the tag is the lowercase logical name of the environment (e.g. "dev", "test", or "prod").  The ```oc``` tools provide a "tag" command to apply tags.  

As an example, to tag an image with the "dev" tag in order to cause a deployment in the gcpe-mygovbc-dev project, the following command would be used.  In order to deploy within another environment, "dev" would be replaced with the lowercase logical name of the environment (e.g. test or prod).
 
```
oc project gcpe-mygovbc-tools
oc tag mygovbc-app:latest mygovbc-app:dev
```

### First Deployment

Prior to a first deployment within an environment, some one-time steps are required.  
 
 The first step is granting "image-puller" access to an environment's service account on the "TOOLS" project.  This is done as follows, adjusting the "gcpe-mygovbc-dev" to match the name of the target deployment environment.
  
  ```
  oc policy add-role-to-user system:image-puller system:serviceaccount:gcpe-mygovbc-dev:default -n gcpe-mygovbc-tools
  ```

The MyGovBC application will not deploy cleanly within Wildfly upon first deployment due to missing properties files required by the pspd/mygov app.  These must be loaded once manually, and will persist across subsequent deployments.  The command to put the files in place is as follows:

**Note:*** The command below assumes you have a local copy of the properties files in a local directory called "config". Adjust the command below as appropriate if this is not the case. Also, if you are not running in a bash shell, you will need to manually determine the name of the pod running the app and use it in place of the inline oc command.
  
```
oc rsync config/ $(oc get pods --no-headers -l deploymentconfig=mygovbc-app | awk '{print $1}'):/pspdConfig
```

#### Listing Values of Generated Environment Variables

Several values are dynamicall generated in the process of crating the deployment configuration described above.  Within components in the deployment these values are made available by shared environment variables and the components are confgured to use them without further intervention.  However, these values may be needed by developers to interact with application components.  The command below will list the values for use in these cases.  Note "gcpe-mygovbc-dev" should be replaced if not targeting the "dev" environment as each environment will have distinct values.  

```
oc env dc/mygovbc-app --list -n gcpe-mygovbc-dev
```

The output will be similar to below.

~~~~
PSPD_CONFIGURATION=/pspdConfig
MYSQL_USER=******
MYSQL_DATABASE=*****
MYSQL_PASSWORD=*******
JAVA_OPTS=-Xms1024m -Xmx6g -XX:MaxPermSize=1024m -XX:ReservedCodeCacheSize=64m -Djava.net.preferIPv4Stack=true -Dorg.jboss.resolver.warning=true -Dsun.rmi.dgc.client.gcInterval=3600000 -Dsun.rmi.dgc.server.gcInterval=3600000
WILDFLY_ADMIN_PASSWORD=*******
~~~~

The values may be changed using the ```oc env``` command.


 
