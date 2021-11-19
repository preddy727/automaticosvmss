# Azure Virtual Machine Scale set upgrade options 
## Manual 
```powershell 

1.	Login to https://resources.azure.com/

2.	Select the subscription -> resourceGroups -> providers -> Microsoft Compute -> virtualMachineScaleSets

https://resources.azure.com/subscriptions/<subid>/resourceGroups/31230-eastus2-nprd-devops-rg/providers/Microsoft.Compute/virtualMachineScaleSets/31230-eastus2-devops-AgentPool-vmss
3.	Control F - Search for ‘image reference’ and remove the references as shown below and click on “PUT” you will get the tick mark.
4.	Open VMSS 
5.	Click on Instances on left blade and select one instance -> click on ‘Upgrade’ wait until it is completed.
6.	Click on same instance which you are selected and click on ‘Reimage’
7.	Once completed you will see latest model as “Yes” as shown below.
8.	How to verify you have latest ones. Go to Home-> Subscriptions-> <sub id> -> Resources -> search for ‘shared images’  -> Click on ‘Ubuntu 20.04’ 
Scroll down and Sort by ‘Published date’ 

9.	Now open VMSS and Instance -> get inside of any instance and click on JSON view
Notice that version exactly matches what we have in golden image.

10.	Go to VMSS -> Operating System 
11.	 Scroll down on the right blade and Check box the ‘Modify custom data’ and then click on ‘Save’
12.	Select both the VMSS and click on ‘Reimage’ 
13.	Verify agents should come online after few minutes




  ```



## Automatic OS Upgrades 
```powershell

#Login and set subscription id 
az login 
az account set --subscription $subscription_id

az group create --name myResourceGroup --location eastus

## Create a Virtual Machine with a public ip. 
az vm create \
  --resource-group myResourceGroup \
  --name myVM \
  --image ubuntults \
  --admin-username azureuser \
  --generate-ssh-keys

## SSH to public vm and install nginx
ssh azureuser@<public ip>
sudo apt-get install -y nginx

## Create a shared image gallery 
az group create --name myGalleryRG --location eastus
az sig create --resource-group myGalleryRG --gallery-name myGallery

## Create an image definition 
az sig image-definition create \
   --resource-group myGalleryRG \
   --gallery-name myGallery \
   --gallery-image-definition myImageDefinition \
   --publisher myPublisher \
   --offer myOffer \
   --sku mySKU \
   --os-type Linux \
   --os-state specialized

## Create an image version based on Virtual machine with nginx 
az sig image-version create \
   --resource-group myGalleryRG \
   --gallery-name myGallery \
   --gallery-image-definition myImageDefinition \
   --gallery-image-version latest \
   --target-regions "southcentralus=1" "eastus=1" \
   --managed-image "/subscriptions/c2483929-bdde-40b3-992e-66dd68f52928/resourceGroups/MyResourceGroup/providers/Microsoft.Compute/virtualMachines/myVM"


## Create a VM scaleset based on image published to shared image gallery 
az group create --name myResourceGroup --location eastus
az vmss create \
   --resource-group myResourceGroup \
   --name myScaleSet \
   --image "/subscriptions/c2483929-bdde-40b3-992e-66dd68f52928/resourceGroups/myGalleryRG/providers/Microsoft.Compute/galleries/myGallery/images/myImageDefinition" \
   --specialized
   
## Show the public ip of the load balancer
  az network public-ip show \
  --resource-group myResourceGroup \
  --name myScaleSetLBPublicIP \
  --query [ipAddress] \
  --output tsv
   
## Verify nginx webpage comes up on public ip of vm 


## Create a load balancer rule to port 80 of nginx 
az network lb rule create \
  --resource-group myResourceGroup \
  --name myLoadBalancerRuleWeb \
  --lb-name myScaleSetLB \
  --backend-pool-name myScaleSetLBBEPool \
  --backend-port 80 \
  --frontend-ip-name loadBalancerFrontEnd \
  --frontend-port 80 \
  --protocol tcp




## If already using health probe from previous step, do not use health extenson. Save the json below as extension.json  
  {
  "type": "extensions",
  "name": "HealthExtension",
  "apiVersion": "2018-10-01",
  "location": "eastus",  
  "properties": {
    "publisher": "Microsoft.ManagedServices",
    "type": "ApplicationHealthLinux",
    "autoUpgradeMinorVersion": true,
    "typeHandlerVersion": "1.0",
    "settings": {
      "protocol": "http",
      "port": "80",
      "requestPath": "/"
    }
  }
}
## Set the health extension on all instances within VMSS 
az vmss extension set \
  --name ApplicationHealthLinux \
  --publisher Microsoft.ManagedServices \
  --version 1.0 \
  --resource-group myResourceGroup \
  --vmss-name myScaleSet \
  --settings ./extension.json

# Get the id of the health probe  
az network lb probe show --resource-group myResourceGroup --lb-name myScaleSetLB --name test

# Update the scaleset with health probe information 
az vmss update --name myScaleSet --resource-group myResourceGroup --query virtualMachineProfile.networkProfile.healthProbe --set virtualMachineProfile.networkProfile.healthProbe.id="subscriptions/c2483929-bdde-40b3-992e-66dd68f52928/resourceGroups/myResourceGroup/providers/Microsoft.Network/loadBalancers/myScaleSetLB/probes/test"

#Update teh instances to get latest updates  
az vmss update-instances --name myScaleSet --resource-group myResourceGroup --instance-ids *

#Enable automatic OS upgrades. 

az vmss update --name myScaleSet --resource-group myResourceGroup --set UpgradePolicy.AutomaticOSUpgradePolicy.EnableAutomaticOSUpgrade=true




  ```


## Rolling upgrades 

```powershell
az vmss update --name <vmss-name> --resource-group <resource-group-name> --set virtualMachineProfile.storageProfile.imageReference.id="<id of the new image>"
```

## Update an existing scaleset with persistent data disks 
```powershell 
az vmss update \
  --resource-group myResourceGroup \
  --name myScaleSet \
  --upgrade-policy-mode automatic \
  --admin-username azureuser \
  --generate-ssh-keys \
  --data-disk-sizes-gb 64 128
  --image "/subscriptions/c2483929-bdde-40b3-992e-66dd68f52928/resourceGroups/myGalleryRG/providers/Microsoft.Compute/galleries/myGallery/images/myImageDefinition" \
  --specialized

# Attach an additional 128Gb data disk
az vmss disk attach \
  --resource-group myResourceGroup \
  --name myScaleSet \
  --size-gb 128

# Install the Azure Custom Script Extension to run a script that prepares the data disks
az vmss extension set \
  --publisher Microsoft.Azure.Extensions \
  --version 2.0 \
  --name CustomScript \
  --resource-group myResourceGroup \
  --vmss-name myScaleSet \
  --settings '{"fileUris":["https://raw.githubusercontent.com/Azure-Samples/compute-automation-configurations/master/prepare_vm_disks.sh"],"commandToExecute":"./prepare_vm_disks.sh"}'


 ```



