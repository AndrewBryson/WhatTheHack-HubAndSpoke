# Challenge 1

#Commands to be executed via `bash` (WSL or Linux)

#Configuration
```
RG=rg-wth
REGION=uksouth
VMUSERNAME=azureuser
VMPASSWORD=""
```
#Resource Group
```
az group create -l "$REGION" -n "$RG"
```

#Virtual Networks
#Hub
```
az network vnet create \
  --name Hub \
  --resource-group $RG \
  --address-prefix 10.0.0.0/16
  ```

#Spokes
```
az network vnet create \
  --name Spoke1 \
  --resource-group $RG \
  --address-prefix 10.1.0.0/16
```
```
az network vnet create \
  --name Spoke2 \
  --resource-group $RG \
  --address-prefix 10.2.0.0/16
```

#On-prem
```
az network vnet create \
  --name Onprem \
  --resource-group $RG \
  --address-prefix 172.16.0.0/16
```

#Peerings
```
# Get the id for Hub
HubId=$(az network vnet show --resource-group $RG --name Hub --query id --out tsv)

# Get the id for Spoke1.
Spoke1=$(az network vnet show --resource-group $RG --name Spoke1 --query id --out tsv)

# Peer Hub to Spoke1.
az network vnet peering create \
  --name LinkHubToSpoke1 \
  --resource-group $RG \
  --vnet-name Hub \
  --remote-vnet Spoke1 \
  --allow-vnet-access \
  --allow-gateway-transit \
  --allow-forwarded-traffic

# Peer Spoke1 to Hub.
az network vnet peering create \
--name LinkSpoke1ToHub \
--resource-group $RG \
--vnet-name Spoke1 \
--remote-vnet Hub \
--allow-vnet-access \
--use-remote-gateways \
--allow-forwarded-traffic

# Peer Hub to Spoke2.
az network vnet peering create \
--name LinkHubToSpoke2 \
--resource-group $RG \
--vnet-name Hub \
--remote-vnet Spoke2 \
  --allow-vnet-access \
  --allow-gateway-transit \
  --allow-forwarded-traffic

# Peer Spoke2 to Hub.
az network vnet peering create \
--name LinkSpoke2ToHub \
--resource-group $RG \
--vnet-name Spoke2 \
--remote-vnet Hub \
--allow-vnet-access \
--use-remote-gateways \
--allow-forwarded-traffic
```

#Subnets
#Hub

```
az network vnet subnet create --name GatewaySubnet --resource-group $RG --vnet-name Hub --address-prefixes 10.0.0.0/24

az network vnet subnet create --name default --resource-group $RG --vnet-name Hub --address-prefixes 10.0.10.0/24
```

#Spoke1

```
az network vnet subnet create --name default --resource-group $RG --vnet-name Spoke1 --address-prefixes 10.1.10.0/24
```

#Spoke2

```
az network vnet subnet create --name default --resource-group $RG --vnet-name Spoke2 --address-prefixes 10.2.10.0/24
```

#On-prem
```
az network vnet subnet create --name GatewaySubnet --resource-group $RG --vnet-name Onprem --address-prefixes 172.16.0.0/24

az network vnet subnet create --name default --resource-group $RG --vnet-name Onprem --address-prefixes 172.16.10.0/24
```

#Virtual Machines

#Hub
```
az vm create -n hub -g $RG \
    --public-ip-address "" \
    --vnet-name Hub --subnet default \
    --image Ubuntu2204 \
    --size Standard_B1s \
    --authentication-type "password" \
    --admin-username "$VMUSERNAME" --admin-password "$VMPASSWORD"
```

#Spoke1
```
az vm create -n spoke1 -g $RG \
    --public-ip-address "" \
    --vnet-name Spoke1 --subnet default \
    --image Ubuntu2204 \
    --size Standard_B1s \
    --authentication-type "password" \
    --admin-username "$VMUSERNAME" --admin-password "$VMPASSWORD"
```

#Spoke2
```
az vm create -n spoke2 -g $RG \
    --public-ip-address "" \
    --vnet-name Spoke2 --subnet default \
    --image Ubuntu2204 \
    --size Standard_B1s \
    --authentication-type "password" \
    --admin-username "$VMUSERNAME" --admin-password "$VMPASSWORD"
```

#Onprem
```
az vm create -n onprem -g $RG \
    --public-ip-address "" \
    --vnet-name Onprem --subnet default \
    --image Ubuntu2204 \
    --size Standard_B1s \
    --authentication-type "password" \
    --admin-username "$VMUSERNAME" --admin-password "$VMPASSWORD"
```

#Enable Boot Diagnostics with a managed storage account
```
az vm boot-diagnostics enable -g $RG -n hub
az vm boot-diagnostics enable -g $RG -n spoke1
az vm boot-diagnostics enable -g $RG -n spoke2
az vm boot-diagnostics enable -g $RG -n onprem
```

#VPN Connection

#Public IPs
```
az network public-ip create \
  -n pip-hub \
  -g $RG \
  --allocation-method Dynamic

az network public-ip create \
  -n pip-onprem \
  -g $RG \
  --allocation-method Dynamic

PIPofVPNGWHub=$(az network public-ip show -n pip-hub -g $RG --query "ipAddress" -o tsv | tr -d '[:space:]')
PIPofVPNGWOnPrem=$(az network public-ip show -n pip-onprem -g $RG --query "ipAddress" -o tsv | tr -d '[:space:]')

echo "PIPofVPNGWHub: $PIPofVPNGWHub"
echo "PIPofVPNGWOnPrem: $PIPofVPNGWOnPrem"
```

### VPN Gateways
```
az network vnet-gateway create \
  -g $RG \
  -n vpngw-hub \
  --public-ip-address pip-hub \
  --vnet Hub \
  --gateway-type Vpn \
  --sku VpnGw1 \
  --vpn-type RouteBased \
  --asn 65010 \
  --no-wait

az network vnet-gateway create \
  -g $RG \
  -n vpngw-onprem \
  --public-ip-address pip-onprem \
  --vnet onprem \
  --gateway-type Vpn \
  --sku VpnGw1 \
  --vpn-type RouteBased \
  --asn 65011 \
  --no-wait

IDofVPNGWHub=$(az network vnet-gateway show -n vpngw-hub -g $RG --query "id" -o tsv)
IDofVPNGWOnPrem=$(az network vnet-gateway show -n vpngw-onprem -g $RG --query "id" -o tsv)
```

#Wait until the vNet Gateways are created
```
az network vnet-gateway wait --updated --ids $IDofVPNGWHub $IDofVPNGWOnPrem
```

#Create the Local Network Gateways

```
PeerAddressOfHub=$(az network vnet-gateway show -n vpngw-hub -g $RG --query "bgpSettings.bgpPeeringAddress" -o tsv | tr -d '[:space:]')
PeerAddressOfOnPrem=$(az network vnet-gateway show -n vpngw-onprem -g $RG --query "bgpSettings.bgpPeeringAddress" -o tsv | tr -d '[:space:]')

echo "PeerAddressOfHub: $PeerAddressOfHub"
echo "PeerAddressOfOnPrem: $PeerAddressOfOnPrem"

az network local-gateway create \
  --gateway-ip-address $PIPofVPNGWOnPrem \
  -n lng-onprem \
  -g $RG \
  --local-address-prefixes $PeerAddressOfOnPrem/32 \
  --asn 65011 \
  --bgp-peering-address $PeerAddressOfOnPrem

az network local-gateway create \
  --gateway-ip-address $PIPofVPNGWHub \
  -n lng-hub \
  -g $RG \
  --local-address-prefixes $PeerAddressOfHub/32 \
  --asn 65010 \
  --bgp-peering-address $PeerAddressOfHub  

IDofLNGtoHub=$(az network local-gateway show -n lng-hub -g $RG --query "id" -o tsv)
IDofLNGtoOnPrem=$(az network local-gateway show -n lng-onprem -g $RG --query "id" -o tsv)

echo "IDofLNGtoHub: $IDofLNGtoHub"
echo "IDofLNGtoOnPrem: $IDofLNGtoOnPrem"
```

# Create the VPN Connections
```
az network vpn-connection create \
  -n conn-to-onprem \
  -g $RG \
  --vnet-gateway1 vpngw-hub \
  --enable-bgp \
  --shared-key "abc123" \
  --local-gateway2 lng-onprem

az network vpn-connection create \
  -n conn-to-hub \
  -g $RG \
  --vnet-gateway1 vpngw-onprem \
  --enable-bgp \
  --shared-key "abc123" \
  --local-gateway2 lng-hub
```