# How to integrate APIM with Service Fabric service

To implement the following topology with APIM and Service Fabric

![](\images\Topology.png?raw=true)

#### Perquisitions :
- Service Fabric Cluster
- APIM 
- Service Fabric Stateless service


1. Create a subnet within the VNET  where Service Fabric is deployed and then deploy APIM into the newly created subnet

![](images\VirtualNetwork.png?raw=true)

2. Export the Service Fabric certificate from Azure Key Vault. If you have the certificate already, please skip this step

```sh
$vaultName = "{AzureKeyVaultName}"
$certificateName = "{CertName}"
$pfxPath = [Environment]::GetFolderPath("Desktop") + "\$sfclustercert.pfx"
$password = "{Password}"

$pfxSecret = Get-AzureKeyVaultSecret -VaultName $vaultName -Name $certificateName
$pfxUnprotectedBytes = [Convert]::FromBase64String($pfxSecret.SecretValueText)
$pfx = New-Object Security.Cryptography.X509Certificates.X509Certificate2
$pfx.Import($pfxUnprotectedBytes, $null, [Security.Cryptography.X509Certificates.X509KeyStorageFlags]::Exportable)
$pfxProtectedBytes = $pfx.Export([Security.Cryptography.X509Certificates.X509ContentType]::Pkcs12, $password)
[IO.File]::WriteAllBytes($pfxPath, $pfxProtectedBytes)
```

3. Upload the certificate to the client certificate blade of APIM
![](\images\clientcertblade.png?raw=true)

4. Import stateless service API into APIM by using swagger file
![](images\importapi.png?raw=true)

**Please note that you dont need to set the backend URL for the backend API as we will add a backend later to get the actual backend URL from Service Fabric Naming Service**
![](images\apisetting.png?raw=true)

5. Now we need to create a backend object in APIM 

https://docs.microsoft.com/en-us/rest/api/apimanagement/2019-01-01/backend/createorupdate
```sh
URL: https://management.azure.com/subscriptions/72ec07e7-1eb7-40c3-8847-075fd68da703/resourceGroups/HarryTestResourceGroup/providers/Microsoft.ApiManagement/service/harryTestAPIM/backends/harryServicefabric?api-version=2019-01-01

Method: PUT

Body payload:

{
	"properties": {
		"description": "Harry Service Fabric",
		"url": "fabric:/SFGetMyName/MyNameService",
		"protocol": "http",
		"properties": {
			"serviceFabricCluster": {
				"managementEndpoints": [
					"https://harrysfcluster.southeastasia.cloudapp.azure.com:19080"
				],
				"clientCertificatethumbprint": "{ServiceFabricClusterCertificateThumbprint}",
				"serverCertificateThumbprints": ["{ServiceFabricClusterCertificateThumbprint}"],
				"maxPartitionResolutionRetries": 5
			}
		}
	}
}
```

Please note the URL is the service name of the stateless service. Replace the highlighted part with your actual resource values


6. Add the inbound policy to your API in APIM at API scope. Please replace the highlighted part with your actual resource values. Backend-id is the backend id which you just created in step 5. sf-service-instance-name is the name of the stateless service from Service Fabric

```sh
<inbound>
	<set-backend-service backend-id="harryServicefabric" sf-resolve-condition="@(context.LastError?.Reason == "BackendConnectionFailure")" sf-service-instance-name="fabric:/SFGetMyName/MyNameService" />
	<base />
</inbound>
```

7. Now your APIM is configured with Service Fabric cluster.

8. Try to invoke one of the operations from API in APIM and observe the trace log from APIM. You will see APIM is calling the management endpoint to get the stateless service parition information  (Service Fabric Naming service is resolving partition and return each replica endpoint). The URL that APIM calling is "https://harrysfcluster.southeastasia.cloudapp.azure.com:19080/Services/SFGetMyName/MyNameService/$/ResolvePartition?api-version=3.0&PartitionKeyType=1&timeout=60"(In my example)
![](\images\resolvepartition.png?raw=true)

We can invoke the get requesthttps://harrysfcluster.southeastasia.cloudapp.azure.com:19080/Services/SFGetMyName/MyNameService/$/ResolvePartition?api-version=3.0&PartitionKeyType=1&timeout=60  directly and observe the info that Service Fabric naming service returned 
![](\images\namingserviceresolveresult.png?raw=true)

APIM then choose one of the resolved replica endpoints and send request to service fabric.
![](\images\apimforwardrequest.png?raw=true)
