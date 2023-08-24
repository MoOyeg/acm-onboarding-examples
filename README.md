# acm-onboarding-examples

This repo focuses on showing examples of how to use [Red Hat Advanced Cluster management](https://www.redhat.com/en/technologies/management/advanced-cluster-management) for onboarding users and application unto OpenShift Platform/Kubernetes Platform.

### Requirements
- Red Hat Advanced Cluster Management must be installed.
- The user running the below commands must have cluster-admin privilege.
- The user running this commands must have subscription-admin privileges.
    ```
    oc adm policy add-cluster-role-to-user open-cluster-management:subscription-admin $(oc whoami)
    ```
- The policy example used below use a namespace called global-policies. The global policy namespace must have a managedclustersetbinding for all clusters.
    ```
    oc apply -k ./acm-onboarding-examples/requirements/
    ```

### Notes
- Some of the RHACM policy examples used below are quite intensive.If they are to be used in a realistic setting their [evaluation intervals should be set to a reasonable value](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.8/html/governance/governance#configuration-policy-yaml-table)

### ACM Usage Example 1  
(./namespace-kustomize-example1/) shows an example of how a hierarchical configuration of namespaces might work. [Team A](./namespace-kustomize-example1/team-overlays-build/team-a/) and [Team B](./namespace-kustomize-example1/team-overlays-build/team-b/) inherit a Secret, ClusterResourceQuota and LimitRange from the [global-team configuration](./namespace-kustomize-example1/global-teams) customized for each team via the use of [Kustomize](https://kubectl.docs.kubernetes.io/references/kustomize/).

With kustomize we can apply and delete each team's resources with ease like below:
```
oc apply -k ./acm-onboarding-examples/namespace-kustomize-example1/team-overlays-build/team-a/
```
```
oc apply -k ./acm-onboarding-examples/namespace-kustomize-example1/team-overlays-build/team-b/
```

One problem that might still exist though is how to handle the creation of the above manifests in a scalable fashion e.g Team A might need these manifests applied on 2 clusters they manage while team b might need these manifests applied to ephemeral clusters they spin up and spin down weekly.This is a problem Red Hat's Advanced Cluster Management can help solve.Using ACM we can control which clusters get our team manifests applied.

  - Red Hat Advanced Cluster Management provides(RHACM) a [Policy Generator](https://github.com/stolostron/policy-generator-plugin/tree/main). The Policy Generator allows us to generate RHACM policies by passing existing yaml manifests directly to kustomize. This makes it much easier to move existing policies to RHACM. Review the [policygenerator example file](./namespace-kustomize-example1/team-overlays-build/policygenerator.yaml) to understand more.
  
    **Note**: To use the policy generator kindly review the link above for the installation process.

    - With the policygenerator plugin installed we can create our RHACM Policies.

        ```
        kustomize build --enable-alpha-plugins ./acm-onboarding-examples/namespace-kustomize-example1/team-overlays-build/ | oc create -f -
        ```

    - Now our policies which create our team-a and team-b manifests are ready to be applied to any of our clusters. We can get the list of clusters. Using our local-cluster as an example.

        - Get list of ManagedClusters
            ```
            oc get managedclusters
            ```

        - Get Namespaces we might have created. Since our manifests are tagged with a label we can use that to check. It should return no resources. If it does return resources , please run a delete like below.
            ```
            oc delete -k ./acm-onboarding-examples/namespace-kustomize-example1/team-overlays-build/team-a/
            ```
            ```
            oc delete -k ./acm-onboarding-examples/namespace-kustomize-example1/team-overlays-build/team-b/
            ```
            ```
            oc get namespaces -l deployer=kustomize-global
            ```
        - Our team-a manifests will only be applied to clusters tagged with team-a-cluster and our team-b manifests will only be tagged with team-b-cluster. [Policy Generator file](./namespace-kustomize-example1/team-overlays-build/policygenerator.yaml).
        
        - Let's tag our local-cluster to get manifests team-a manifests applied
            ```
            oc label managedcluster local-cluster team-a-cluster=true
            ```
        
        - We can confirm by
            ```
            oc get namespaces -l deployer=kustomize-global
            ```


### ACM Usage Example 2 

There are still a few things in [ACM Usage Example 1](#acm-usage-example-1) above that might be difficult to do in a real environment. An example of this would include we want a 'global-secret-for-every-namespace' to be created in every namespace we create, and we used a [Kustomize SecretGenerator](./namespace-kustomize-example1/global-teams/namespaced-resources/kustomization.yaml) to achieve this.Due to the nature of secrets we might prefer that our secrets be created via a seperate process and be available to any namespace we create. We can do this via ACM functions and labels.[ACM provides a series of Template Functions that allow us to do processing on templates we generate](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.8/html/governance/governance#template-functions).

   - We have created a [policy](./namespace-kustomize-example2/configurations/global-secret-example-resources/policy-global-secret.yaml) to help us manage the creation of our secrets.Let's review the policy
     
       - 'object-templates-raw:' means we will be creating an advanced template that can support range and if else functions.

       - '{{- range (lookup "v1" "Namespace" "" "" "deployer=kustomize-global").items }}' means we are going to loop through (range) all objects of type namespace that are labeled with deployer=kustomize-global and apply the template below that line for every object in that loop.

       - 'username: {{hub fromSecret "global-policies" "global-secret-for-every-namespace" "username" hub}}' means we are going to lookup the value of key username from an existing secret called 'global-secret-for-every-namespace' in the 'global-policies' namespace. We also do the same for value secret.

       - 'namespace: '{{ .metadata.name }}' means we are going to take the name of the object we looped through, and since we are looping through namespaces that would be the namespace name.

       - The final effect of this policy is we would be creating a secret in every namespace labeled with deployer=kustomize-global. And in this example this policy would be applied on every cluster in ACM [we set it for our global clusterset](./namespace-kustomize-example2/configurations/global-secret-example-resources/placement-global-secret.yaml)

       - We can apply this policy
            ```
            oc apply -k ./acm-onboarding-examples/namespace-kustomize-example2/configurations/global-secret-example-resources
            ```

### ACM Usage Example 3
We can also use ACM to improve our application delivery.A common use case might be to create a Kubernetes object for every user in a group. Since we do not want to hardcode the users, we can use ACM to look up user group membership and then create the necessary kubernetes object. The example below creates a policy that will create a devworkspace via [OpenShift DevSpaces](https://access.redhat.com/products/red-hat-openshift-dev-spaces/).

This example requires that you have:
- [A storageclass that is annotated as the default storage class](https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/).If you don't customize templates to the your specific storage class.
- Run the requirements for this scenario. This will create an htpasswd provider. It will create multiple htpasswd users demo-test-user[1-6] all with password "test".Please allow OCP to complete the oauth update.
    ```
    oc apply -k ./acm-onboarding-examples/namespace-kustomize-example3/requirements/ -n global-policies
    ```

- Apply the kustomize team files similar to the previous examples
    ```
    kustomize build --enable-alpha-plugins ./acm-onboarding-examples/namespace-kustomize-example3/team-overlays-build/ | oc create -f -
    ```

- Let's tag our local-cluster to get manifests team-a manifests applied
    ```
    oc label managedcluster local-cluster team-a-cluster=true
    ```

- Finally apply our custom devspaces Policy
    ```
    kustomize build --enable-alpha-plugins ./acm-onboarding-examples/namespace-kustomize-example3/team-specific-policies/team-a-policies| oc create -f -
    ```

    - How does the Policy work?
      Review [Policy File](namespace-kustomize-example3/team-specific-policies/team-a-policies/team-a-devspaces/policy-team-a-devspaces.yaml).

       Loop though ll the users in the platform and then check if the user is in team-a.

       ```
       {{- range (lookup "user.openshift.io/v1" "User" "" "" "").items }}          
       {{- if contains .metadata.name ((lookup "user.openshift.io/v1" "Group" "" "team-a" "").users | join "_") }}







    