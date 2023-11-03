# Mattermost Kubernetes Azure setup

This guide shows you how to setup Mattermost deployed via Kubernetes and all the necessary resources.

This guide **does not** show you how to properly lock down your Azure tenant.

## Prereqs

- Have an Azure account
- Have a domain name that can be used.
- Have `helm` installed on the device you will be running `kubectl` from.

    **for MacOS**

    ```bash
    brew install helm
    ```

- Have `yq` installed

    **for MacOS**

    ```bash
    brew install yq
    ```

- Have the minio kube plugin installed

    Docs: https://min.io/docs/minio/kubernetes/upstream/reference/kubectl-minio-plugin.html#id2

    ```bash
    brew install krew
    ```

- Have `mc` installed

    Docs: https://min.io/docs/minio/linux/reference/minio-mc.html?ref=docs

    **macos**

    ```bash
    brew install minio/stable/mc
    ```

## Setup

### Create AWS initial Services

1. Download the azure CLI for your OS

    - [install azure cli](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)

    **macOS:**

    ```bash
    brew update && brew install azure-cli
    ```
  
2. Authenticate the CLI

    This will open a new window to log into your Microsoft account.

    ```bash
    az login
    ```

    If you need a specific tenant you can do the below:

    ```bash
    az login --tenant yourTenant.com    
    ```

3. Create an azure resource group.

    Replace `AZURE_RESOURCE_GROUP` with a group name you choose.

    ```bash
    az group create --name AZURE_RESOURCE_GROUP --location eastus
    ```

4. Create a PostgreSQL database and create a related yaml file. 

    1. Create the database in Azure

        **Note: You must use a `flexible-server` because `single-server` does not included Postgres > 11.**

        - `POSTGRES_SERVER_NAME` : Replace with a name for your database
        - `AZURE_RESOURCE_GROUP`: The same resource group you created above.
        - `database-name`: The name of the database to be created when provisioning the database server. Leaving this as `mattermost` is probably best.
        - `admin-user`: PostgreSQL admin user
        - `admin-password`: password for the PostgreSQL admin user.
        - `tier`: The [Azure database tier](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes) that has the `sku-name` needed.
        - `sku-name`: The sku that has the resources you want in the tier you're in. You can find the descriptions on the above link under the `tier` you pick.
        - `storage-size` : The storage capacity of the server. Minimum is 32 GiB and max is 16 TiB.  Default: 128.
        - `version`: PostgreSQL version of the server.
        - `public-access` : Determines the public access. Enter single or range of IP addresses to be included in the allowed list of IPs. IP address ranges must be dash-separated and not contain any spaces. Specifying 0.0.0.0 allows public access from any resources deployed within Azure to access your server. Setting it to "None" sets the server in public access mode but does not create a firewall rule. 

        ```bash
        az postgres flexible-server create \
            --name POSTGRES_SERVER_NAME \
            --resource-group AZURE_RESOURCE_GROUP \
            --database-name mattermost \
            --location "East US" \
            --admin-user mmuser \
            --admin-password Testpassword123! \
            --tier MemoryOptimized \
            --sku-name Standard_E2ads_v5 \
            --storage-size 128 \
            --public-access 0.0.0.0 \
            --version 14
        ```

        **Example:**

        ```bash
        > az postgres flexible-server create \ 
            --name mattermost-postgres \ 
            --resource-group myResourceGroup \     
            --database-name mattermost \
            --location "East US" \
            --admin-user mmuser \
            --admin-password Testpassword123! \
            --tier MemoryOptimized \
            --sku-name Standard_E2ads_v5 \
            --storage-size 128 \
            --public-access 0.0.0.0 \
            --version 14
        Checking the existence of the resource group 'myResourceGroup'...
        Resource group 'myResourceGroup' exists ? : True 
        Creating PostgreSQL Server 'mattermost-postgres' in group 'myResourceGroup'...
        Your server 'mattermost-postgres' is using sku 'Standard_E2ads_v5' (Paid Tier). Please refer to https://aka.ms/postgres-pricing for pricing details
        Configuring server firewall rule, 'azure-access', to accept connections from all Azure resources...
        Creating PostgreSQL database 'mattermost'...
        Make a note of your password. If you forget, you would have to reset your password with "az postgres flexible-server update -n mattermost-postgres -g myResourceGroup -p <new-password>".
        Try using 'az postgres flexible-server connect' command to test out connection.
        {
            "connectionString": "postgresql://mmuser:Testpassword123!@mattermost-postgres.postgres.database.azure.com/mattermost?sslmode=require",
            "databaseName": "mattermost",
            "firewallName": "AllowAllAzureServicesAndResourcesWithinAzureIps_2023-11-3_12-13-50",
            "host": "mattermost-postgres.postgres.database.azure.com",
            "id": "/subscriptions/4c8a58d9-6291-4bcc-b7bd-4192a2d14fda/resourceGroups/myResourceGroup/providers/Microsoft.DBforPostgreSQL/flexibleServers/mattermost-postgres",
            "location": "East US",
            "password": "Testpassword123!",
            "resourceGroup": "myResourceGroup",
            "skuname": "Standard_E2ads_v5",
            "username": "mmuser",
            "version": "14"
        }
        ```

        **Take note of the `connectionString` value returned for later.**

    2. Take the connectionString from step 1, and convert it to base64.

        Note you need to switch `postgresql` from the string to `postgres` before converting.

        ```bash
        > echo -n 'postgres://mmuser:Testpassword123!@mattermost-postgres.postgres.database.azure.com/mattermost?sslmode=require' | base64
        cG9zdGdyZXM6Ly9tbXVzZXI6VGVzdHBhc3N3b3JkMTIzIUBtYXR0ZXJtb3N0LXBvc3RncmVzLnBvc3RncmVzLmRhdGFiYXNlLmF6dXJlLmNvbS9tYXR0ZXJtb3N0P3NzbG1vZGU9cmVxdWlyZQ==
        ```

    3. Edit the `mattermost-secret-postgres.yaml` file and replace the  `POSTGRES_BASE64_CONNECTION_STRING` with the value from step 2.

        Both `DB_CONNECTION_CHECK_URL` and `DB_CONNECTION_STRING` should match.

        It should now look like the below.

        ```yaml
        apiVersion: v1
        data:
            DB_CONNECTION_CHECK_URL: cG9zdGdyZXM6Ly9tbXVzZXI6VGVzdHBhc3N3b3JkMTIzIUBtYXR0ZXJtb3N0LXBvc3RncmVzLnBvc3RncmVzLmRhdGFiYXNlLmF6dXJlLmNvbS9tYXR0ZXJtb3N0P3NzbG1vZGU9cmVxdWlyZQ==
            DB_CONNECTION_STRING: cG9zdGdyZXM6Ly9tbXVzZXI6VGVzdHBhc3N3b3JkMTIzIUBtYXR0ZXJtb3N0LXBvc3RncmVzLnBvc3RncmVzLmRhdGFiYXNlLmF6dXJlLmNvbS9tYXR0ZXJtb3N0P3NzbG1vZGU9cmVxdWlyZQ==
        kind: Secret
        metadata:
            name: mattermost-postgres
        type: Opaque
        ```

    4. Save the file.

5. Create an AKS cluster.

    - `AZURE_RESOURCE_GROUP` - The resource group you created above
    - `AZURE_AKS_CLUSTER_NAME` - A name for your azure cluster. Something like `aks-mattermost` would work. 

    ```bash
    az aks create \
        -g AZURE_RESOURCE_GROUP \
        -n AZURE_AKS_CLUSTER_NAME \
        --enable-managed-identity \
        --node-count 3 \
        --generate-ssh-keys \
        --network-policy calico \
        --network-plugin kubenet \
        --enable-blob-driver
    ```

6. Install the AKS credentials to kubectl

    Replace `myResourceGroup` and `myAKSCluster` with the group and name you created above.

    ```bash
    az aks get-credentials --resource-group AZURE_RESOURCE_GROUP --admin --name AZURE_AKS_CLUSTER_NAME
    ```

    **Example:**

    ```bash
    > az aks get-credentials --resource-group mattermost-reources --admin --name aks-mattermost
    Merged "aks-mattermost-admin" as current context in /Users/test/.kube/config
    >       
    ```

### Deploy the Mattermost Operator and NGINX

1. Install the NGINX controller to the AKS cluster

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
    ```

2. Confirm the nginx operator is running and has an external IP

    ```bash
    kubectl get service -n ingress-nginx
    ```

    Example:

    ```bash
    > kubectl get service -n ingress-nginx 
    NAME                                 TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
    ingress-nginx-controller             LoadBalancer   10.0.205.237   4.156.190.182   80:31900/TCP,443:32089/TCP   2m20s
    ingress-nginx-controller-admission   ClusterIP      10.0.14.165    <none>          443/TCP                      2m20s
    >           
    ```

    Note: However you handle DNS, now is a good time to point your A record towards this external IP.

3. Install the Mattermost Operator

    ```bash
    kubectl create ns mattermost-operator
    kubectl apply -n mattermost-operator -f https://raw.githubusercontent.com/mattermost/mattermost-operator/master/docs/mattermost-operator/mattermost-operator.yaml
    ```

    Check that the operator pod is running with `kubectl get pod -n mattermost-operator`.

3. Create a namespace for mattermost services.

    This is not used yet, will be used for minio and mattermost later.

    ```bash
    kubectl create namespace MATTERMOST_NAMESPACE
    ```

    ```bash
    > kubectl create namespace mattermost
    namespace/mattermost created
    ```

### Deploy the Minio Operator

This is an abridged version of the helm install guide - https://min.io/docs/minio/kubernetes/upstream/operations/install-deploy-manage/deploy-operator-helm.html
For detailed questions, reference the guide.

1. Download the minio helm operator and tenant files locally.

    ```bash
    curl -O https://raw.githubusercontent.com/minio/operator/master/helm-releases/operator-5.0.10.tgz
    curl -O https://raw.githubusercontent.com/minio/operator/master/helm-releases/tenant-5.0.10.tgz
    ```

2. Deploy the minio operator

    Note: You can modify the `--namespace minio-operator`, it could cause confusion in the future though. So, not recommended.

    ```bash
    helm install \
        --namespace minio-operator \
        --create-namespace \
        minio-operator operator-5.0.10.tgz
    ```

    **Example**:

    ```bash
    > helm install \  
        --namespace minio-operator \
        --create-namespace \
        minio-operator operator-5.0.10.tgz
    NAME: minio-operator
    LAST DEPLOYED: Fri Nov  3 12:54:29 2023
    NAMESPACE: minio-operator
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    NOTES:
    1. Get the JWT for logging in to the console:
    kubectl apply -f - <<EOF
    apiVersion: v1
    kind: Secret
    metadata:
    name: console-sa-secret
    namespace: minio-operator
    annotations:
        kubernetes.io/service-account.name: console-sa
    type: kubernetes.io/service-account-token
    EOF
    kubectl -n minio-operator  get secret console-sa-secret -o jsonpath="{.data.token}" | base64 --decode
    ```

3. Configure the operator

    1. Create the service.yaml

        Replace PORT_NUMBER with the port on which to serve the Operator GUI. Something like `30080` would be fine.
        The range of valid ports is 30000-32767

        ```bash
        kubectl get service console -n minio-operator -o yaml > minio-service.yaml
        yq e -i '.spec.type="ClusterIP"' minio-service.yaml
        yq e -i '.spec.ports[0].nodePort = PORT_NUMBER' minio-service.yaml
        ```

    2. Create the operator.yaml file

        ```bash
        kubectl get deployment minio-operator -n minio-operator -o yaml > minio-operator.yaml
        yq -i -e '.spec.replicas |= 1' minio-operator.yaml
        ```

4. Apply the new file changes to the minio operator

    Note: the `minio-console-secret.yaml` file has already been created for you in this repo. Just copy it or use the exiting one. 

    ```bash
    kubectl apply -f minio-service.yaml
    kubectl apply -f minio-operator.yaml
    kubectl apply -f minio-console-secret.yaml
    ```

5. Confirm everything is running

    ```bash
    kubectl get all --namespace minio-operator   
    ```

    **Example:**

    ```bash
    > kubectl get all --namespace minio-operator        

        NAME                                  READY   STATUS    RESTARTS   AGE
    pod/console-7f8686864-wj56j           1/1     Running   0          2m29s
    pod/minio-operator-5d77754785-mrgrn   1/1     Running   0          2m29s

    NAME               TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                         AGE
    service/console    NodePort    10.0.157.180   <none>        9090:30080/TCP,9443:31270/TCP   2m29s
    service/operator   ClusterIP   10.0.206.217   <none>        4221/TCP                        2m29s
    service/sts        ClusterIP   10.0.165.240   <none>        4223/TCP                        2m29s

    NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/console          1/1     1            1           2m29s
    deployment.apps/minio-operator   1/1     1            1           2m29s

    NAME                                        DESIRED   CURRENT   READY   AGE
    replicaset.apps/console-7f8686864           1         1         1       2m29s
    replicaset.apps/minio-operator-5d77754785   1         1         1       2m29s
    ```

6. (Optional) You can connect to the console via the UI and confirm it's up and running.

    1. Get the JWT Token

        ```bash
        SA_TOKEN=$(kubectl -n minio-operator  get secret console-sa-secret -o jsonpath="{.data.token}" | base64 --decode)
        echo $SA_TOKEN
        ```

    2. Forward the Operator console port to allow access

        ```bash
        kubectl --namespace minio-operator port-forward svc/console 9090:9090
        ```

    3. Access the console via `localhost:9090` and provide the JWT token.

### Create the Minio Tenant and Mattermost Bucket

**Make sure you have the minio plugin installed for kubectl. See prereqs for more details**.

1. Create the tenant

    Official Docs: https://min.io/docs/minio/kubernetes/upstream/operations/install-deploy-manage/deploy-minio-tenant.html#deploy-a-minio-tenant-using-the-command-line

    [Docs on all these values](https://min.io/docs/minio/kubernetes/upstream/reference/kubectl-minio-plugin/kubectl-minio-tenant-create.html#kubectl.minio.tenant.create.-capacity)

    - `--servers`: Number of minio servers to deploy. Cannot exceed the number of nodes in the kube cluster.
    - `--capacity`: Total capacity of the tenant. This should match your azure blob max. 
    - `--volumes` : Number of volumes per server, mounted as separate drives. 
    - `--storage-class`: Should match the same storage class defined in your azure blob.
    - `TENANT_NAME` - The name for your Minio tenant for Mattermost. I use `minio-mattermost`
    - `--disable-tls` - This allows minio and Mattermost to communicate over `http` on the local ports. We are not exposing minio to any external traffic.
    - `--namespace` - This is the namespace you created for Mattermost services. 

    ```bash
    kubectl minio tenant create  \
        TENANT_NAME \
        --disable-tls \
        --capacity 200Gi \
        --servers 2 \
        --volumes 4 \
        --namespace MATTERMOST_NAMESPACE \
        --storage-class azureblob-nfs-premium
    ```

    Note: Copy / store the credentials that it outputs. If you ever lose these, they can be found again in the minio operator console.

    **Example**:

    ```bash
    > kubectl minio tenant create  \                                                                                                                                                    node system at kube aks-mattermost-admin
        minio-mattermost \
        --disable-tls \
        --capacity 200Gi \
        --servers 2 \
        --volumes 4 \
        --namespace mattermost \          
        --storage-class azureblob-nfs-premium
    W1103 13:07:45.459691   53687 warnings.go:70] unknown field "spec.pools[0].volumeClaimTemplate.metadata.creationTimestamp"

    Tenant 'minio-mattermost' created in 'mattermost' Namespace

    Username: EJ3Y0K7EF062AYNS317A 
    Password: VodEt4LDDUnzbcWJA7BKTrp4EFeu6EC37KTe0Rsp 
    Note: Copy the credentials to a secure location. MinIO will not display these again.

    APPLICATION     SERVICE NAME                    NAMESPACE       SERVICE TYPE    SERVICE PORT 
    MinIO           minio                           mattermost      ClusterIP       80          
    Console         minio-mattermost-console        mattermost      ClusterIP       9090        
    > 
    ```

2. Forward local port `9000` into the minio service on port `80`

    ```bash
    kubectl -n MATTERMOST_NAMESPACE port-forward svc/minio 9000:80
    ```

    **Example:** 

    ```bash
    > kubectl -n mattermost port-forward svc/minio 9000:80
    Forwarding from 127.0.0.1:9000 -> 9000
    Forwarding from [::1]:9000 -> 9000
    ```

3. Create the `mc` connection to your minio pod, create the bucket, and add a service account.

    - `TENANT_NAME` - Same tenant name as above.
    - `MINIO_TENANT_USERNAME` - The username that was output at step 1 when creating the tenant
    - `MINIO_TENANT_PASSWORD` - The password that was output at step 1 when creating the tenant.

    ```bash
    mc config host add TENANT_NAME http://localhost:9000 MINIO_TENANT_USERNAME MINIO_TENANT_PASSWORD
    mc mb TENANT_NAME/mattermost
    mc admin user add TENANT_NAME SERVICE_USERNAME SERVICE_PASSWORD
    ```

    **Example:**

    ```bash
    > mc config host add minio-mattermost http://localhost:9000 EJ3Y0K7EF062AYNS317A VodEt4LDDUnzbcWJA7BKTrp4EFeu6EC37KTe0Rsp
    Added `minio-mattermost` successfully.

    > mc mb minio-mattermost/mattermost 
    Bucket created successfully `minio-mattermost/mattermost`.
    
    > mc admin user add minio-mattermost mattermost SuperSecretPassword123!
    Added user `mattermost` successfully.
    ```

5. Convert the service username and password to a base 64 value to be used in step 6.

    ```bash
    echo -n 'SERVICE_USERNAME' | base64
    echo -n 'SERVICE_PASSWORD' | base64
    ```

    **Example:**

    ```bash
    > echo -n 'mattermost' | base64
    bWF0dGVybW9zdA==

    > echo -n 'SuperSecretPassword123!' | base64
    VGVzdFBhc3N3b3JkMTIzIQ==
    ```

6. Edit the `mattermost-secret-minio.yaml` file and replace `SERVICE_USERNAME` and `SERVICE_PASSWORD` with the above values.

    It should look roughly like this when done.

    ```yaml
    apiVersion: v1
    data:
        accesskey: bWF0dGVybW9zdA==
        secretkey: U3VwZXJTZWNyZXRQYXNzd29yZDEyMyE=
    kind: Secret
    metadata:
        name: mattermost-secret-minio
    type: Opaque
    ```

7. Create a policy for the `SERVICE_USERNAME` account.

    1. Modify the `minio-policy.json` file IF NEEDED and replace `mattermost` with your bucket name.

    2. Add the policy to minio via mc

        ```bash
        mc admin policy create TENANT_NAME mattermost-policy ./minio-policy.json 
        ```

        **Example:**

        ```bash
        > mc admin policy create minio-mattermost mattermost-policy ./minio-policy.json
        Created policy `mattermost-policy` successfully.
        ```

    3. Assign the policy to your `SERVICE_USERNAME`

        ```bash
        mc admin policy set TENANT_NAME mattermost-policy --user=SERVICE_USERNAME
        ```

        **Example:**

        ```bash
        > mc admin policy set minio-mattermost mattermost-policy --user=mattermost
        Created policy `mattermost-policy` successfully.
        ```
### Deploy the Mattermost Service

1. Update the `mattermost-secret-license.yaml` file with your license key.

    Replace `LICENSE_STRING` with the output from your mattermost license file.

2. Update the `mattermost-install.yaml` file with the values relevant to your deployment.

    The file itself has descriptions on every relevant value. If you've not deviated from this deployment, you may not have to update anything.

3. Apply the config file

    Note: `-n MATTERMOST_NAMESPACE` - This is the namespace you created above.

    ```bash
    kubectl apply -n MATTERMOST_NAMESPACE -f mattermost-secret-postgres.yaml
    kubectl apply -n MATTERMOST_NAMESPACE -f mattermost-secret-minio.yaml
    kubectl apply -n MATTERMOST_NAMESPACE -f mattermost-secret-license.yaml
    kubectl apply -n MATTERMOST_NAMESPACE -f mattermost-installation.yaml
    ```

4. Monitor until complete. This process takes 2-3 minutes.

    Note: `-n mattermost` - This is the namespace you created above.

    ```bash
    kubectl -n mattermost get mm -w
    ```

    If you need to debug anything you can use the below commands. Usually the error is in the operator events/logs or the mattermost pods. 

    **Common Commands**

    - `kubectl get pods -A` - Returns a list of the existing pods. You'll see `mattermost-service` pods that are bring create. 
    - `kubectl logs -n [namespace] [pod]` - View logs for a specific namespace / pod
    - `kubectl events -n [namespace] [pod]`-  view events for a specific namespace / pod

5. Access your Mattermost install via the domain name and confirm all is working.

    - If your domain is not up yet you can port forward to it. 