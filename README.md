Custom Policy with Azure Policy add-on for AKS
==============================================

Basic demo of Azure Policy for AKS and creating and assigning custom policy.

To assign Azure Policies to Kubernetes you must have either one of these RBAC roles:

* Resource Policy Contributor
* Owner

Installation
------------

```sh
# Provider register: Register the Azure Policy provider
az provider register --namespace Microsoft.PolicyInsights

# Install Azure Policy add-on for AKS
az aks enable-addons --addons azure-policy --name $CLUSTER --resource-group $RESOURCE_GROUP

# azure-policy pod is installed in kube-system namespace
kubectl get pods -n kube-system | grep azure-policy

# gatekeeper pod is installed in gatekeeper-system namespace
kubectl get pods -n gatekeeper-system

# Verify that the latest add-on is installed inm the cluster
az aks show --query addonProfiles.azurepolicy --name $CLUSTER --resource-group $RESOURCE_GROUP
```

Debugging and troubleshooting
-----------------------------

View azure policy and gatekeeper logs:

```sh
# Get the azure-policy pod name installed in kube-system namespace
kubectl logs <azure-policy pod name> -n kube-system

# Get the gatekeeper pod name installed in gatekeeper-system namespace
kubectl logs <gatekeeper pod name> -n gatekeeper-system
```

View install Constraint Templates:

```sh
kubectl get constrainttemplates
```

View mapping between policy definition and contraint template:

```sh
kubectl get constrainttemplates <TEMPLATE_NAME> -o yaml
```

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
    annotations:
    azure-policy-definition-id: /subscriptions/<SUBID>/providers/Microsoft.Authorization/policyDefinitions/<GUID>
    constraint-template-installed-by: azure-policy-addon
    constraint-template: <URL-OF-YAML>
    creationTimestamp: "2021-09-01T13:20:55Z"
    generation: 1
    managedFields:
    - apiVersion: templates.gatekeeper.sh/v1beta1
    fieldsType: FieldsV1
...
```

`<SUBID>` is the subscription ID and `<GUID>` is the ID of the mapped policy definition. `<URL-OF-YAML>` is the source location of the constraint template that the add-on downloaded to install on the cluster.

View constraints related to a constraint template:

```sh
kubectl get <constraintTemplateName>
```

View constraint details:

```sh
kubectl get <CONSTRAINT-TEMPLATE> <CONSTRAINT> -o yaml
```

Assign a built-in policy
------------------------

* Go to **Policy** in Azure Portal
* Go to **Authoring** / **Definitions**
* Select `Kubernetes under **Category** filter
* Type `privileged` in the **Search** box
* Choose **Kubernetes cluster should not allow privileged containers**
* Click **Assign**
* Select your Subscription and Resource Group for your AKs cluster under **Scope**
* Click **Next**
* Click **Next**
* Click **Next**
* Click **Review + create**
* Click **Create**

Allow up to 30 mins for the policy to be assigned to the cluster and to take effect.

```sh
kubectl get constrainttemplate k8sazurecontainernoprivilege -o yaml
kubectl get K8sAzureContainerNoPrivilege
```

If you have enabled Microsoft Defender for Containers or the legacy Micriosoft Defender for Kubernetes plans you can see duplicate policies as audit policies are automatically added to the cluster on your behalf.  Look for the policy with `deny` effect.

Test policy:

```sh
kubectl apply -f nginx-privileged.yaml

# Error from server ([azurepolicy-container-no-privilege-4e736bc37394228c19fd] Privileged container is not allowed: nginx-privileged, securityContext: {"privileged": true}): error when creating "nginx-privileged.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [azurepolicy-container-no-privilege-4e736bc37394228c19fd] Privileged container is not allowed: nginx-privileged, securityContext: {"privileged": true}

kubectl apply -f nginx-non-privileged.yaml
kubectl get pod nginx-non-privileged
kubectl delete pod nginx-non-privileged
```

Create and assign a custom policy
---------------------------------

[Custom policy definitions](https://docs.microsoft.com/en-us/azure/aks/use-azure-policy#create-and-assign-a-custom-policy-definition-preview) are written in JSON.

Use the [block-default-namespace](https://github.com/Azure/Community-Policy/blob/master/Policies/Kubernetes/block-default-namespace/azurepolicy.json) sample as a starting point for your constraint template.  But ensure you change the name (`k8sazureblockdefault`), kind, and package names to avoid clashes with existing templates (built-in or custom) deployed to your AKs cluster.

Base64 encode the template:

```sh
base64 -w0 block-default-namespace-template.yaml
```

Paste base64 encoded constraint template into the Azure Policy JSON file, e.g.:

```json
"then": {
    "effect": "[parameters('effect')]",
    "details": {
        "templateInfo": {
            "sourceType": "Base64Encoded",
            "content": "<base64-encoded-constraint-template>"
        },
        "values": {
            "excludedNamespaces": "[parameters('excludedNamespaces')]"
        },
        "apiGroups": [
            ""
        ],
        "kinds": [
            "Pod"
        ]
    }
}
```

See full example: [`./block-pods-in-default-namespace-policy.json`](./block-pods-in-default-namespace-policy.json)

Go to Azure Portal, Policy, Authoring / Definitions:

* Create a new Definition
* Enter **Name** as "Block pods from using the default namespace in a Kubernetes cluster (custom policy)"
* Choose **Existing** category as `Kubernetes`
* Paste in the custom policy json (from `./block-pods-in-default-namespace-policy.json`)
* Click **Save**

Go to Azure Portal, Policy, Authoring / Assignments:

* Choose your subscription and resource group for your cluster as the scope
* Select your custom policy under **Policy definition**
* Click **Next** to proceed to **Parameters**
* Uncheck "Only show parameters that need input or review"
* Set Effect to `deny`
* Click **Next**
* Click **Next**
* Click **Review + create**
* Click **Create**

Allow up to 30 mins for Azure Policy reconcillation process to detect and apply the new policy.

```sh
kubectl get constrainttemplates -w
# Watch for the new template to appear

NAME                                     AGE
...
k8sazureblockdefault                     3h32m
k8sazureblockdefaultcustom               17m
...

# Then CTRL+C
```

Test that the custom policy blocks creation of a pod in `default` namespace:

```sh
kubectl apply -f nginx-non-privileged.yaml

# Error from server ([azurepolicy-k8sazureblockdefaultcustom-1785ddce6af1a09a04fb] Usage of the default namespace is not allowed, name: nginx-non-privileged, kind: Pod): error when creating "nginx-non-privileged.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [azurepolicy-k8sazureblockdefaultcustom-1785ddce6af1a09a04fb] Usage of the default namespace is not allowed, name: nginx-non-privileged, kind: Pod
```

Cleanup
-------

* Delete the custom policy assignment
* Delete the custom policy defintion
* Disable Azure Policy add-on for AKS

```sh
az aks disable-addons --addons azure-policy --name $CLUSTER --resource-group $RESOURCE_GROUP
```

Resources
---------

* [Understand Azure Policy for Kubernetes clusters](https://docs.microsoft.com/en-us/azure/governance/policy/concepts/policy-for-kubernetes)
* [Secure your cluster with Azure Policy](https://docs.microsoft.com/en-us/azure/aks/use-azure-policy)
* [Azure Policy Built-in Policies - Kubernetes](https://docs.microsoft.com/en-us/azure/governance/policy/samples/built-in-policies#kubernetes)
* [Gatekeeper](https://github.com/open-policy-agent/gatekeeper) - Policy Controller for Kubernetes
* [Open Policy Agent](https://www.openpolicyagent.org/) - Policy-based control for cloud native environments
* [OPA Constraint Framework](https://github.com/open-policy-agent/frameworks/tree/master/constraint)
* [What is Rego?](https://www.openpolicyagent.org/docs/latest/policy-language/#what-is-rego)
* [Community Policy Samples - Kubernetes Polices](https://github.com/Azure/Community-Policy/tree/master/Policies/Kubernetes)
* [Azure Policy definition structure](https://docs.microsoft.com/en-us/azure/governance/policy/concepts/definition-structure)
* [Create and assign a custom policy definition (preview)](https://docs.microsoft.com/en-us/azure/aks/use-azure-policy#create-and-assign-a-custom-policy-definition-preview)
* [Tutorial: Create a custom policy definition](https://docs.microsoft.com/en-us/azure/governance/policy/tutorials/create-custom-policy-definition)
* [Azure Policy for Kubernetes releases support for custom policy](https://techcommunity.microsoft.com/t5/azure-governance-and-management/azure-policy-for-kubernetes-releases-support-for-custom-policy/ba-p/2699466)
* [Use Azure Policy extension for Visual Studio Code](https://docs.microsoft.com/en-us/azure/governance/policy/how-to/extension-for-vscode)