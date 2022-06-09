# Deploy [Sterling File Gateway](https://developer.ibm.com/components/sterling/tutorials/)


### Infrastructure - Kustomization.yaml
1. Edit the Infrastructure layer `${GITOPS_PROFILE}/1-infra/kustomization.yaml`, un-comment the following lines, commit and push the changes and synchronize the `infra` Application in the ArgoCD console.

    ```bash        
    cd multi-tenancy-gitops/0-bootstrap/single-cluster/1-infra
    ```

    ```yaml
    - argocd/consolenotification.yaml
    - argocd/namespace-db2.yaml
    - argocd/namespace-mq.yaml
    - argocd/namespace-tools.yaml
    - argocd/namespace-sealed-secrets.yaml
    - argocd/serviceaccounts-db2.yaml
    - argocd/serviceaccounts-mq.yaml
    - argocd/serviceaccounts-tools.yaml
    ```

### Services - Kustomization.yaml

1. This recipe is can be implemented using a combination of storage classes. Not all combination will work, the following table lists the storage classes that we have tested to work:

    | Component | Access Mode | Azure Cloud | OCS/ODF |
    | --- | --- | --- | --- |
    | SQL | RWX | azure-file | ocs-storagecluster-cephfs |
    | MQ | RWX | azure-file | ocs-storagecluster-cephfs |
    | SFG | RWX | azure-file | ocs-storagecluster-cephfs |

1. Edit the Services layer `${GITOPS_PROFILE}/2-services/kustomization.yaml` and install Sealed Secrets by uncommenting the following line, **commit** and **push** the changes and refresh the `services` Application in the ArgoCD console.
    ```yaml
    - argocd/instances/sealed-secrets.yaml
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
        - argocd/instances/ibm-qm-instance.yaml
        - argocd/instances/ibm-sfg-b2bi-setup.yaml
        ```

>  💡 **NOTE**  
> Push the changes & sync ArgoCD. 

1. Edit the Services layer `${GITOPS_PROFILE}/2-services/kustomization.yaml` by uncommenting the following line to install Sterling File Gateway, commit and push the changes and synchronize the `services` Application in the ArgoCD console:
   
1. Generate Helm Chart values.yaml for the Sterling File Gateway Helm Chart:
    ```
    cd multi-tenancy-gitops-services/instances/ibm-sfg-b2bi
    ./ibm-sfg-b2bi-overrides-values.sh
    ```
1. Edit the Services layer `${GITOPS_PROFILE}/2-services/kustomization.yaml` by uncommenting the following line to install Sterling File Gateway, **commit** and **push** the changes and synchronize the `services` Application in the ArgoCD console:

    ```yaml
    - argocd/instances/ibm-sfg-b2bi.yaml
    ```

>  💡 **NOTE**  
> Push the changes & sync ArgoCD this will take around 1.5 hr.
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

- running sqlscripts:
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
- TLS 1.2 network
