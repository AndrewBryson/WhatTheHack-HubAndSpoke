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

```

##Firewall Policy and Rule Collection Group
```
az network firewall policy create \
    -g $RG \
    -n fw-policy 

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
    --description "Allow VM network traffic" \
    --destination-addresses "172.16.0.0/16" \
    --source-addresses "10.0.0.0/8" \
    --destination-ports "*" \
    --ip-protocols TCP UDP ICMP \
    --collection-priority 10000

```

##Associate the Policy to the Firewall
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

##Routes
```
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

az network route-table create \
    -g $RG \
    -n rt-gateway \
    --disable-bgp-route-propagation true

az network route-table route create \
    -g $RG \
    --route-table-name rt-gateway \
    -n to-firewall \
    --next-hop-type VirtualAppliance \
    --address-prefix 10.0.0.0/8 \
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
    --route-table rt

az network vnet subnet update \
    -g $RG \
    -n GatewaySubnet \
    --vnet-name hub \
    --route-table rt-gateway

```