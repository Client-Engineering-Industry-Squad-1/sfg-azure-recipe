Once Lightwell Framework is installed:
    - Edit the `admin` account and grant it with the "APIUser" permission to be able to log in to B2Bi Customization > Customization view.
        Accounts > User Accounts and search for "admin".
        Grant it the "APIUser" permission and save.
    - Log in to Customization > Customization view
        - Click on CustomJar tab and click "Create CustomJar" button
            - Vendor Name: LW
            - Vendor Version: 1.0
            - Jar Type: Library
            - File: LWUtility.jar
            - Target Path: DCL
            - Click on "Save CustomJar" button
        - Click on CustomService tab and click "Create CustomService" button
            - Service Name: LW-RuleService
            - File: LwRuleService.jar
            - Click on "Save CustomService" button
        - Click on CustomService tab and click "Create CustomService" button
            - Service Name: LW-UtilityService
            - File: LwUtilityService.jar
            - Click on "Save CustomService" button
        - Click on PropertyFile tab and click "Create PropertyFile" button
            - PropertyFile Prefix: customer_LW
            - Property File: customer_LW.properties
            - Click on "Save PropertyFile" button
        - Update customer_overrides.properties for LW deployment (ie. set DB configurations)
        - Click on PropertyFile tab and edit "customer_overrides" PropertyFile
            - Click on "customer_overrides" and select "General" tab
            - Click on "Edit" button
            - Property File: customer_overrides.properties
            - Click on "Replace Existing Property File" checkbox
            - Click on "Save PropertyFile" button
        - Edit "customer_LW" Property File
            - Click on "customer_LW" Property File and select "Property" tab
            - Modify the following properties:
                Property: DefaultEmail
                Property Value: <Email>
                Property: DefaultEmailSender
                Property Value: <Email>
                Property: ErrorAckEmail
                Property Value: <Email>
                Property: OverdueAckEmail
                Property Value: <Email>
                Property: PortalAuditEmail
                Property Value:: <Email>
                Property: DatabaseStorage
                Property Value: true
                Property: TempDirectory
                Property Value: /files
                Property: ArchiveOlderThanDays
                Property Value: <Days before archive>
                Property: ArchiveRootDirectory
                Property Value: /files/archive
    - Log in to B2Bi Console
        - Deplooyment > Resource Manager > Import/Export
        - Select "Import"
            File Name: EnvelopeExport.xml
            Passphrase: password
            Import All Resources: Select checkbox
            Click Next x3, Finish
        - Select "Import"
            File Name: FrameworkExport.xml
            Passphrase: password
            Import All Resources: Select checkbox
            Click Next x3, Finish
            Issue: File too big so had to restart the application for customer_overrides to take effect
                From B2Bi console, Operations > System > Troubleshooter > Stop the System
                From the `asi` pod, go to the terminal window
                    cd ibm/b2bi/install/bin
                    ./hardstop.sh
                    ./run.sh
                    Note: This will install the CustomJar and CustomServices along with the customer_overrides
                    Issue: Stopping the B2Bi service will cause the STS probes to restart the pod as its failing the check
                        Increase the configuration of the probes or remove as a workaround
        NOTE: FrameworkExportSPE.xml is only used if ITXA is installed
    - Log in to B2Bi Console
        - Deployment > Services > Configuration
        - Search SMTP
            - Edit LW_SA_SMTP as needed
            - Click Save
        - Search Portal
            - Edit LW_SA_HTTP_S1N1LOCAL_PORTAL
                HTTP Listen Port: 5580
            - Click Save
        - Search b2bi
            - Edit LW_SA_LWJDBC_B2BI
                Pool Name: db2Pool
            - Click Save


Validation:
    - Log in to Lightwell Portal and check Framework Version
    - Framework Management > Test Flow
        Protocol: PROXY
        User ID: admin
        File: Partner1_8850.edi
        Click Submit
    - Framework Management > Rules > Route Rules
        Click New and set the following:
            Document Type: 850
            Click Save
    - Framework Management > Rules > Send Rules
        Click New and set the following:
            Document Type: 850
            Send BP: LW_BP_SEND_EMAIL
            Subject Mask: 850
            Email Recipient: <Email>
            Click Save
    - Document Visibility > Submit File to B2Bi
        Protocol: PROXY
        User ID: admin
        File: Partner1_850.edi
        Click Submit
        Click "Show Documents"
