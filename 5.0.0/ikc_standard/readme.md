# Deployment for ikc_standard

##### Pre Requisite
 - Need to install Openshift AI using [link](https://github.ibm.com/IBMPrivateCloud/ibm-watsonx-ai-ifm-bundle/wiki/Install-OpenShift-AI)
 

##### issues that could arise with db2u
 - `c-db2oltp-wkc` stuck in pending,  
  ```
    oc apply -f - <<EOF
    apiVersion: v1
    data:
    DB2U_RUN_WITH_LIMITED_PRIVS: "false"
    kind: ConfigMap
    metadata:
    name: db2u-product-cm
    namespace: <operator namespace>
    EOF
```
run cpd-cli manage apply-db2-kubelet this will require the use to download the cpdcli binary. we will create a workaround for this. 

-  WKC Db2U pods not provisioning. - follow instructions in this [link](https://www.ibm.com/support/pages/wkc-db2u-failed-provision-databases-bgdb-ilgdb-and-wfdb)
