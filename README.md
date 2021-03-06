#  Build and deploy a Java Springboot microservice application on Azure Container Service (AKS)

In a nutshell, you will work on the following tasks.
1.  Define a **Build Pipeline** in VSTS (Visual Studio Team Services).  Execute the build pipeline to package a containerized Springboot Java Microservice Application (**po-service 1.0**) and push it to ACR (Azure Container Registry).  This task focuses on the **Continuous Integration** aspect of the DevOps process.  Complete Steps [A] thru [C].
2.  Deploy an AKS (Azure Container Service) Kubernetes cluster and manually deploy the containerized microservice application on AKS.  Complete Step [D].
3.  Define a **Release Pipeline** in VSTS.  Execute both build and release pipelines in VSTS in order to update and re-deploy the SpringBoot microservice (**po-service 2.0**) application on AKS.  This task focuses on the **Continuous Deployment** aspect of the DevOps process.  Complete Step [E].

This Springboot application demonstrates how to build and deploy a *Purchase Order* microservice (`po-service`) as a containerized application on Azure Container Service (AKS) on Microsoft Azure. The deployed microservice supports all CRUD operations on purchase orders.

For easy and quick reference, readers can refer to the following on-line resources as needed.
- [Spring Getting Started Guides](https://spring.io/guides)
- [Docker Documentation](https://docs.docker.com/)
- [Kubernetes Documentation](https://kubernetes.io/docs/home/?path=users&persona=app-developer&level=foundational)
- [Creating an Azure VM](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-cli)
- [Azure Container Service (AKS) Documentation](https://docs.microsoft.com/en-us/azure/aks/)
- [Azure Container Registry Documentation](https://docs.microsoft.com/en-us/azure/container-registry/)
- [Visual Studio Team Services Documentation](https://docs.microsoft.com/en-us/vsts/index?view=vsts)
- [Install Azure CLI 2.0](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)

**PREREQUISITES:**
1.  An active **Microsoft Azure Subscription**.  You can obtain a free Azure subscription by accessing the [Microsoft Azure](https://azure.microsoft.com/en-us/?v=18.12) website.  In order to execute all the labs in this project, either your *Azure subscription* or the *Resource Group* **must** have **Owner** Role assigned to it.
2.  A **GitHub** Account to fork and clone this GitHub repository.
3.  A **Visual Studio Team Services** Account.  You can get a free VSTS account by accessing the [Visual Studio Team Services](https://www.visualstudio.com/team-services/) website.
4.  To connect your VSTS project to your Azure subscription, you may need to define a **Service Endpoint** in VSTS.  Refer to the article [Service endpoints for builds and releases](https://docs.microsoft.com/en-us/vsts/pipelines/library/service-endpoints?view=vsts).  Review the steps for [Azure Resource Manager service endpoint](https://docs.microsoft.com/en-us/vsts/pipelines/library/service-endpoints?view=vsts#sep-servbus). 
5.  Review [Overview of Azure Cloud Shell](https://docs.microsoft.com/en-us/azure/cloud-shell/overview).  **Azure Cloud Shell** is an interactive, browser accessible shell for managing Azure resources.  You will be using the Cloud Shell to create the Bastion Host (Linux VM) and logging into the VM via SSH.

**Important Notes:**
- This project assumes readers are familiar with Linux containers (`eg., docker, containerd`), Container Platforms (`eg., Kubernetes`), DevOps (`Continuous Integration/Continuous Deployment`) concepts and developing/deploying Microservices.  As such, this project is primarily targeted at technical/solution architects who have a good understanding of some or all of these solutions/technologies.  If you are new to Linux Containers/Kubernetes and/or would like to get familiar with container solutions available on Microsoft Azure, please go thru the hands-on labs that are part of the [MTC Container Bootcamp](https://github.com/Microsoft/MTC_ContainerCamp) first.
- AKS is a managed [Kubernetes](https://kubernetes.io/) service on Azure.  Please refer to the [AKS](https://azure.microsoft.com/en-us/services/container-service/) product web page for more details.
- This project has been tested on both an unmanaged (Standalone) Kubernetes cluster v1.9.x and on AKS v1.9.1+.  Kubernetes artifacts such as manifest files for application *Deployments* may not work **as-is** on **AKS v1.8.x**.  Some of these objects are `Beta` level objects in Kubernetes v1.8.x and therefore version info. for the corresponding API objects will have to be changed in the manifest files prior to deployment to AKS.
- Commands which are required to be issued on a Linux terminal window are prefixed with a `$` sign.  Lines that are prefixed with the `#` symbol are to be treated as comments.
- This project requires **all** resources to be deployed to the same Azure **Resource Group**.
- Make sure to specify either **eastus** or **centralus** as the *location* for the Azure *Resource Group* and the *AKS cluster*.  At the time of this writing, AKS is available in **Public Preview** in East US (eastus), Central US (centralus), Canada (canadaeast, canadacentral) and West Europe (westeurope) regions only.  

### A] Deploy a Linux CentOS VM on Azure (~ Bastion Host)
This Linux VM will be used for the following purposes
- Running a VSTS build agent (docker container) which will be used for running application and container builds.
- Installing Azure CLI 2.0 client.  This will allow us to administer and manage all Azure resources including the AKS cluster resources.
- Installing Git client.  We will be cloning this repository to make changes to the Kubernetes resources before deploying them to the AKS cluster.
- (TBD) Installing Jenkins.  If you would like to learn how to build and deploy this SpringBoot microservice to AKS using Jenkins CI/CD, then you will also need to install Java run-time and Jenkins.

Follow the steps below to create the Bastion host (Linux VM), install Azure CLI, login to your Azure account using the CLI and install Git client.

1.  Login to the [Azure Portal](https://portal.azure.com) using your credentials and use a **Azure Cloud Shell** session to perform the next steps.  Azure Cloud Shell is an interactive, browser-accessible shell for managing Azure resources.  The first time you access the Cloud Shell, you will be prompted to create a resource group, storage account and file share.  You can use the defaults or click on *Advanced Settings* to customize the defaults.  Accessing the Cloud Shell is described in [Overview of Azure Cloud Shell](https://docs.microsoft.com/en-us/azure/cloud-shell/overview). 

2.  An Azure resource group is a logical container into which Azure resources are deployed and managed.  From the Cloud Shell, use Azure CLI to create a **Resource Group**.  Azure CLI is already pre-installed and configured to use your Azure account (subscription) in the Cloud Shell.  Alternatively, you can also use Azure Portal to create this resource group.  
```
az group create --name myResourceGroup --location eastus
```
**NOTE:** Keep in mind, if you specify a different name for the resource group (other than **myResourceGroup**), you will need to substitute the same value in multiple CLI commands in the remainder of this project!  If you are new to AKS, it's best to use the suggested name.

3.  Use the command below to create a **CentOS 7.4** VM on Azure.  Make sure you specify the correct **resource group** name and provide a value for the *password*.  Once the command completes, it will print the VM connection info. in the JSON message (response).  Note down the public IP address, login name and password info. so that we can connect to this VM using SSH (secure shell).
Alternatively, if you prefer you can use SSH based authentication to connect to the Linux VM.  The steps for creating and using an SSH key pair for Linux VMs in Azure is documented [here](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/mac-create-ssh-keys).  You can then specify the location of the public key with the `--ssh-key-path` option to the `az vm create ...` command.
```
az vm create --resource-group myResourceGroup --name k8s-lab --image OpenLogic:CentOS:7.4:7.4.20180118 --size Standard_B2s --generate-ssh-keys --admin-username labuser --admin-password <password> --authentication-type password
```

4.  Login into the Linux VM via SSH. 
```
# In the Cloud Shell, SSH into the VM.  Substitute the public IP address for the Linux VM in the command below.
$ ssh labuser@x.x.x.x
#
```

5.  Install Azure CLI, Git client and Open JDK on this VM.
```
# Install Azure CLI on this VM so that we can to deploy this application to the AKS cluster later in step [D].
#
# Import the Microsoft repository key.
$ sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
#
# Create the local azure-cli repository information.
$ sudo sh -c 'echo -e "[azure-cli]\nname=Azure CLI\nbaseurl=https://packages.microsoft.com/yumrepos/azure-cli\nenabled=1\ngpgcheck=1\ngpgkey=https://packages.microsoft.com/keys/microsoft.asc" > /etc/yum.repos.d/azure-cli.repo'
#
# Install with the yum install command.
$ sudo yum install azure-cli
#
# Test the install
$ az -v
#
# Login to your Azure account
$ az login -u <user name> -p <password>
#
# View help on az commands, sub-commands
$ az --help
#
# Install Git client
$ sudo yum install git
#
# Check Git version number
$ git --version
#
# Install OpenJDK 8 on the VM.
$ sudo yum install -y java-1.8.0-openjdk-devel
#
# Check JDK version
$ java --version
```

6.  Next, install **docker-ce** container runtime. Refer to the commands below.  You can also refer to the [Docker CE install docs for CentOS](https://docs.docker.com/install/linux/docker-ce/centos/).
```
$ sudo yum update
$ sudo yum install -y yum-utils device-mapper-persistent-data lvm2
$ sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
$ sudo yum install docker-ce-18.03.0.ce
$ sudo systemctl enable docker
$ sudo groupadd docker
$ sudo usermod -aG docker labuser
```

LOGOUT AND RESTART YOUR VM BEFORE PROCEEDING.  You can restart the VM via Azure Portal.  Once the VM is back up, you can either use the Cloud Shell or a terminal window in your workstation to Login to the Linux VM via SSH.

```
$ sudo docker info
```

7.  Pull the Microsoft VSTS agent container from docker hub.  It will take a few minutes to download the image.
```
$ docker pull microsoft/vsts-agent
$ docker images
```

8.  Next, we will generate a VSTS personal access token (PAT) to connect our VSTS build agent to your VSTS account.  Login to VSTS using your account ID. In the upper right, click on your profile image and click **security**.  

![alt tag](./images/C-01.png)

Click on **Add** to create a new PAT.  In the next page, provide a short description for this token, select a expiry period and click **Create Token**.  See screenshot below.

![alt tag](./images/C-02.png)

In the next page, make sure to **copy and store** the PAT (token) into a file.  Keep in mind, you will not be able to retrieve this token again.  Incase you happen to lose or misplace the token, you will need to generate a new PAT and use it to reconfigure the VSTS build agent.  So save this PAT (token) to a file.

9.  Use the command below to start the VSTS build container.  Substitute the correct value for **VSTS_TOKEN** parameter, the value which you copied and saved in a file in the previous step.  The VSTS build agent will initialize and you should see a message indicating "Listening for Jobs".
```
docker run -e VSTS_ACCOUNT=ganrad -e VSTS_TOKEN=<xyz> -v /var/run/docker.sock:/var/run/docker.sock --name vstsagent -it microsoft/vsts-agent
```

### B] Deploy Azure Container Registry (ACR)
In this step, we will deploy an instance of Azure Container Registry to store container images which we will build in later steps.  A container registry such as ACR allows us to store multiple versions of application container images in one centralized repository and consume them from multiple nodes (VMs/Servers) where our applications are deployed.

1.  Login to your Azure portal account.  Then click on **Container registries** in the navigational panel on the left.  If you don't see this option in the nav. panel then click on **All services**, scroll down to the **COMPUTE** section and click on the star beside **Container registries**.  This will add the **Container registries** option to the service list in the navigational panel.  Now click on the **Container registries** option.  You will see a page as displayed below.

![alt tag](./images/B-01.png)

2.  Click on **Add** to create a new ACR instance.  Give a meaningful name to your registry, select an Azure subscription, select the **Resource group** which you created in step [A] and leave the location as-is.  The location should default to the location assigned to the resource group.  Select the **Basic** pricing tier.  Click **Create** when you are done.

![alt tag](./images/B-02.png)

3.  From the Linux terminal window connected to the Bastion host, create a **Service Principal** and grant relevant permissions (role) to this principal.  This is required to allow our AKS cluster to pull container images from this ACR registry.
```
# List your Azure subscriptions.  Note down the subscription ID.  We will need it in the next step.
$ az account list
#
# (Optional) Select and use the appropriate subscription ID (value of 'id' from previous step) 
$ az account set --subscription <SUBSCRIPTION_ID>
#
# Create the Azure service principal.  Substitute values for SUBSCRIPTION_ID, RG_GROUP, REGISTRY_NAME & SERVICE_PRINCIPAL_NAME.  Specify a meaningful name for the service principal.
$ az ad sp create-for-rbac --scopes /subscriptions/<SUBSCRIPTION_ID>/resourcegroups/<RG_NAME>/providers/Microsoft.ContainerRegistry/registries/<REGISTRY_NAME> --role Contributor --name <SERVICE_PRINCIPAL_NAME>
```
**NOTE:** From the `JSON` output of the previous command, copy and save the values for **appId** and **password**.  We will need these values in step [D] when we deploy this application to AKS.

### C] Create a new Build definition in VSTS to deploy the Springboot microservice
In this step, we will define the tasks for building the microservice (binary artifacts) application and packaging (layering) it within a docker container.  The build tasks use **Maven** to build the Springboot microservice & **docker-compose** to build the application container.  During the application container build process, the application binary is layered on top of a base docker image (CentOS 7).  Finally, the built application container is pushed into ACR which we deployed in step [B] above.

Before proceeding with the next steps, feel free to inspect the dockerfile and source files in the GitHub repository (under src/...).  This will give you a better understanding of how continuous integration (CI) can be easily implemented using VSTS.

1.  Fork this [GitHub repository](https://github.com/ganrad/k8s-springboot-data-rest) to **your** GitHub account.  In the browser window, click on **Fork** in the upper right hand corner to get a separate copy of this project added to your GitHub account.  You must be signed in to your GitHub account in order to fork this repository.

![alt tag](./images/A-01.png)

From the terminal window connected to the Bastion host, clone this repository.  Ensure that you are using the URL of your fork when cloning this repository.
```
# Switch to home directory
$ cd
# Clone your GitHub repository.  This will allow you to make changes to the application artifacts without affecting resources in the forked (original) GitHub project.
$ git clone https://github.com/<YOUR-GITHUB-ACCOUNT>/k8s-springboot-data-rest.git
#
# Switch to the 'k8s-springboot-data-rest' directory
$ cd k8s-springboot-data-rest
```

2.  If you haven't already done so, login to [VSTS](https://www.visualstudio.com/team-services/) using your Microsoft Live ID (or Azure AD ID) and create an *Account*.  Give the Account a meaningful name (eg., Your initials-AzureLab) and then create a new VSTS project. Give a name to your VSTS project.

![alt tag](./images/A-02.png)

**NOTE:** At this point, you may need to create an Azure **Service Principal** using the Azure CLI and then use the generated key and password to define a **Service Endpoint** in VSTS.  If you are not familiar with this process, please ask a lab procter to assist you!

3.  We will now create a **Build** definition and define tasks which will execute as part of the application build process.  Click on **Build and Release** in the top menu and then click on *Builds*.  Click on **New definition**

![alt tag](./images/A-03.png)

4.  In the **Select a source** page, select *GitHub* as the source repository. Give your connection a *name* and then select *Authorize using OAuth* link.  Optionally, you can use a GitHub *personal access token* instead of OAuth.  When prompted, sign in to your **GitHub account**.  Then select *Authorize* to grant access to your VSTS account.

5.  Once authorized, select the **GitHub Repo** which you forked in step [1] above.  Make sure you replace the account name in the **GitHub URL** with your account name.  Then hit continue.

![alt tag](./images/A-05.png)

6.  Search for text *Maven* in the **Select a template** field and then select *Maven* build.  Then click apply.

![alt tag](./images/A-06.png)

7.  Select *Default* in the **Agent Queue** field.  The VSTS build agent which you deployed in step [A] connects to this *queue* and listens for build requests.

![alt tag](./images/A-07.png)

8.  On the top extensions menu in VSTS, click on **Browse Markplace** and then search for text **replace tokens**.  In the results list below, click on **Colin's ALM Corner Build and Release Tools** (circled in yellow in the screenshot).  Then click on **Get it free** to install this extension in your VSTS account.

![alt tag](./images/A-08.PNG)

Next, search for text **Release Management Utility Tasks** extension provided by Microsoft DevLabs.  This extension includes the **Tokenizer utility** which we will be using in a continuous deployment (CD) step later on in this project.  Click on **Get it free** to install this extension in your VSTS account.  See screenshot below.

![alt tag](./images/A-81.PNG)

![alt tag](./images/A-82.PNG)

9.  Go back to your build definition and click on the plus symbol beside **Phase 1**.  Search by text **replace tokens** and then select the extension **Replace Tokens** which you just installed in the previous step.  Click **Add**.

![alt tag](./images/A-09.png)

10.  Click on the **Replace Tokens** task and drag it to the top of the task list.  In the **Source Path** field, select *src/main/resources* and specify `*.properties` in the **Target File Pattern** field.  In the **Token Regex** field, specify `__(\w+[.\w+]*)__` as shown in the screenshot below.  In the next step, we will use this task to specify the target kubernetes service name and namespace name.

![alt tag](./images/A-10.png)

11.  Click on the **Variables** tab and add a new variable to specify the Kubernetes service name and namespace name as shown in the screenshot below.

Variable Name | Value
------------- | ------
svc.name.k8s.namespace | mysql.development

![alt tag](./images/A-11.png)

12.  Switch back to the **Tasks** tab and click on the **Maven** task.  Specify values for fields **Goal(s)**, **Options** as shown in the screen shot below.  Ensure **Publish to TFS/Team Services** checkbox is enabled.

![alt tag](./images/A-12.png)

13.  Go thru the **Copy Files...** and **Publish Artifact:...** tasks.  These tasks copy the application binary artifacts (*.jar) to the **drop** location on the VSTS server.

14.  Next, we will package our application binary within a container.  Review the **docker-compose.yml** and **Dockerfile** files in the source repository to understand how the application container image is built.  Click on the plus symbol to add a new task. Search for task *Docker Compose* and click **Add**.

![alt tag](./images/A-14.png)

15.  Click on the *Docker Compose ...* task on the left panel.  Specify *Azure Container Registry* for **Container Registry Type**.  In the **Azure Subscription** field, select your Azure subscription.  Click on **Authorize**.  In the **Azure Container Registry** field, select the ACR which you created in step [B] above.  Check to make sure the **Docker Compose File** field is set to `**/docker-compose.yml`.  Enable **Qualify Image Names** checkbox.  In the **Action** field, select *Build service images* and specify *$(Build.BuildNumber)* for field **Additional Image Tags**.  Also enable **Include Latest Tag** checkbox.  See screenshot below.

![alt tag](./images/A-15.PNG)

16.  Once our application container image has been built, we will push it into the ACR.  Let's add another task to publish the container image built in the previous step to ACR.  Similar to step [15], search for task *Docker Compose* and click **Add**.

17.  Click on the *Docker Compose ...* task on the left.  Specify *Azure Container Registry* for **Container Registry Type**.  In the **Azure Subscription** field, select your Azure subscription.  In the **Azure Container Registry** field, select the ACR which you created in step [B] above.  Check to make sure the **Docker Compose File** field is set to `**/docker-compose.yml`.  Enable **Qualify Image Names** checkbox.  In the **Action** field, select *Push service images* and specify *$(Build.BuildNumber)* for field **Additional Image Tags**.  Also enable **Include Latest Tag** checkbox.  See screenshot below.

![alt tag](./images/A-17.PNG)

18.  Click **Save and Queue** to save the build definition and queue it for execution. Wait for the build process to finish.  When all build tasks complete OK and the build process finishes, you will see the screen below.

![alt tag](./images/A-18.png)

In the VSTS build agent terminal window, you will notice that a build request was received from VSTS and processed successfully. See below.

![alt tag](./images/A-19.png)

### D] Create an Azure Container Service (AKS) cluster and deploy Springboot microservice
In this step, we will first deploy an AKS cluster on Azure.  The Springboot **Purchase Order** microservice application reads/writes purchase order data from/to a relational (MySQL) database.  So we will deploy a **MySQL** database container (ephemeral) first and then deploy our Springboot Java application.  Kubernetes resources (object definitions) are usually specified in manifest files (yaml/json) and then submitted to the API Server.  The API server is responsible for instantiating corresponding objects and bringing the state of the system to the desired state.

Kubernetes manifest files for deploying the **MySQL** and **po-service** (Springboot application) containers are provided in the **k8s-scripts/** folder in the GitHub repository.  There are two manifest files in this folder **mysql-deploy.yaml** and **app-deploy.yaml**.  As the names suggest, the *mysql-deploy* manifest file is used to deploy the **MySQL** database container and the other file is used to deploy the **Springboot** microservice respectively.

Before proceeding with the next steps, feel free to inspect the Kubernetes manifest files to get a better understanding of the following.  These are all out-of-box capabilities provided by Kubernetes.
-  How confidential data such as database user names & passwords are injected (at runtime) into the application container using **Secrets**
-  How application configuration information such as database connection URL and the database name parameters are injected (at runtime) into the application container using **ConfigMaps**
-  How **environment variables** such as the MySQL listening port is injected (at runtime) into the application container.
-  How services in Kubernetes can auto discover themselves using the built-in **Kube-DNS** proxy.

In case you want to modify the default values used for MySQL database name and/or database connection properties (user name, password ...), refer to [Appendix A](#appendix-a) for details.  You will need to update the Kubernetes manifest files.

Follow the steps below to provision the AKS cluster and deploy the *po-service* microservice.
1.  Ensure the *Resource provider* for AKS service is enabled (registered) for your subscription.  A quick and easy way to verify this is, use the Azure portal and go to *->Azure Portal->Subscriptions->Your Subscription->Resource providers->Microsoft.ContainerService->(Ensure registered)*.  Alternatively, you can use Azure CLI to register all required service providers.  See below.
```
az provider register -n Microsoft.Network
az provider register -n Microsoft.Storage
az provider register -n Microsoft.Compute
az provider register -n Microsoft.ContainerService
```

2.  Switch back to the Bastion host (Linux VM) terminal window where you have Azure CLI installed and make sure you are logged into to your Azure account.  We will install **kubectl** which is a command line tool for administering and managing a Kubernetes cluster.  Refer to the commands below in order to install *kubectl*.
```
# Switch to your home directory
$ cd
#
# Create a new directory 'aztools' under home directory to store the kubectl binary
$ mkdir aztools
#
# Install kubectl binary in the new directory
$ az aks install-cli --install-location=./aztools/kubectl
#
# Add the location of 'kubectl' binary to your search path and export it.
# Alternatively, add the export command below to your '.bashrc' file in your home directory. Then logout of your VM (Bastion Host) from the terminal window and log back in for changes to take effect.  By including this command in your '.bashrc' file, you don't have to set the location of the 'kubectl' binary in the PATH environment variable and export it every time you logout and log in to this VM.
$ export PATH=$PATH:/home/labuser/aztools
#
# Check if kubectl is installed OK
$ kubectl version -o yaml
```
**NOTE:** At this point, you can use a) The Azure Portal Web UI to create an AKS cluster and b) The Kubernetes Dashboard UI to deploy the Springboot Microservice application artifacts.  To use a browser (*Web UI*) for deploying the cluster and application artifacts, refer to the steps in [extensions/k8s-dash-deploy](./extensions/k8s-dash-deploy).  Alternatively, if you prefer Azure CLI for deploying and managing resources on Azure, proceed with the next steps.

3.  Refer to the commands below to create an AKS cluster.  If you haven't already created a **resource group**, you will need to create one first.  If needed, go back to step [A] and review the steps for the same.  Cluster creation will take a few minutes to complete.
```
# Create a 1 Node AKS cluster
$ az aks create --resource-group myResourceGroup --name akscluster --node-count 1 --dns-name-prefix akslab --generate-ssh-keys
#
# Verify state of AKS cluster
$ az aks show -g myResourceGroup -n akscluster --output table
```

4.  Connect to the AKS cluster.
```
# Configure kubectl to connect to the AKS cluster
$ az aks get-credentials --resource-group myResourceGroup --name akscluster
#
# Check cluster nodes
$ kubectl get nodes -o wide
#
# Check default namespaces in the cluster
$ kubectl get namespaces
```

5.  Next, create a new Kubernetes **namespace** resource.  This namespace will be called *development*.  
```
# Make sure you are in the *k8s-springboot-data-rest* directory.
$ kubectl create -f k8s-scripts/dev-namespace.json 
#
# List the namespaces
$ kubectl get namespaces
```

6.  Create a new Kubernetes context and associate it with the **development** namespace.  We will be deploying all our application artifacts into this namespace in subsequent steps.
```
# Create the 'dev' context
$ kubectl config set-context dev --cluster=akscluster --user=clusterUser_myResourceGroup_akscluster --namespace=development
#
# Switch the current context to 'dev'
$ kubectl config use-context dev
#
# Check your current context (should list 'dev' in the output)
$ kubectl config current-context
```

7.  Configure Kubernetes to use the ACR (configured in step [B]) to pull our application container images and deploy containers.
When creating deployments, replica sets or pods, AKS (Kubernetes) will try to use docker images already stored locally (on nodes) or pull them from the public docker hub.  To change this, we need to specify the ACR as part of Kubernetes object configuration (yaml or json).  Instead of specifying this directly in the configuration, we will use Kubernetes **Secrets**.  By using secrets, we tell the Kubernetes runtime to use the info. contained in the secret to authenticate against ACR and push/pull images.  In the Kubernetes object (pod definition), we reference the secret by it's name only.

kubectl parameter | Value to substitute
----------------- | -------------------
SERVICE_PRINCIPAL_ID | 'appId' value from step [B]
YOUR_PASSWORD | 'password' value from step [B]

```
# Create a secret containing credentials to authenticate against ACR.  Substitute values for REGISTRY_NAME, YOUR_MAIL, SERVICE_PRINCIPAL ID and YOUR_PASSWORD.
$ kubectl create secret docker-registry acr-registry --docker-server <REGISTRY_NAME>.azurecr.io --docker-email <YOUR_MAIL> --docker-username=<SERVICE_PRINCIPAL_ID> --docker-password <YOUR_PASSWORD>
#
# List the secrets
$ kubectl get secrets
```

8.  Update the **k8s-scripts/app-deploy.yaml** file.  The *image* attribute should point to your ACR.  This will ensure AKS pulls the application container image from the correct registry. Substitute the correct value for the *ACR registry name* in the *image* attribute (highlighted in yellow) in the pod spec as shown in the screenshot below.

![alt tag](./images/D-01.PNG)

9.  Deploy the **MySQL** database container.
```
# Make sure you are in the *k8s-springboot-data-rest* directory.
$ kubectl create -f k8s-scripts/mysql-deploy.yaml
#
# List pods.  You can specify the '-w' switch to watch the status of pod change.
$ kubectl get pods
```
The status of the mysql pod should change to *Running*.  See screenshot below.

![alt tag](./images/D-02.png)

(Optional) You can login to the mysql container using the command below. Specify the correct value for the pod ID (Value under 'Name' column listed in the previous command output).  The password for the 'mysql' user is 'password'.
```
$ kubectl exec <pod ID> -i -t -- mysql -u mysql -p sampledb
```

10.  Deploy the **po-service** microservice container.
```
# Make sure you are in the *k8s-springboot-data-rest* directory.
$ kubectl create -f k8s-scripts/app-deploy.yaml
#
# List pods.  You can specify the '-w' switch to watch the status of pod change.
$ kubectl get pods
```
The status of the mysql pod should change to *Running*.  See screenshot below.

![alt tag](./images/D-03.png)

11.  (Optional) As part of deploying the *po-service* Kubernetes service object, an Azure cloud load balancer gets auto-provisioned and configured. The load balancer accepts HTTP requests for our microservice and re-directes all calls to the service endpoint (port 8080).  Take a look at the Azure load balancer.

![alt tag](./images/D-04.png)

### Accessing the Purchase Order Microservice REST API 

As soon as the **po-service** application is deployed in AKS, 2 purchase orders will be inserted into the backend (MySQL) database.  The inserted purchase orders have ID's 1 and 2.  The application's REST API supports all CRUD operations (list, search, create, update and delete) on purchase orders.

In a Kubernetes cluster, applications deployed within pods communicate with each other via services.  A service is responsible for forwarding all incoming requests to the backend application pods.  A service can also expose an *External IP Address* so that applications thatare external to the AKS cluster can access services deployed within the cluster.

Use the command below to determine the *External* (public) IP address (Azure load balancer IP) assigned to the service end-point.
```
# List the kubernetes service objects
$ kubectl get svc
```
The above command will list the **IP Address** (both internal and external) for all services deployed within the *development* namespace as shown below.  Note how the **mysql** service doesn't have an *External IP* assisgned to it.  Reason for that is, we don't want the *MySQL* service to be accessible from outside the AKS cluster.

![alt tag](./images/E-01.png)

The REST API exposed by this microservice can be accessed by using the _context-path_ (or Base Path) `orders/`.  The REST API endpoint's exposed are as follows.

URI Template | HTTP VERB | DESCRIPTION
------------ | --------- | -----------
orders/ | GET | To list all available purchase orders in the backend database.
orders/{id} | GET | To get order details by `order id`.
orders/search/getByItem?{item=value} | GET | To search for a specific order by item name
orders/ | POST | To create a new purchase order.  The API consumes and produces orders in `JSON` format.
orders/{id} | PUT | To update a new purchase order. The API consumes and produces orders in `JSON` format.
orders/{id} | DELETE | To delete a purchase order.

You can access the Purchase Order REST API from your Web browser, e.g.:

- http://<Azure_load_balancer_ip>/orders
- http://<Azure_load_balancer_ip>/orders/1

Use the sample scripts in the **./scripts** folder to test this microservice.

Congrats!  You have just built and deployed a Java Springboot microservice on Azure Kubernetes Service!!

If you would like to learn how to implement **Continuous Deployment** in VSTS, continue with the next steps.  We will define a **Release Pipeline** in VSTS to perform automated application deployments next.

### E] Create a simple *Release Pipeline* in VSTS
1.  Using a web browser, login to your VSTS account (if you haven't already) and select your project which you created in Step [C]. Click on *Build and Release* menu on the top panel and select *Releases*.  Next, click on the *+* icon on the *Releases* tab and select *Create release definition*.

![alt tag](./images/E-02.PNG)

In the *Select a Template* page, click on *Empty process*.  See screenshot below.

![alt tag](./images/E-03.PNG)

In the *Environment* page, specify *Staging-A* as the name for the environment.  Then click on *+Add* besides *Artifacts* (under *Pipeline* tab).

![alt tag](./images/E-04.PNG)

In the *Add artifact* page, select *Build* for **Source type**, select your VSTS project from the **Project** drop down menu and select your *Build definition* in the drop down menu for **Source (Build definition)**.  Leave the remaining field values as is and click **Add**.  See screenshot below. 

![alt tag](./images/E-05.PNG)

In the *Pipeline* tab, click on the *trigger* icon (highlighted in yellow) and enable **Continuous deployment trigger**.  See screenshot below.

![alt tag](./images/E-06.PNG)

Next, click on *1 phase, 0 task* in the **Environments** box under environment *Staging-A*.  Click on *Agent phase* under the *Tasks* tab and make sure **Agent queue** value is set to *Hosted VS2017*.  Leave the remaining field values as is.  See screenshot below.

![alt tag](./images/E-07.PNG)

Recall that we had installed a **Tokenizer utility** extension in VSTS in Step [C].  We will now use this extension to update the container image *Tag value* in Kubernetes deployment manifest file *./k8s-scripts/app-update-deploy.yaml*.  Open/View the deployment manifest file in an editor (vi) and search for variable **__Build.BuildNumber__**.  When we re-run (execute) the *Build* pipeline, it will generate a new tag (Build number) for the *po-service* container image.  The *Tokenizer* extension will then substitute the latest tag value in the substitution variable.

Click on the ** + ** symbol beside **Agent phase** and search for text **Tokenize with** in the *Search* text box (besides **Add tasks**). Click on **Add**.  See screenshot below.

![alt tag](./images/E-08.PNG)

Click on the **Tokenizer** task and click on the ellipsis (...) below field **Source filename**.  In the **Select File Or Folder** window, select the deployment manifest file from the respective folder as shown in the screenshots below. Click **OK**.

![alt tag](./images/E-09.PNG)

![alt tag](./images/E-10.PNG)

Again, click on the ** + ** symbol beside **Agent phase** and search for text **Deploy to Kubernetes**, select this extension and click **Add*.  See screenshot below.

![alt tag](./images/E-11.PNG)

Click on the **Deploy to Kubernetes** task on the left panel and fill out the details (numbered) as shown in the screenshot below.  This task will **apply** (update) the changes (image tag) to the kubernetes **Deployment** object on the Azure AKS cluster and do a **Rolling** deployment for the **po-service** microservice application.

![alt tag](./images/E-12.PNG)

Expand the **Secrets** field panel and fill in the values as shown in the screenshot below.  Choose the correct value for **Azure subscription**.  See screenshot below.

![alt tag](./images/E-13.PNG)

Change the name of the release pipeline to **cd-po-service** and click **Save** on the top panel.  Provide a comment and click **OK**.

![alt tag](./images/E-14.PNG)

We have now finished defining the **Release pipeline**.  This pipeline will in turn be triggered whenever the build pipeline completes Ok.

2.  Edit the build pipeline and click on the **Triggers** tab.  See screenshot below.

![alt tag](./images/E-15.PNG)

Click the checkbox for both **Enable continuous integration** and **Batch changes while a build is in progress**.  Leave other fields as is.  Click on **Save & queue** menu and select the **Save** option.

![alt tag](./images/E-16.PNG)

3.  Modify the microservice code to calculate **Discount amount** and **Order total** for purchase orders.  These values will be returned in the JSON response for the **GET** API (operation).  

Open a web browser tab and navigate to this project (your Fork) on GitHub.  Go to the **model** sub directory within **src** directory and click on **PurchaseOrder.java** file.  See screenshot below.

![alt tag](./images/E-17.PNG)

Click on the pencil (Edit) icon on the top right of the code view panel (see below) to edit this file.

![alt tag](./images/E-18.PNG)

Uncomment lines 100 thru 108 (highlighted in yellow).

![alt tag](./images/E-19.PNG)

Provide a comment and commit (save) the file.  The git commit will trigger a new build (**Continuous Integration**) for the **po-service** microservice in VSTS.  Upon successful completion of the build process, the updated container images will be pushed into the ACR and the release pipeline (**Continuous Deployment**) will be executed.   As part of the CD process, the Kubernetes deployment object for the **po-service** microservice will be updated with the newly built container image.  This action will trigger a **Rolling** deployment of **po-service** microservice in AKS.  As a result, the **po-service** containers (*Pods*) from the old deployment (version 1.0) will be deleted and a new deployment (version 2.0) will be instantiated in AKS.  The new deployment will use the latest container image from the ACR and spin up new containers (*Pods*).  During this deployment process, users of the **po-service** microservice will not experience any downtime as AKS will do a rolling deployment of containers.

4.  Switch to a browser window and test the **po-Service** REST API.  Verify that the **po-service** API is returning two additional fields (*discountAmount* and *orderTotal*) in the JSON response.

Congrats!  You have successfully used DevOps to automate the build and deployment of a containerized microservice application on Kubernetes.  

In this project, we experienced how DevOps, Microservices and Containers can be used to build next generation applications.  These three technologies are changing the way we develop and deploy software applications and are at the forefront of fueling digital transformation in enterprises today!

### Appendix A
In case you want to change the name of the *MySQL* database name, root password, password or username, you will need to make the following changes.  See below.

- Update the *Secret* object **mysql** in file *./k8s-scripts/mysql-deploy.yaml* file with appropriate values (replace 'xxxx' with actual values) by issuing the commands below.
```
# Create Base64 encoded values for the MySQL server user name, password, root password and database name. Repeat this command to generate values for each property you want to change.
$ echo "xxxx" | base64 -w 0
# Then update the corresponding parameter value in the Secret object.
```

- Update the *./k8s-scripts/app-deploy.yaml* file.  Specify the correct value for the database name in the *ConfigMap* object **mysql-db-name** parameter **mysql.dbname** 

- Update the *Secret* object **mysql-sql** in file *./k8s-scripts/app-deploy.yaml* file with appropriate values (replace 'xxxx' with actual values) by issuing the commands below.
```
# Create Base64 encoded values for the MySQL server user name and password.
$ echo "mysql.user=xxxx" | base64 -w 0
$ echo "mysql.password=xxxx" | base64 -w 0
# Then update the *db.username* and *db.password* parameters in the Secret object accordingly.
```

### Troubleshooting
- In case you created the **po-service** application artifacts in the wrong Kubernetes namespace (other than `development`), use the commands below to clean all API objects from the current namespace.  Then follow instructions in Section D starting Step 6 to create the API objects in the 'development' namespace.
```
#
# Delete replication controllers - mysql, po-service
$ kubectl delete rc mysql
$ kubectl delete rc po-service
#
# Delete service - mysql, po-service
$ kubectl delete svc mysql
$ kubectl delete svc po-service
#
# Delete secrets - acr-registry, mysql, mysql-secret
$ kubectl delete secret acr-registry
$ kubectl delete secret mysql
$ kubectl delete secret mysql-secret
#
# Delete configmap - mysql-db-name
$ kubectl delete configmap mysql-db-name
```

- In case you want to delete all API objects in the 'development' namespace and start over again, delete the 'development' namespace.  Also, delete the 'dev' context.  Then start from Section D Step 5 to create the 'development' namespace, create the API objects and deploy the microservices.
```
# Make sure you are in the 'dev' context
$ kubectl config current-context
#
# Switch to the 'akscluster' context
$ kubectl config use-context akscluster
#
# Delete the 'dev' context
$ kubectl config delete-context dev
#
# Delete the 'development' namespace
$ kubectl delete namespace development
```

- A few useful Kubernetes commands.
```
# List all user contexts
$ kubectl config view
#
# Switch to a given 'dev' context
$ kubectl config use-context dev
#
# View compute resources (memory, cpu) consumed by pods in current namespace.
$ kubectl top pods
#
# List all pods
$ kubectl get pods
#
# View all details for a pod - Start time, current status, volume mounts etc
$ kubectl describe pod <Pod ID>
```
