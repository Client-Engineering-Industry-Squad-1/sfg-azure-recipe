# Deploy [Sterling File Gateway](https://developer.ibm.com/components/sterling/tutorials/)

```
## Setting up environment variables for this tutorial
```bash
export GIT_ORG=#enter name for GitHub organization
```
```bash
echo $GIT_ORG #To validate that GIT_ORG has the correct value.
```
```bash
oc login --token=#token --server=#server
```

### Clone your GitOps repositories from your Github Organization 
```bash
cd ~
mkdir $GIT_ORG
cd $GIT_ORG
git clone https://github.com/$GIT_ORG/multi-tenancy-gitops.git
git clone https://github.com/$GIT_ORG/multi-tenancy-gitops-infra.git
git clone https://github.com/$GIT_ORG/multi-tenancy-gitops-services.git
git clone https://github.com/$GIT_ORG/multi-tenancy-gitops-apps.git
ls -l
```
#### Launch and login to ArgoCD
the Following command will provide ArgoCD URL
```bash
#ARGOCD username: admin
#ARGOCD URL:
oc get route -n openshift-gitops | grep openshift-gitops-cntk-server | awk '{print "https://"$2}'
#ARGOCD_PASSWORD: 
oc get secret/openshift-gitops-cntk-cluster -n openshift-gitops -o json | jq -r '.data."admin.password"' | base64 -D
```

### Infrastructure - Kustomization.yaml
1. Edit the Infrastructure layer `${GITOPS_PROFILE}/1-infra/kustomization.yaml`, un-comment the following lines, commit and push the changes and synchronize the `infra` Application in the ArgoCD console.

    ```bash        
    cd multi-tenancy-gitops/0-bootstrap/single-cluster/1-infra
    ```

    ```yaml
    - argocd/consolenotification.yaml
    - argocd/namespace-tools.yaml
    - argocd/namespace-sealed-secrets.yaml
    - argocd/serviceaccounts-tools.yaml
    ```

### Services - Kustomization.yaml

1. This recipe is can be implemented using a combination of storage classes. Not all combination will work, the following table lists the storage classes that we have tested to work:

    | Component | Access Mode | [Azure NetApp](https://mobb.ninja/docs/aro/trident/) | OCS/ODF |
    | --- | --- | --- | --- |
    | SQL | RWX | standard | ocs-storagecluster-cephfs |
    | MQ | RWX | standard | ocs-storagecluster-cephfs |
    | SFG | RWX | standard | ocs-storagecluster-cephfs |

1. Edit the Services layer `${GITOPS_PROFILE}/2-services/kustomization.yaml` and install Sealed Secrets by uncommenting the following line, **commit** and **push** the changes and refresh the `services` Application in the ArgoCD console.
    ```yaml
    - argocd/instances/sealed-secrets.yaml
    - argocd/operators/ibm-mq-operator.yaml
    ```

    >  💡 **NOTE**  
    > Commit and Push the changes for `multi-tenancy-gitops` & sync ArgoCD. 

1. Clone the services repo for GitOps, open a terminal window and clone the `multi-tenancy-gitops-services` repository under your Git Organization.
        
    ```bash
    git clone git@github.com:${GIT_ORG}/multi-tenancy-gitops-services.git
    ```
2. Modify the B2BI pre-requisites components which includes the secrets and PVCs required for the B2BI helm chart.

    1. Go to the directory:

        ```bash
        cd multi-tenancy-gitops-services/instances/ibm-sfg-b2bi-setup
        ```
    1. Generate the Sealed Secret for the credentials.
        ```bash
        B2B_DB_SECRET=dbadmin ./b2b-db-secret-secret.sh
        ```
        ```bash
        JMS_PASSWORD=passw0rd JMS_KEYSTORE_PASSWORD=passw0rd JMS_TRUSTSTORE_PASSWORD=passw0rd ./b2b-jms-secret.sh
        ```
        ```bash
        B2B_SYSTEM_PASSPHRASE_SECRET=password ./b2b-system-passphrase-secret.sh
        ```

    1. Generate Persistent Volume Yamls required by Sterling File Gateway (the default is set in RWX_STORAGECLASS environment variable to `managed-premium` - if you are installing on ODF, set `RWX_STORAGECLASS=ocs-storagecluster-cephfs`) if your using `azure-file`, set `RWX_STORAGECLASS=azure-file`

        - add (re-visit)
    >  💡 **NOTE**  
    > Commit and Push the changes for `multi-tenancy-gitops-services` 

1. Enable SQL, MQ and prerequisites in the main `multi-tenancy-gitops` repository

    1. Edit the Services layer `${GITOPS_PROFILE}/2-services/kustomization.yaml` by uncommenting the following lines to install the pre-requisites for Sterling File Gateway.
        ```yaml
        - argocd/instances/ibm-queuemanager-instance.yaml
        - argocd/instances/ibm-sfg-b2bi-setup.yaml
        ```

        >  💡 **NOTE**  
        > Push the changes & sync ArgoCD. 

1. Edit the Services layer `${GITOPS_PROFILE}/2-services/kustomization.yaml` by uncommenting the following line to install Sterling File Gateway, commit and push the changes and synchronize the `services` Application in the ArgoCD console:
   
1. Generate Helm Chart values.yaml for the Sterling File Gateway Helm Chart:
    ```bash
    cd multi-tenancy-gitops-services/instances/ibm-sfg-b2bi
    ./ibm-sfg-b2bi-overrides-values.sh
    ```
1. Edit the Services layer `${GITOPS_PROFILE}/2-services/kustomization.yaml` by uncommenting the following line to install Sterling File Gateway, **commit** and **push** the changes and synchronize the `services` Application in the ArgoCD console:

    ```yaml
    - argocd/instances/ibm-sfg-b2bi.yaml
    ```

    >  💡 **NOTE**  
    > Push the changes & sync ArgoCD this will take around 1.5 hr.
1. Deploying `SFTP` service support blob storage azure [click here](https://docs.microsoft.com/en-us/azure/storage/blobs/secure-file-transfer-protocol-support) for business processes.

    > Steps on b2bi side needs to be filled up with David ash.

1. Edit the Services layer `${GITOPS_PROFILE}/2-services/kustomization.yaml` by uncommenting the following line to install Lightwell, **commit** and **push** the changes and synchronize the `services` Application in the ArgoCD console:

    ```yaml
    - argocd/instances/lightwell-framework.yaml
    ```
### [Installation](./Post-install-lw.md) of Lightwell
    >  💡 **NOTE**  
    > Push the changes & sync ArgoCD.
---
> **⚠️** Warning:  
> If you decided to scale the pods or upgrade the verison you should do the following steps:
>> **This is to avoid going through the job again**

- Step 1:
    ```bash
    cd multi-tenancy-gitops-services/instances/ibm-sfg-b2bi
    ```
- Step 2:
  - Inside `values.yaml`, find & set 
  - ```bash
    datasetup:
        enable: false
    dbCreateSchema: false
    ```
- [USE-CASES](https://github.ibm.com/client-engineering-devops/general-admin/blob/master/tips/gitops/sterling/Scenarios.md)
___
### Validation

1.  Retrieve the Sterling File Gateway console URL.

    ```bash
    oc get route -n tools ibm-sfg-b2bi-sfg-asi-internal-route-filegateway -o template --template='https://{{.spec.host}}'
    ```

2. Log in with the default credentials:  username:`fg_sysadmin` password: `password` 

<!-- - running sqlscripts:
- For b2bi download create_scc_db_b2bidb.sql
- create dockerfile
- create script with the connection string 
- create a k8's job to pull the image and run the script
same for lightwell. 
- lightwell sql script order
    - MSSQL-PORTAL.sql
    - MSSQL-FW.sql
    - Portal-Seed.sql
    - FW-Seed.sql
- sftp support blob storage azure [click here](https://docs.microsoft.com/en-us/azure/storage/blobs/secure-file-transfer-protocol-support)
- TLS 1.2 network -->
