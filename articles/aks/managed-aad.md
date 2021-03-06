---
title: Use Azure AD in Azure Kubernetes Service
description: Learn how to use Azure AD in Azure Kubernetes Service (AKS) 
services: container-service
manager: gwallace
author: mlearned
ms.topic: article
ms.date: 06/25/2020
ms.author: mlearned
---

# AKS-managed Azure Active Directory integration (Preview)

> [!NOTE]
> Existing AKS (Azure Kubernetes Service) clusters with Azure Active Directory (Azure AD) integration are not affected by the new AKS-managed Azure AD experience.

AKS-managed Azure AD integration is designed to simplify the Azure AD integration experience, where users were previously required to create a client app, a server app, and required the Azure AD tenant to grant Directory Read permissions. In the new version, the AKS resource provider manages the client and server apps for you.

## Azure AD authentication overview

Cluster administrators can configure Kubernetes role-based access control (RBAC) based on a user's identity or directory group membership. Azure AD authentication is provided to AKS clusters with OpenID Connect. OpenID Connect is an identity layer built on top of the OAuth 2.0 protocol. For more information on OpenID Connect, see the [Open ID connect documentation][open-id-connect].

Learn more about the AAD integration flow on the [Azure Active Directory integration concepts documentation](concepts-identity.md#azure-active-directory-integration).

## Limitations

* You can't currently upgrade an existing AKS Azure AD-Integrated cluster to the new AKS-managed Azure AD experience.

> [!IMPORTANT]
> AKS preview features are available on a self-service, opt-in basis. Previews are provided "as-is" and "as available," and are excluded from the Service Level Agreements and limited warranty. AKS previews are partially covered by customer support on a best-effort basis. As such, these features are not meant for production use. For more information, see the following support articles:
>
> - [AKS Support Policies](support-policies.md)
> - [Azure Support FAQ](faq.md)

## Before you begin

* Locate your Azure Account tenant ID by navigating to the Azure portal and select Azure Active Directory > Properties > Directory ID

> [!Important]
> You must use Kubectl with a minimum version of 1.18

You must have the following resources installed:

- The Azure CLI, version 2.5.1 or later
- The aks-preview 0.4.38 extension
- Kubectl with a minimum version of [1.18](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.18.md#v1180)

To install/update the aks-preview  extension or later, use the following Azure CLI commands:

```azurecli
az extension add --name aks-preview
az extension list
```

```azurecli
az extension update --name aks-preview
az extension list
```

To install kubectl, use the following commands:

```azurecli
sudo az aks install-cli
kubectl version --client
```

Use [these instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl/) for other operating systems.

```azurecli-interactive
az feature register --name AAD-V2 --namespace Microsoft.ContainerService
```

It might take several minutes for the status to show as **Registered**. You can check the registration status by using the [az feature list](https://docs.microsoft.com/cli/azure/feature?view=azure-cli-latest#az-feature-list) command:

```azurecli-interactive
az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/AAD-V2')].{Name:name,State:properties.state}"
```

When the status shows as registered, refresh the registration of the `Microsoft.ContainerService` resource provider by using the [az provider register](https://docs.microsoft.com/cli/azure/provider?view=azure-cli-latest#az-provider-register) command:

```azurecli-interactive
az provider register --namespace Microsoft.ContainerService
```

## Create an AKS cluster with Azure AD enabled

Create an AKS cluster by using the following CLI commands.

Create an Azure resource group:

```azurecli-interactive
# Create an Azure resource group
az group create --name myResourceGroup --location centralus
```

You can use an existing Azure AD group, or create a new one. You need the object ID for your Azure AD group.

```azurecli-interactive
# List existing groups in the directory
az ad group list
```

To create a new Azure AD group for your cluster administrators, use the following command:

```azurecli-interactive
# Create an Azure AD group
az ad group create --display-name MyDisplay --mail-nickname MyDisplay
```

Create an AKS cluster, and enable administration access for your Azure AD group

```azurecli-interactive
# Create an AKS-managed Azure AD cluster
az aks create -g MyResourceGroup -n MyManagedCluster --enable-aad [--aad-admin-group-object-ids <id>] [--aad-tenant-id <id>]
```

A successful creation of an AKS-managed Azure AD cluster has the following section in the response body
```
"AADProfile": {
    "adminGroupObjectIds": null,
    "clientAppId": null,
    "managed": true,
    "serverAppId": null,
    "serverAppSecret": null,
    "tenantId": "72f9****-****-****-****-****d011db47"
  }
```

The cluster is created within a few minutes.

## Access an Azure AD enabled cluster

You'll need the [Azure Kubernetes Service Cluster User](https://docs.microsoft.com/azure/role-based-access-control/built-in-roles#azure-kubernetes-service-cluster-user-role) built-in role to do the following steps.

Get the user credentials to access the cluster:
 
```azurecli-interactive
 az aks get-credentials --resource-group myResourceGroup --name MyManagedCluster
```
Follow the instructions to sign in.

Use the kubectl get nodes command to view nodes in the cluster:

```azurecli-interactive
kubectl get nodes

NAME                       STATUS   ROLES   AGE    VERSION
aks-nodepool1-15306047-0   Ready    agent   102m   v1.15.10
aks-nodepool1-15306047-1   Ready    agent   102m   v1.15.10
aks-nodepool1-15306047-2   Ready    agent   102m   v1.15.10
```
Configure [Role Based Access Control (RBAC)](https://docs.microsoft.com/azure/aks/azure-ad-rbac) to configure additional security groups for your clusters.

## Troubleshooting access issues with Azure AD

> [!Important]
> The steps described below are bypassing the normal Azure AD group authentication. Use them only in an emergency.

If you're permanently blocked by not having access to a valid Azure AD group with access to your cluster, you can still obtain the admin credentials to access the cluster directly.

To do these steps, you'll need to have access to the [Azure Kubernetes Service Cluster Admin](https://docs.microsoft.com/azure/role-based-access-control/built-in-roles#azure-kubernetes-service-cluster-admin-role) built-in role.

```azurecli-interactive
az aks get-credentials --resource-group myResourceGroup --name MyManagedCluster --admin
```

## Non-interactive sign in with kubelogin

There are some non-interactive scenarios, such as continuous integration pipelines, that aren't currently available with kubectl. You can use [`kubelogin`](https://github.com/Azure/kubelogin) to access the cluster with non-interactive service principal sign-in.

## Next steps

* Learn about [Azure RBAC integration for Kubernetes Authorization][azure-rbac-integration]
* Learn about [Azure AD integration with Kubernetes RBAC][azure-ad-rbac].
* Use [kubelogin](https://github.com/Azure/kubelogin) to access features for Azure authentication that aren't available in kubectl.
* Learn more about [AKS and Kubernetes identity concepts][aks-concepts-identity].
* Use [Azure Resource Manager (ARM) templates ][aks-arm-template] to create AKS-managed Azure AD enabled clusters.

<!-- LINKS - external -->
[kubernetes-webhook]:https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication
[kubectl-apply]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply
[aks-arm-template]: https://docs.microsoft.com/azure/templates/microsoft.containerservice/managedclusters

<!-- LINKS - Internal -->
[azure-rbac-integration]: manage-azure-rbac.md
[aks-concepts-identity]: concepts-identity.md
[azure-ad-rbac]: azure-ad-rbac.md
[az-aks-create]: /cli/azure/aks?view=azure-cli-latest#az-aks-create
[az-aks-get-credentials]: /cli/azure/aks?view=azure-cli-latest#az-aks-get-credentials
[az-group-create]: /cli/azure/group#az-group-create
[open-id-connect]:../active-directory/develop/v2-protocols-oidc.md
[az-ad-user-show]: /cli/azure/ad/user#az-ad-user-show
[rbac-authorization]: concepts-identity.md#role-based-access-controls-rbac
[operator-best-practices-identity]: operator-best-practices-identity.md
[azure-ad-rbac]: azure-ad-rbac.md
[azure-ad-cli]: azure-ad-integration-cli.md

