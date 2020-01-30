# Build and deploy Azure Functions for ingesting Claims records at scale
This sub-project describes the steps for ingesting Claims records into the backend Azure SQL Server Database using **Azure Functions** serverless applications.

The Functions application modules are implemented in .NET Core and Nodejs respectively. The Azure Functions runtime itself is packaged within a Linux container and deployed on AKS.

A brief description of the Functions applications is provided below.
- [ClaimsApiSyncFunc](./ClaimsApiAsyncFunc)

  This .NET Core application comprises of two Functions (modules) written in C#.
  - **GenHttpApiFunctions**:

    This Function exposes a generic REST API for processing new medical Claims.  Upon receiving medical Claims transactions, this function stores them in a persistent Azure Service Bus Queue. This step ensures the received Claims transactions are processed in a guaranteed and reliable manner.
  - **AsyncSbQueueApiFunc**:

    This Function reads medical Claims transactions from an Azure Service Bus Queue and then invokes the Claims REST API end-points implemented in the parent project. In the event the backend application is down or unreachable, the Claims transactions persist in the service bus queue.

- [ClaimsAsyncApiFunc](./ClaimsAsyncApiFunc)

  This Nodejs application comprises of one Function (module) written in Javascript.
  - **GetSbQCallWebApi**:

    This Function reads the medical Claims transactions from the Azure Service Bus Queue and then invokes the Claims REST API end-point implemented in the parent project.  This step executes an atomic transaction and ensures the Claims records are successfully delivered to the backend application.  In the event the backend application is down or unreachable, the Claims transactions remain in the service bus queue.

The solution leverages *Kubernetes Event Driven Auto-scaling* (KEDA) on AKS.  Kubernetes based Functions provides the Functions runtime in a Docker container with event-driven scaling through KEDA.

**Prerequisites:**
1. Readers are required to complete Sections A thru G in the [parent project](https://github.com/ganrad/aks-aspnet-sqldb-rest) before proceeding with the hands-on labs in this sub-project.

Readers are advised to go thru the following on-line resources before proceeding with the hands-on sections.
- [Azure Functions Documentation](https://docs.microsoft.com/en-us/azure/azure-functions/)
- [Kubernetes based event driven auto-scaling - KEDA](https://keda.sh/)
- [KEDA on AKS](https://docs.microsoft.com/en-us/azure/azure-functions/functions-kubernetes-keda)

## A. Deploy an Azure Service Bus Namespace and Queues
**Approx. time to complete this section: 15 minutes**

1. Login to the Azure Portal.

   Login to the [Azure Portal](https://portal.azure.com) using your credentials.

2. Provision an Azure Service Bus Namespace and Queues.

   Refer to the tutorial below to provision an Azure Service Bus, Namespace and Queues.  Create two Queues named **claims-req-queue** and **claims-del-queue**.

   - [Create a Service Bus queue - Azure Portal](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-quickstart-portal)
   - [Create a Service Bus queue - Azure CLI](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-quickstart-cli)

## B. Install pre-requisite tools on the Linux VM (Bastion Host)
**Approx. time to complete this section: 15 minutes**

1. Login into the Linux VM via SSH.
  
   ```bash
   # ssh into the VM. Substitute the public IP address for the Linux VM in the command below.
   $ ssh labuser@x.x.x.x
   #
   ```

2. Install Nodejs on the Linux VM.

   ```bash
   # Switch to home directory
   $ cd
   #
   # Create a directory to save 'nodejs' runtime
   $ mkdir nodejs
   #
   # Download and unzip the binary into $HOME/nodejs
   $ curl -sL "https://nodejs.org/dist/v12.14.1/node-v12.14.1-linux-x64.tar.xz" | unxz | tar -xv --directory=$HOME/nodejs
   #
   # Edit '.bashrc' file in your home ($HOME) directory and set the PATH variable to include the 
   # path of the 'nodejs' bin directory => '$HOME/nodejs/node-vx.x.x-linux-x64/bin'
   #
   # Check 'nodejs' version
   $ node --version
   #
   ```

3. Install Azure Function Core Tools v3.x.

   **NOTE:** If you prefer, you can also install Azure Function Core Tools v2.x runtime.

   ```bash
   # Switch to home directory
   $ cd
   #
   # Download Azure Function Core Tools 2.x runtime
   # wget https://github.com/Azure/azure-functions-core-tools/releases/download/2.7.2045/Azure.Functions.Cli.linux-x64.2.7.2045.zip
   #
   # Download Azure Function Core Tools 3.x runtime
   $ wget https://github.com/Azure/azure-functions-core-tools/releases/tag/3.0.2009
   #
   # Create a directory to save 'Azure Function Tools' binaries
   $ mkdir az-func-core-tools
   #
   # Unzip the Azure Function Core Tools binaries into the 'az-func-core-tools' directory
   $ unzip -d ~/az-func-core-tools/ Azure.Functions.Cli.linux-x64.3.0.2009.zip
   #
   # Edit '.bashrc' file in your home ($HOME) directory and set the PATH variable to include the 
   # path of the 'Azure Function Core Tools' directory. See below.
   # => AZ_FUNC_CORE_TOOLS=$HOME/az-func-core-tools
   # => export PATH=$AZ_FUNC_CORE_TOOLS:$PATH
   #
   # Check Azure Function Core Tools version
   $ func --version
   #
   ```