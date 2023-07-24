# acm-onboarding-examples

This repo focuses on showing examples of how to use [Red Hat Advanced Cluster management](https://www.redhat.com/en/technologies/management/advanced-cluster-management) for onboarding users and application unto OpenShift Platform/Kubernetes Platform.

### Requirements
- The user running the below commands must have cluster-admin privilege.
- The user running this commands must have subscription-admin privileges.
    ```
    oc adm policy add-cluster-role-to-user open-cluster-management:subscription-admin $(oc whoami)
    ```
- The policy example used below use a namespace called global-policies. The global policy namespace must have a managedclustersetbinding for all clusters.
    ```
    oc apply -k ./acm-onboarding-examples/requirements/
    ```

### ACM Usage Example 1  
(./namespace-kustomize-example1/) shows an example of how a hierarchical configuration of namespaces might work. [Team A](./namespace-kustomize-example1/team-overlays-build/team-a/) and [Team B](./namespace-kustomize-example1/team-overlays-build/team-b/) inherit a Secret, ClusterResourceQuota and LimitRange from the [global-team configuration](./namespace-kustomize-example1/global-teams) customized for each team via the use of [Kustomize](https://kubectl.docs.kubernetes.io/references/kustomize/).

With kustomize we can apply and delete each team's resources with ease like below:
```
oc apply -k ./acm-onboarding-examples/namespace-kustomize-example1/team-overlays-build/team-a/
```
```
oc apply -k ./acm-onboarding-examples/namespace-kustomize-example1/team-overlays-build/team-b/
```

One problem that might still exist though is how to handle the creation of the above manifests in a scalable fashion e.g Team A might need these manifests applied on 2 clusters they manage while team b might need these manifests applied to ephemeral clusters they spin up and spin down weekly.This is a problem Red Hat's Advanced Cluster Management can help solve.

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
    - Using ACM we can control which clusters get our team manifests applied. 

    




    