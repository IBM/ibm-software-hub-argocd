## Prerequisites

1. Prepare an OpenShift cluster (target cluster) with proper storage classes required by CPD/WatsonX, and set up image pull secrets  https://www.ibm.com/docs/en/cloud-paks/cp-data/5.0.x?topic=installing-preparing-your-cluster
2. Install Openshift Gitops (ArgoCD). https://docs.openshift.com/gitops/1.14/installing_gitops/installing-openshift-gitops.html. ArgoCD may be installed in a different cluster. If so, connect ArgoCD with your target cluster. https://github.ibm.com/IBMSoftwareHub/ibm-software-hub-argoCD/wiki/Adding-Cluster-to-ArgoCD
3. Configure ArgoCD to pull from https://github.com/IBM/ibm-software-hub-argocd.git repo. See https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/.
4. Set up global pull secret in the target cluster. See https://www.ibm.com/docs/en/cloud-paks/cp-data/5.0.x?topic=cluster-updating-global-image-pull-secret
5. Deploy all cluster single components: cert-manager, ibm licensing service and CPD scheduler. Use either cpd-cli or ArgoCD. cpd-cli methods: https://www.ibm.com/docs/en/cloud-paks/cp-data/5.0.x?topic=cluster-installing-shared-components. ArgoCD method: See `install-cluster-singleton.md` file.

## Procedure for Install/Upgrade

1. Create operator and instance namespaces. 
```
oc new-project <operator-ns>
oc new-project <operand-ns>
```

2. Grant ArgoCD access to these new namespaces. If ArgoCD is on the same cluster as the target cluster, run
```bash
oc label namespace <operator-ns> argocd.argoproj.io/managed-by=openshift-gitops
oc label namespace <operand-ns> argocd.argoproj.io/managed-by=openshift-gitops
```

3. Create a `ibm-entitlement-key` secret in the instance namespace. 
   ``` bash
   oc create secret docker-registry ibm-entitlement-key  --docker-username=cp --docker-server=cp.icr.io -n <cpd-instance-ns> --docker-password=<ibm-entilement-key>
   ```

4. Deploy a cpd-root app.
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cpd-root #-<suffix> #add an optional suffix to your root if you plan to deploy multiple.
  namespace: openshift-gitops
spec:
  destination:
    name: ''
    namespace: default
    server: 'https://kubernetes.default.svc'
  source:
    path: 5.0.0/root #[INPUT] Change release version (e.g. 5.1.0/root), must match with the value of cpd_release parameter
    repoURL: 'https://github.com/IBM/ibm-software-hub-argocd'
    targetRevision: main
    helm:
      parameters:
        - name: cpd_instance_namespace
          value: cpd-instance #[INPUT] instance namespace
        - name: cpd_operators_namespace
          value: cpd-operator #[INPUT] operator namespace
        - name: block_storage_class
          value: managed-nfs-storage #[INPUT] block storage class
        - name: file_storage_class
          value: managed-nfs-storage #[INPUT] file storage class
        - name: pre_sync_job_image
          value: icr.io/cpopen/cpd/olm-utils-v3:5.0.4
        - name: components[0] #[INPUT] list all components to install/upgrade here
          value: cpd_platform
        - name: components[1]
          value: watsonx_ai
        # - name: cluster_name #[INPUT] If ArgoCD is not running on the target cluster, uncomment and put the cluster name (as defined in ArgoCD) here
        #   value: <cluster name>
  project: default
  syncPolicy:
    retry:
      limit: 10
      backoff:
        duration: 10s
        factor: 2
        maxDuration: 3m0s
```


5. Trigger a synchronize operation on the cpd-root app. Log into your ArgoCD web console and click on `Synchronize` button for `cpd-root` app. Alternatively, argocd's cli tool can also be used (https://argo-cd.readthedocs.io/en/stable/getting_started/#syncing-via-cli)
