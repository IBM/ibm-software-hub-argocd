
## Prerequisites

1. Prepare an OpenShift cluster with proper storage classes required by CPD. Login to the cluster as a cluster admin
2. Install ArgoCD on your cluster.


## Procedure for Install/Upgrade
1. Run `oc adm policy add-cluster-role-to-user cluster-admin  system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n openshift-gitops --rolebinding-name=argocd-cluster-admin` to grant cluster admin access to ArgoCD application controller service account. This cluster role binding can be removed afterwards
2. Deploy a cluster-root app. Either use the ArgoCD webconsole, or oc apply the following app:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cluster-root-app
  namespace: openshift-gitops
spec:
  destination:
    name: ''
    namespace: default
    server: 'https://kubernetes.default.svc'
  source:
    path: cluster-root-app
    repoURL: 'https://github.com/IBM/ibm-software-hub-argocd'
    targetRevision: main
    helm:
      parameters: #overall overrides that go into every chile application and charts.
        #[INPUT] set namespace for each component; note that key cannot contains `_` so `ibm_cert_manager` and `ibm_licensing` are used here
        - name: scheduler.namespace
          value: my-scheduler-namespace
        - name: ibm_cert_manager.namespace #comment out this parameter if not installing ibm-cert-manager
          value: cert-manager-ns
        - name: ibm_licensing.namespace
          value: licensing-ns
        - name: cpd_release
          value: 5.0.0
        - name: components[0] #[INPUT] set components to install, include all dependencies 
          value: ibm-licensing
        - name: components[1]
          value: scheduler
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
3. Start sync-ing the cluster root app, either on the web console or using argocd cli tool `argocd app sync cluster-root-app`
