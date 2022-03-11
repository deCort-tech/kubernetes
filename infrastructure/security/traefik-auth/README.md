```
kubectl create secret generic authenticationserver 
--from-literal=issuerurl=https://sts.windows.net/<YOUR AZURE TENANT ID>/ 
--from-literal=clientid=<YOUR APPLICATION ID> 
--from-literal=clientsecret=<YOUR CLIENT SECRET> 
--from-literal=jwtsecret=<YOUR JWT SECRET>
```
- The Azure Tenant ID can be retrieved by executing the command az account get-access-token --query tenant --output tsv.
- The Application ID and Client Secret are the values that were output from the Azure AD application creation script run earlier.
- The JWT SECRET is used to sign the authentication cookie and can be any randomly generated value. If you have open ssl installed you can easily generate one by running the command openssl rand -hex 16.

https://medium.com/@javaadpatel/securing-traefik-dashboard-with-azure-ad-50a9f6e79552