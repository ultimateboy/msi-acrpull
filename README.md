
# MSI ACR Pull
MSI ACR Pull enables deployments in a Kubernetes cluster to use any user assigned managed identity to pull images from Azure Container Registry. With this, each application can use its own identity to pull container images.

# Install
Run following command to install latest build from main branch. It will install the needed custom resource definition `ACRPullBinding` and deploy msi-acrpull controllers in `msi-acrpull-system` namespace.

```bash
kubectl apply -f https://raw.githubusercontent.com/Azure/msi-acrpull/main/deploy/latest/crd.yaml -f https://raw.githubusercontent.com/Azure/msi-acrpull/main/deploy/latest/deploy.yaml
```

# How to use
> NOTE: following steps assumes you already have:
> 1) An Kubernetes cluster, and have user assigned managed identities on node pool VMSS.
> 1) An ACR, and the user assigned identity has `pull` permission on ACR.

Once msi-acrpull installs to your cluster. All you need is to deploy a custom resource `AcrPullBinding` to the application namesapce to bind an user assigned identity to an ACR. Following sample specifies all pods using default service account in the namespace to use user managed identity `my-acr-puller` to pull image from `veryimportantcr.azurecr.io`.

```yaml
apiVersion: msi-acrpull.microsoft.com/v1beta1
kind: AcrPullBinding
metadata:
  name: acrpulltest
spec:
  acrServer: veryimportantcr.azurecr.io
  managedIdentityResourceID: /subscriptions/712288dc-f816-4242-b73f-a0a87265dcc8/resourceGroups/my-identities/providers/Microsoft.ManagedIdentity/userAssignedIdentities/my-acr-puller
```

Once the custom resource deployed, you can deploy your application to pull images from the ACR. No changes to the application deployment yaml is needed. 

> If the application pod uses a custom service account, then specify `serviceAccountName` property in AcrPullBinding spec.

# How it works
The architecture looks like below. As an user you will create a custom resource `ACRPullBinding`, which binds a managed identity (using client ID or resource ID) to an Azure container registry (using its FQDN). 

Internally, the `ACRPullBindingController` watches the `ACRPullBinding` resource, and for each of them, create a secret in the namespace. The secret content is a Docker image pull config, and the password is the ACR access token that the controller exchanged from ACR using managed identity. The secret will be refreshed 30min before it expire automatically. The controller will also associate the secret to the specified service account in namespace (by default, use the default service account). With this, any pods created in the namespace will automatically pull images from the ACR using the specified managed identity credential.

![Diagram](https://github.com/Azure/msi-acrpull/blob/main/docs/msi-acrpull-flow.png)

# Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.
