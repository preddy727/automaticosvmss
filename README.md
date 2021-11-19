# Azure Virtual Machine Scale set upgrade options 
## Manual 



## Automatic OS Upgrades 
```powershell

az login 
az account set --subscription $subscription_id

az group create --name myResourceGroup --location eastus

az vm create \
  --resource-group myResourceGroup \
  --name myVM \
  --image ubuntults \
  --admin-username azureuser \
  --generate-ssh-keys

ssh azureuser@<public ip>
sudo apt-get install -y nginx

az group create --name myGalleryRG --location eastus
az sig create --resource-group myGalleryRG --gallery-name myGallery

az sig image-definition create \
   --resource-group myGalleryRG \
   --gallery-name myGallery \
   --gallery-image-definition myImageDefinition \
   --publisher myPublisher \
   --offer myOffer \
   --sku mySKU \
   --os-type Linux \
   --os-state specialized

az sig image-version create \
   --resource-group myGalleryRG \
   --gallery-name myGallery \
   --gallery-image-definition myImageDefinition \
   --gallery-image-version latest \
   --target-regions "southcentralus=1" "eastus=1" \
   --managed-image "/subscriptions/c2483929-bdde-40b3-992e-66dd68f52928/resourceGroups/MyResourceGroup/providers/Microsoft.Compute/virtualMachines/myVM"

az group create --name myResourceGroup --location eastus
az vmss create \
   --resource-group myResourceGroup \
   --name myScaleSet \
   --image "/subscriptions/c2483929-bdde-40b3-992e-66dd68f52928/resourceGroups/myGalleryRG/providers/Microsoft.Compute/galleries/myGallery/images/myImageDefinition" \
   --specialized

az network lb rule create \
  --resource-group myResourceGroup \
  --name myLoadBalancerRuleWeb \
  --lb-name myScaleSetLB \
  --backend-pool-name myScaleSetLBBEPool \
  --backend-port 80 \
  --frontend-ip-name loadBalancerFrontEnd \
  --frontend-port 80 \
  --protocol tcp

  az network public-ip show \
  --resource-group myResourceGroup \
  --name myScaleSetLBPublicIP \
  --query [ipAddress] \
  --output tsv

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

az vmss extension set \
  --name ApplicationHealthLinux \
  --publisher Microsoft.ManagedServices \
  --version 1.0 \
  --resource-group myResourceGroup \
  --vmss-name myScaleSet \
  --settings ./extension.json

az network lb probe show --resource-group myResourceGroup --lb-name myScaleSetLB --name test
az vmss update --name myScaleSet --resource-group myResourceGroup --query virtualMachineProfile.networkProfile.healthProbe --set virtualMachineProfile.networkProfile.healthProbe.id="subscriptions/c2483929-bdde-40b3-992e-66dd68f52928/resourceGroups/myResourceGroup/providers/Microsoft.Network/loadBalancers/myScaleSetLB/probes/test"
az vmss update-instances --name myScaleSet --resource-group myResourceGroup --instance-ids *

az vmss update --name myScaleSet --resource-group myResourceGroup --set UpgradePolicy.AutomaticOSUpgradePolicy.EnableAutomaticOSUpgrade=true




  ```


## Rolling upgrades 



