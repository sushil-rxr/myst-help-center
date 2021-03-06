## {{page.title}}

Yes.

In this use case, we will do the following:

1. Redirect all traffic to node 1
2. Take node 2 out of the domain
3. Upgrade node 2 to OSB BP 5
4. Redirect all traffic to node 2
5. Verify working of upgraded node 2
6. Upgrade node 1 to OSB BP 5
7. Put node 1 and 2 back into the domain
8. Redirect traffic to both nodes

Example: Upgrading OSB to Bundle Patch 5 using a rolling restart approach

## One Time Configuration Steps {#HowdoIperformrollingrestartforpatcheswithMySTStudio?-OneTimeConfigurationSteps}

### Retrieve API key from MyST Studio console {#HowdoIperformrollingrestartforpatcheswithMySTStudio?-RetrieveAPIkeyfromMySTStudioconsole}

1. Login to MyST Studio with an administrator account
2. Click on "Administration" then select "Users"
3. Under the MySTAdministrator \(API User\) click on drop-down and select "Show API Key"
![](img/howto-patch-rollstart-1.show-api-key.png)
4. Copy the key, we will use this later in our MyST workspace. If you want, you can generate a new key at any time.
![](img/howto-patch-rollstart-2.api-key-view.png)

### Create and version control the MyST Workspace

MyST CLI has the concept of a MyST workspace which is a metadata directory that can be version controlled. In this example we will use Git.

For non-Studio customers, the MyST workspace is used for storing Platform Blueprint and Models in XML or Properties format. For Studio customers, the Platform Blueprints and Models are version-controlled in the MyST Studio repository. Platform Blueprints and Models can be retrieved from MyST Studio  by defining MyST Studio connectivity details within the MyST workspace. Common use cases for this approach are:

* Creating CI server jobs that perform operations using configuration data from MyST Studio
* Using a hybrid of configuration and deployment data from a version control system like Git and MyST Studio \(e.g. Secure platform connectivity details are stored in MyST Studio but the deployment data is stored on the file system in the MyST workspace\)
* Creating MyST custom action extensions which use configuration data from MyST Studio. 

In our rolling restart use case, it is ideal to use a CI server to orchestrate  custom operations outside of core MyST such as the load balancer re-routing. We will use a MyST workspace. If you don't have an existing MyST workspace, you can create an version control one as follows:

<!-- NOTE: I'm intentionally breaking the number list format because markdown can't retain numbering order across the complex blocks of text below : gokelly -->

1 . Create a new folder for our MyST workspace.
                                                                        
```
mkdir myst-workspace
```

2 . Navigate to the myst-workspace and initialize it using MyST                         
```
cd myst-workspace
myst init
```
This will provide an output similar to the following:

```
MYST_HOME=/opt/myst
MYST_WORKSPACE=/u01/app/oracle/admin/myst-workspace

------------------------------------------------------------------------
 R U B I C O N >< R E D                             MyST v3.6.2.1 (149)
...delivery...deMySTified..............................................
(c) 2011-2016 Rubicon Red Software Pty Ltd. All rights reserved
------------------------------------------------------------------------
Registered to : EMAILADDRESS=support@rubiconred.com, CN=Rubicon Red, L=Brisbane, ST=QLD, C=Brisbane
                                 Expires : Tue Aug 20 02:44:07 EDT 2019
                                    Session ID : 05-04-16-22-14-15-8134
------------------------------------------------------------------------

Analysing request...

MyST workspace created at /u01/app/oracle/admin/myst-workspace

SUCCESS - 1 Second                                                                                      
```

3 .  Create a file relative to the myst-workspace at resources/myst.properties

Add the details for your MyST Studio instance and include the API key that you copied before:

```
myst.studio.hostname=<MYST STUDIO HOST>
myst.studio.port=8085
myst.studio.api.key=<API KEY>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       
```
4 . Run the following command to display the myst usage

```
myst usage
```

This should retrieve all of the available Platform Models from MyST Studio. These platform models are named in the following convention:

`env.<environment type>.<platform model>.platform.` 

For example:

```
Configuration           Description
-------------           -----------
env.ci.soa.platform     CI SOA Environment
env.dev.soa.platform    Development SOA Environment
env.uat.soa.platform    User Acceptance Testing Environment
env.prod.soa.platform   Production SOA Environment
```

Take note of the Platform Model which you want to use for the minimal downtime patching.

5 . Execute the MyST properties action on your platform model of choice by performing the following:

```
myst properties env.prod.soa.platform
```

Take note of the unique server logical IDs, you will use these later to specific the rolling in a sequence of your choice. In the example below, which is a 2 node OSB cluster, we have osb.server1 and osb.server2

![](img/howto-patch-rollstart-3.server-logical-ids.png)

Also, take note of the unique node logical IDs for each OSB node. These are defined as a comma-separated list under product.osb.node-list. In our example, lets assume the nodes are called node1 and node2.

6 . Lastly, version control your MyST workspace using your Version Control System of choice. In this example, we are using Git

```
git add myst-workspace/conf
git add myst-workspace/resources
git commit -m "Added MyST workspace"
git push origin master
```

> NOTE:  From the previous execution there will be a logs directory created under myst-workspace. You should avoid version controlling this file by following the steps above to only version control the conf and resources directories.

### Create the Jenkins job to orchestrate the MyST patching with rolling restarts {#HowdoIperformrollingrestartforpatcheswithMySTStudio?-CreatetheJenkinsjobtoorchestratetheMySTpatchingwithrollingrestarts}

Jenkins is a CI server which allows for orchestrating multiple steps in a single job which can be executed on demand or as part of a schedule. The same principles would apply for setting up an orchestration in another CI server or dashboard such as Rundeck. In the example below it is assumed that MyST is installed under /opt/myst on the master \(or slave\) where the Jenkins job is being run under.

<!-- NOTE: I'm intentionally breaking the number list format because markdown can't retain numbering order across the complex blocks of text below : gokelly -->

1 . From the Jenkins console, click on "New Item"

2 . Create a "Freestyle project" 

3 . Select the SCM which you wish to pull the myst-workspace from. In our example, it is a Git repository
![](img/howto-patch-rollstart-4.choose-scm.png)

4 . Click on "Add build step" and select "Execute shell" 
![](img/howto-patch-rollstart-5.scm-execute-shell.png)

> NOTE:  Alternatively, you can use the MyST CLI plugin for Jenkins. In order to provide instructions that would be compatible for other CI server and orchestration tools, we are going to use the generic "Execute shell" option in our example.

5 . Under "Execute shell", enter your desired patching sequence. An example matching the use case described at the start of this document is listed below. In this example, osb.server1 and osb.server2 are on node1 and osb.server3 and osb.server4 are on node2

```
cd myst-workspace
MYST_HOME=/opt/myst
PATH=$PATH:$MYST_HOME
MYST_WORKSPACE=/u01/app/oracle/admin/myst/jenkins/home/jobs/PROD.RollingRestart/workspace/myst-workspace
CONFIGURATION=env.ci.2_node.platform
 
# 1. Redirect all traffic to node 1
# TODO: Update the Load Balancer to redirect all traffic to node 1
  
# 2. Take node 2 out of the domain
myst stop-via-as $CONFIGURATION -Dservers=osb.server3,osb.server4
 
# 3. Upgrade node 2 to OSB BP 5
myst patch $CONFIGURATION -Dproduct.osb.node-list=node2
myst start-via-as $CONFIGURATION -Dservers=osb.server3,osb.server4
# If servers fail to start in the previous step due to an issue with the upgrade it will
#  automatically cause this job to abort. In this case, a revert of the patch(es) and
#  restart of node 2 can be performed by a separate patch revert job in Jenkins (or MyST Studio).
 
# 4. Redirect all traffic to node 2
# TODO: Update the Load Balancer to redirect all traffic to node 2
 
# 5. Verify the upgraded node 2 instance is working as expected
# If desired, you can add automated tests to run at this point
 
# 6. Upgrade node 1 to OSB BP 5
myst patch $CONFIGURATION -Dproduct.osb.node-list=node1
 
# 7. Put node 1 back into the domain
myst stop-via-as $CONFIGURATION -Dservers=osb.server1,osb.server2
myst start-via-as $CONFIGURATION -Dservers=osb.server1,osb.server2
 
# 8. Redirect traffic to both nodes
# TODO: Update the Load Balancer to redirect all traffic in it's original state (between node 1 and 2)
```

Placeholders have been left in the script above for automated tests and interaction with the load balancer for redirects. If the load balancer interactions are intended to be manual, you may wish to split the above up into multiple CI jobs which are triggered at the appropriate times after each manual load balancer configuration step. This same approach could also be done with MyST Studio alone and no CI server required as detailed in the section "Alternative Approach: Applying the patch and rolling restart exclusively with MyST Studio"  

6 . Save the job



## Orchestrating a fully-automated patch rolling restart from Jenkins using MyST {#HowdoIperformrollingrestartforpatcheswithMySTStudio?-Orchestratingafully-automatedpatchrollingrestartfromJenkinsusingMyST}

This part is simple, just go to the job in Jenkins and click the run button when you want to perform the fully-automated patch rolling restart.

## Alternative Approach: Applying the patch and rolling restart directly with MyST Studio as individual steps {#HowdoIperformrollingrestartforpatcheswithMySTStudio?-AlternativeApproach:ApplyingthepatchandrollingrestartdirectlywithMySTStudioasindividualsteps}

If you need to perform manual steps during the process such as redirecting the load balancer or running manually sanity checks, the Jenkins approach would not be ideal. In this case, you can simply perform the operations directly from MyST Studio as required as part of the overall manual sequence. 

For example, you could run the "Update" operation to apply the patch using the out-of-the-box feature in MyST Studio. Then when it comes to restarting certain nodes, you could use the "Control" operation to run the actions such as stop-via-as and start-via-as with the given overrides.





