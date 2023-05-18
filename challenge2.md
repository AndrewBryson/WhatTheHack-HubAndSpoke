# Challenge 2

#Commands to be executed via `bash` (WSL or Linux)

#Configuration
```
RG=rg-wth
REGION=uksouth
```

#AzureFirewallSubnet and AzureFirewallManagementSubnet for Basic FW.
```
az network vnet subnet create \
    --name AzureFirewallSubnet \
    --resource-group $RG \
    --vnet-name Hub \
    --address-prefixes 10.0.1.0/24

```

#Firewall deploy
```
az config set extension.use_dynamic_install=yes_without_prompt

## Data plane PIP
az network public-ip create \
    --name fw-pip \
    -g $RG \
    --allocation-method static \
    --sku standard

az network public-ip show \
    --name fw-pip \
    -g $RG

fwprivaddr="$(az network firewall ip-config list -g $RG -f fw --query "[?name=='FW-config'].privateIpAddress" --output tsv | tr -d '[:space:]')"

fwpublicaddr="$(az network public-ip show -g $RG -n fw-pip --query "ipAddress" -o tsv | tr -d '[:space:]')"

```

##Firewall Policy and Rule Collection Group
```
az network firewall policy create \
    -g $RG \
    -n fw-policy 

# Rule Collection Group create
az network firewall policy rule-collection-group create \
    -g $RG \
    --name collection-1 \
    --policy-name fw-policy \
    --priority 1000
```

##Network rules
```
az network firewall policy rule-collection-group collection add-filter-collection \
    -g $RG \
    --policy-name fw-policy \
    --rule-collection-group-name collection-1 \
    --name filter_collection \
    --action Allow \
    --rule-name network_rule \
    --rule-type NetworkRule \
    --description "Azure to onprem" \
    --destination-addresses "172.16.0.0/16" \
    --source-addresses "10.0.0.0/8" \
    --destination-ports "*" \
    --ip-protocols Any \
    --collection-priority 10000

az network firewall policy rule-collection-group collection rule add \
    -g $RG \
    --policy-name fw-policy \
    --collection-name filter_collection \
    --rcg-name collection-1 \
    --name onprem-to-azure \
    --rule-type NetworkRule \
    --description "Onprem to Azure" \
    --destination-addresses "10.0.0.0/8" \
    --source-addresses "172.16.0.0/16" \
    --destination-ports "*" \
    --ip-protocols Any

az network firewall policy rule-collection-group collection rule add \
    -g $RG \
    --policy-name fw-policy \
    --collection-name filter_collection \
    --rcg-name collection-1 \
    --name azure-to-azure \
    --rule-type NetworkRule \
    --description "Intra Azure" \
    --destination-addresses "10.0.0.0/8" \
    --source-addresses "10.0.0.0/8" \
    --destination-ports "*" \
    --ip-protocols Any

az network firewall policy rule-collection-group collection rule add \
    -g $RG \
    --policy-name fw-policy \
    --collection-name filter_collection \
    --rcg-name collection-1 \
    --name all-networks-to-internet \
    --rule-type NetworkRule \
    --description "All networks to internet" \
    --destination-addresses "0.0.0.0/0" \
    --source-addresses "10.0.0.0/8" "172.16.0.0/16" \
    --destination-ports "*" \
    --ip-protocols Any

```

## NAT rules for incoming web traffic
```

# NAT rule collection
az network firewall policy rule-collection-group collection add-nat-collection \
    -n nat_collection \
    --collection-priority 500 \
    --policy-name fw-policy \
    -g $RG \
    --rule-collection-group-name collection-1 \
    --action DNAT \
    --ip-protocols TCP \
    --rule-name port-8080-to-onprem-web-server \
    --source-addresses "*" \
    --description "port-8080-to-onprem-web-server" \
    --destination-addresses "$fwpublicaddr" \
    --destination-ports 8080 \
    --translated-address "172.16.10.4" \
    --translated-port 80

az network firewall policy rule-collection-group collection rule add \
    -g $RG \
    --policy-name fw-policy \
    --collection-name nat_collection \
    --rcg-name collection-1 \
    --name port-8081-to-hub-web-server \
    --rule-type NatRule \
    --source-addresses "*" \
    --description "port-8081-to-hub-web-server" \
    --destination-addresses "$fwpublicaddr" \
    --destination-ports 8081 \
    --translated-address "10.0.10.4" \
    --translated-port 80 \
    --ip-protocols TCP

az network firewall policy rule-collection-group collection rule add \
    -g $RG \
    --policy-name fw-policy \
    --collection-name nat_collection \
    --rcg-name collection-1 \
    --name port-8082-to-spoke1-web-server \
    --rule-type NatRule \
    --source-addresses "*" \
    --description "port-8082-to-spoke1-web-server" \
    --destination-addresses "$fwpublicaddr" \
    --destination-ports 8082 \
    --translated-address "10.1.10.4" \
    --translated-port 80 \
    --ip-protocols TCP

az network firewall policy rule-collection-group collection rule add \
    -g $RG \
    --policy-name fw-policy \
    --collection-name nat_collection \
    --rcg-name collection-1 \
    --name port-8083-to-spoke2-web-server \
    --rule-type NatRule \
    --source-addresses "*" \
    --description "port-8083-to-spoke2-web-server" \
    --destination-addresses "$fwpublicaddr" \
    --destination-ports 8083 \
    --translated-address "10.1.10.4" \
    --translated-port 80 \
    --ip-protocols TCP

```

##Create the Firewall and associate the Policy
```
az network firewall create \
    --name fw \
    -g $RG \
    --sku AZFW_VNet \
    --tier Standard \
    --vnet-name Hub

az network firewall ip-config create \
    --firewall-name fw \
    --name FW-config \
    --public-ip-address fw-pip \
    -g $RG \
    --vnet-name Hub
    
```

## Firewall diagnostic settings
```
fwid=$(az network firewall show -g $RG -n fw --query "id" -o tsv | tr -d '[:space:]')
### Create a Log Analytics Workspace for FW logs
az monitor log-analytics workspace create \
    -g $RG \
    -n law-fw

az monitor diagnostic-settings create \
    -n DiagLogAnalytics \
    --resource $fwid \
    --logs '[{"category":"AZFWNetworkRule", "Enabled":true},{"category":"AZFWApplicationRule", "Enabled":true},{"category":"AZFWNatRule", "Enabled":true}]' \
    --workspace law-fw

```

##Routes
```
# For the spokes
az network route-table create \
    -g $RG \
    -n rt \
    --disable-bgp-route-propagation true

az network route-table route create \
    -g $RG \
    --route-table-name rt \
    -n to-firewall \
    --next-hop-type VirtualAppliance \
    --address-prefix 0.0.0.0/0 \
    --next-hop-ip-address $fwprivaddr

az network route-table route create \
    -g $RG \
    --route-table-name rt \
    -n to-hub \
    --next-hop-type VirtualAppliance \
    --address-prefix 10.0.0.0/16 \
    --next-hop-ip-address $fwprivaddr

# For the GatewaySubnet
az network route-table create \
    -g $RG \
    -n rt-gateway

az network route-table route create \
    -g $RG \
    --route-table-name rt-gateway \
    -n to-firewall \
    --next-hop-type VirtualAppliance \
    --address-prefix 10.1.0.0/16 \
    --next-hop-ip-address $fwprivaddr

# For the Hub subnets
az network route-table create \
    -g $RG \
    -n rt-hub

az network route-table route create \
    -g $RG \
    --route-table-name rt-hub \
    -n spoke1-to-firewall \
    --next-hop-type VirtualAppliance \
    --address-prefix 10.1.0.0/16 \
    --next-hop-ip-address $fwprivaddr

az network route-table route create \
    -g $RG \
    --route-table-name rt-hub \
    -n spoke2-to-firewall \
    --next-hop-type VirtualAppliance \
    --address-prefix 10.2.0.0/16 \
    --next-hop-ip-address $fwprivaddr

az network route-table route create \
    -g $RG \
    --route-table-name rt-hub \
    -n onprem-to-firewall \
    --next-hop-type VirtualAppliance \
    --address-prefix 172.16.0.0/16 \
    --next-hop-ip-address $fwprivaddr

```

##Route to Subnet association
```
az network vnet subnet update \
    -g $RG \
    -n default \
    --vnet-name spoke1 \
    --route-table rt

az network vnet subnet update \
    -g $RG \
    -n default \
    --vnet-name spoke2 \
    --route-table rt
    
az network vnet subnet update \
    -g $RG \
    -n default \
    --vnet-name hub \
    --route-table rt-hub

az network vnet subnet update \
    -g $RG \
    -n GatewaySubnet \
    --vnet-name hub \
    --route-table rt-gateway

```

# apache2 install
```
s=("onprem" "hub" "spoke1" "spoke2") 
for vm in ${s[@]}; 
do
    az vm run-command create \
        -g $RG \
        --async-execution true \
        --name "install-apache2" \
        --script 'apt update && apt install -y apache2' \
        --vm-name $vm
done

```