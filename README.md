# How to setup Service Principal Authenication across AAD tenants
In this sample, we're going to demostrate how use Service Principal authenciation across tenants which is typically used for requests where a primary and an auxiliary token are required.  We will leverage the typical scenario of pairing vnets from two separate tenants to demostrate.

## Prerequisites for this sample
1. Access to two separate AAD tenants with subscriptions 
2. Within Tenant 1 create a resource group called "vnetpeer1" and a Vnet resource called "vnetpeer1" within it.
3. Do the same within Tenant 2 with resource group, "vnetpeer2" and a Vnet, "vnetpeer2".   

## Instructions

### Tenant 1
1. Create a app registration in your primary tenant, Tenant 1 and mark as a "multi-tenant app".  We'll need it as multi-tenant app so the other Tenant, i.e. Tenant 2 upon accepting consent, creates the equivalent Service Principal (SP) within its tenant.
2. Go to the resource group vnetpeer1 and add the role assignment for the SP with role of "Contributor".  
3. We'll be using the Client Credential flow, which means we'll need to the create a client secret within the app registration.  So go to the "Certificates & secrets" blade within your app registration, create a secret and copy the *Directory (tenant) ID*, *Appplication ID* and *secret* you just created.  These will be used later to make your Postman calls.

### Tenant 2
3. Consent to the mulit-tenant app in Tenant 2 so the SP is created, by opening the consent url.  Make sure to specify the same redirect url as the url set up in your app reg from Step 1.  You'll also need to grab the directory (tenant) ID of Tenant 2 to set up the consent url.
Example url: 
    ```
    https://login.microsoftonline.com/[TENANT2_ID]/oauth2/authorize?client_id=[CLIENT_ID_OF_MULTI_TENATED_APPLICATION]&response_type=code&redirect_uri=https://localhost:1234/
    ```
4. Once tenant wide consent is accepted, do a quick check on Tenant 2 and verify a new SP has been created in the Enterprise Apps blade of the Tenant 2's AAD.
5. Go to the resource group vnetpeer2 and add the role assigment for the SP with role of "Contributor".

### Generate the Access Tokens  
6. We'll need to obtain the access tokens from both SPs in tenant 1 and 2.  
    ### Postman Example
   -  Setup request to get an access token for SP in tenant 1
  https://login.microsoftonline.com/**[TENANT1_ID]**/oauth2/token with the following headers:
      -  grant_type="client_credentials" 
      -  client_id=[appId from Step 2], 
      -  client_secret=[secret from Step 2] 
      -  redirect_url=[redirect url from Step 2]
      -  resource="https://management.core.windows.net/"
  
    **[screen shot here]**


   -  Setup request to get an access token for SP in tenant 2
  https://login.microsoftonline.com/**[TENANT2_ID]**/oauth2/token with the following headers:
      -  grant_type="client_credentials" 
      -  client_id=[appId from Step 2], 
      -  client_secret=[secret from Step 2] 
      -  redirect_url=[redirect url from Step 2]
      -  resource="https://management.core.windows.net/"
  
    **[screen shot here]**

    ### Azcli Example
    Alternatively you can generate these using tokens using azcli commands.
    ```
    az account clear

    az login --service-principal -u "CLIENT_ID" -p "CLIENT_SECRET" --tenant "TENANT1_ID"
    az account get-access-token

    az login --service-principal -u "CLIENT_ID" -p "CLIENT_SECRET" --tenant "TENANT2_ID"
    az account get-access-token
    ```
### Issue the Request 
7. Finally, copy the two access tokens to be used in the Rest API where the primary token is set as the "Bearer" token for the "Authorization" Header and auxiliary token where that is set as "Bearer" token for the "x-ms-authorization-auxiliary" Header.
```
https://management.azure.com/subscriptions/{{SUBSCRIPTION1_ID}}/resourceGroups/{{tenant1_resourcegroup}}/providers/Microsoft.Network/virtualNetworks/vnetpeer1/virtualNetworkPeerings/peer1-peer2?api-version=2018-02-01
```
**[screen shot here]**

8. After completion of the request from vnetpeer1 (tenant1) to the vnetpeer2 (tenant2) we notice a "peer1-peer2" paring in the vnetpeer1 vnet with a Pairing status of "Initiated".
9. Perform the same call but in reverse from vnet2 to vnet1.
```
https://management.azure.com/subscriptions/{{SUBSCRIPTION2_ID}}/resourceGroups/{{tenant2_resourcegroup}}/providers/Microsoft.Network/virtualNetworks/vnetpeer2/virtualNetworkPeerings/peer2-peer1?api-version=2018-02-01
```  
10. Similarily after completion, we notice a "peer2-peer1" pairing in vnetpeer2 vnet and a Peering status of "Connected" since both pairings have been made.
    
**[screen shot here]**   

### Postman Code Sample  
The [Postman Collection](./SP%20auth%20accross%20multiple%20tenants.postman_collection.json) that holds all the requests along with the [Postman Environment](./SP%20auth%20accross%20tenants.postman_environment.json) for all the variables used in the request can be found as part of this repo.  Downloading these will allow your run this sample using your values on your own local Postman.  

### References
- https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/authenticate-multi-tenant
- https://medium.com/@ArsenVlad/azure-vnet-peering-across-azure-active-directory-tenants-using-service-principal-authentication-13c52d3190ab 
