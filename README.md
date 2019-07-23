# IoT Edge lab #1

# Part 1: Deploy code to a Linux device

## Use Azure Cloud Shell

Azure hosts Azure Cloud Shell, an interactive shell environment that you can use through your browser.  You can use the Cloud Shell pre-installed commands to run the code in this lab without having to install anything on your local environment.

**Go to https://shell.azure.com (please use bash version of the shell, not PowerShell)**

Add the Azure IoT extension to the cloud shell instance.

    az extension add --name azure-cli-iot-ext

## Prerequisites

### Cloud resources

A resource group to manage all the resources you use in this lab.

    az group create --name IoTEdgeResources --location westus2

### IoT Edge device

A Linux virtual machine acts as your IoT Edge device. You should use the Microsoft-provided Azure IoT Edge on Ubuntu virtual machine, which preinstalls everything you need to run IoT Edge on a device. Accept the terms of use and create this virtual machine using the following commands:

    az vm image accept-terms --urn microsoft_iot_edge:iot_edge_vm_ubuntu:ubuntu_1604_edgeruntimeonly:latest
    
    az vm create --resource-group IoTEdgeResources --name EdgeVM --image microsoft_iot_edge:iot_edge_vm_ubuntu:ubuntu_1604_edgeruntimeonly:latest --admin-username azureuser --generate-ssh-keys

Be sure to note the VM's public IP address.

## Create an IoT Hub

The free level of IoT Hub works for this lab. If you've used IoT Hub in the past and already have a free hub created, you can use that IoT hub. Each subscription can only have one free IoT hub.

The following code creates a free **F1** hub in the resource group **IoTEdgeResources**. Replace *{hub_name}* with a unique name for your IoT hub.

    az iot hub create --resource-group IoTEdgeResources --name {hub_name} --sku F1

If you get an error because there's already one free hub in your subscription, change the SKU to **S1**. If you get an error that the IoT Hub name isn't available, it means that someone else already has a hub with that name. Try a new name.

## Add an IoT Edge device in IoT Hub

Create a device identity for your IoT Edge device so that it can communicate with your IoT hub. The device identity lives in the cloud, and you use a unique device connection string to associate a physical device to a device identity.

Since IoT Edge devices behave and can be managed differently than typical IoT devices, declare this identity to be for an IoT Edge device with the `--edge-enabled` flag.

1. In the Azure cloud shell, enter the following command to create a device named named **myEdgeDevice1** in your hub. Replace `{hub_name}` with name of your choice. 

        az iot hub device-identity create --hub-name {hub_name} --device-id myEdgeDevice1 --edge-enabled

2. Add a `gbb` tag to the device using the following command to update its twin:

        az iot hub device-twin update --device-id myEdgeDevice1 --hub-name {hub_name} --set tags='{"env":"gbb"}'

3. Retrieve the connection string for your device, which is used to link your physical device with its identity in IoT Hub.

        az iot hub device-identity show-connection-string --device-id myEdgeDevice1 --hub-name {hub_name}

Copy the values of the `connectionString` key for the device from the JSON output and save it. This value is the device connection string. You'll use this connection string to configure the IoT Edge runtime in the next section.

## Configure your IoT Edge device

The IoT Edge runtime is deployed on all IoT Edge devices. It has three components. The **IoT Edge security daemon** starts each time an IoT Edge device boots and bootstraps the device by starting the IoT Edge agent. The **IoT Edge agent** facilitates deployment and monitoring of modules on the IoT Edge device, including the IoT Edge hub. The **IoT Edge hub** manages communications between modules on the IoT Edge device, and between the device and IoT Hub. 

During the runtime configuration, you provide a device connection string. **Use the connection string that you retrieved from the Azure CLI**. This string associates your physical device with the IoT Edge device identity in Azure.

### **Set the connection string on the IoT Edge device**

Since you're using the Azure IoT Edge on Ubuntu virtual machine as described in the prerequisites, your device already has the IoT Edge runtime installed. You just need to configure your device with the device connection string that you retrieved in the previous section. You can do this remotely without having to connect to the virtual machine. Run the following command, replacing **{device_connection_string}** with your own string.

    az vm run-command invoke -g IoTEdgeResources -n EdgeVM --command-id RunShellScript --script "/etc/iotedge/configedge.sh '{device_connection_string}'"

### View the IoT Edge runtime status

Use the following command to connect to your virtual machine. Replace **{azureuser}** if you used a different username than the one suggested in the prerequisites. Replace **{publicIpAddress}** with your machine's address.

    ssh azureuser@{publicIpAddress}

Verify that the runtime was successfully installed and configured on your IoT Edge device:

1. Check to see that the IoT Edge security daemon is running as a system service.

        sudo systemctl status iotedge

    ![](https://docs.microsoft.com/en-us/azure/iot-edge/media/quickstart-linux/iotedged-running.png)

2. If you need to troubleshoot the service, retrieve the service logs.

    
        journalctl -u iotedge
    

3. View the modules running on your device.

        sudo iotedge list
   

    ![](https://docs.microsoft.com/en-us/azure/iot-edge/media/quickstart-linux/iotedge-list-1.png)

Your IoT Edge device is now configured. It's ready to run cloud-deployed modules!

## Deploy an IoT Edge module using an at-scale deployment

Next, we'll deploy a sample pre-built simulated temperature sensor module to our edge device using an at-scale deployment. 

The module that you deploy in this section simulates a sensor and sends generated data. This module is a useful piece of code when you're getting started with IoT Edge because you can use the simulated data for development and testing. If you want to see exactly what this module does, you can view the [simulated temperature sensor source code](https://github.com/Azure/iotedge/blob/027a509549a248647ed41ca7fe1dc508771c8123/edge-modules/SimulatedTemperatureSensor/src/Program.cs).

To deploy your first module from the Azure Marketplace, use the following steps:

1. Go to the [module page in the marketplace](https://azuremarketplace.microsoft.com/en/marketplace/apps/microsoft.edge-simulated-temperature-sensor-ga?tab=Overview) and click **Get it Now** and **Continue.**
2. On the next page, select right **Subscription** and **IoT Hub.**
3. Select the **Deploy at Scale** toggle along with **Add module to new deployment** option and click **Create.**
4. In the **Create Deployment** **wizard, follow instructions below clicking **Next** or **Previous** buttons to move between steps.
    - Step 1: Provide a deployment name.
    - Step 2: Confirm **SimulatedTemperatureSensor** is already be added.
    - Step 3: Replace the JSON in the text field with the following:

            {
              "routes": {
                "upstream": "FROM /messages/* INTO $upstream"
              }
            }

    - Step 4: Leave unchanged
    - Step 5: Use values **10** for Priority and `tags.env='gbb'` for **Target Condition**. Confirm your edge device is targeted by clicking the **View Devices** button.
    - Step 6: Review the final deployment and click **Submit**.

    ### View generated data

    SSH into your IoT Edge VM again, if needed. Confirm that the module deployed from the cloud is running on your IoT Edge device:

        sudo iotedge list

    ![](https://docs.microsoft.com/en-us/azure/iot-edge/media/quickstart-linux/iotedge-list-2.png)

    View the messages being sent from the temperature sensor module:

        sudo iotedge logs SimulatedTemperatureSensor -f

    ![](https://docs.microsoft.com/en-us/azure/iot-edge/media/quickstart-linux/iotedge-logs.png)

    ### Verify cloud ingestion and control

    To can confirm these messages are being ingested in to Azure via IoT Hub, **exit the SSH session to the VM** and enter the following command in your cloud shell:

        az iot hub monitor-events -n {iothub_name} -d myEdgeDevice1

    You use IoT Hub primitives for cloud-to-device communication as well. For example, to send a synchronous command to the device you can invoke a method a module has pre-registered. The simulated sensor module registers a `reset` method which resets the synthetic sensor readings back to the early 20s (each subsequent device-to-cloud message gradually increases this). Call the method using this command.

        az iot hub invoke-module-method -n {iothub_name} -d myEdgeDevice1 -m SimulatedTemperatureSensor --mn reset

# Part 2: Deploy code to Kubernetes

IoT Edge can also deploy workloads to Kubernetes, while retaining cloud manageability and IoT Edge's application model. Here is the high-level architecture diagram, go to [https://aka.ms/iotedge-on-kubernetes](https://aka.ms/iotedge-on-kubernetes) for more details:

![](https://docs.microsoft.com/en-us/azure/iot-edge/media/how-to-install-iot-edge-kubernetes/k8s-arch.png)

Retaining the same workload app model allows for the same edge deployment to target either a single node device or an edge "device" that lives in a Kubernetes cluster. 

In this lab we'll stand up a Kubernetes environment, configure it to host an edge device and apply the same at scale edge deployment to it.

## Bootstrap a Kubernetes cluster environment

We'll use the [k3d](https://github.com/rancher/k3d) project that provides a lightweight single node Kubernetes environment to test this support. 

1. In the Azure cloud shell, enter the following command to create a second device named named **myEdgeDevice2**  in your hub.

        az iot hub device-identity create --hub-name {hub_name} --device-id myEdgeDevice2 --edge-enabled

2. Add the same `gbb` tag to the device using the following command to update its twin:

        az iot hub device-twin update --device-id myEdgeDevice2 --hub-name {hub_name} --set tags='{"env":"gbb"}'

3. Retrieve the connection string for your device, and make note of it.

        az iot hub device-identity show-connection-string --device-id myEdgeDevice2 --hub-name {hub_name}

4. SSH into the Ubuntu VM created in Part 1.

        ssh azureuser@{publicIpAddress}

5. Install k3d.

        sudo curl -s https://raw.githubusercontent.com/rancher/k3d/master/install.sh | bash

6. Create a single node Kubernetes cluster hosted in a docker container.

        sudo k3d create -n test0 -w 2 --image rancher/k3s:v0.7.0-rc2

7. Install kubectl, a command line to tool to interact with a Kubernetes cluster.

        sudo snap install kubectl --classic

8. Associate kubectl with the Kubernetes cluster, and view cluster info.

        export KUBECONFIG="$(sudo k3d get-kubeconfig --name='test0')" && \
        kubectl cluster-info && \
        kubectl get nodes

9. Prepare the cluster for installation of Helm, a package manager for Kubernetes by creating a service account for Tiller (Helm's in-cluster component):

```
echo "
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
" | kubectl apply -f -
```

10. Download and initialize Helm, this can take a minute or two.

        sudo snap install helm --classic && \
        helm init --service-account tiller

11. Add a repo that contains the IoT Edge Helm chart.

        helm repo add edgek8s https://edgek8s.blob.core.windows.net/helm/ && \
        helm repo update

12. Install the edge device into the cluster using the device connection string retrieved in Step 3 of this section.

        # Set kubecontext
        export KUBECONFIG="$(sudo k3d get-kubeconfig --name='test0')" 
        
        # Install IoT Edge chart
        helm install --name k8s-edge1 --set "deviceConnectionString=<replace-with-device-connection-string>" edgek8s/edge-kubernetes

13.  View the Kubernetes namespaces on the cluster. You should see a namespace create for the edge device of the form *msiot-**

        ```
        kubectl get ns
        ```
        
14. Watch the running pods in the namespace, and you should see all modules defined in your edge deployment running.

        watch kubectl get pods -n <msiot-xxx-xxx>

Since this device is also part of the target condition of the at-scale deployment, the same configuration was applied on it. You can verify cloud ingestion and control of the device in the cluster just like any other IoT device.

# Additional resources

There are many other features to explore in IoT Edge -

- Docs landing page:[https://docs.microsoft.com/en-us/azure/iot-edge/](https://docs.microsoft.com/en-us/azure/iot-edge/)
- Github repo: [https://github.com/azure/iotedge](https://github.com/azure/iotedge)
- Using IoT Edge as a gateway in Kubernetes: [https://github.com/Azure-Samples/iotedge-gateway-on-kubernetes/blob/master/README.md](https://github.com/Azure-Samples/iotedge-gateway-on-kubernetes/blob/master/README.md)
