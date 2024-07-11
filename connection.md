

# Connection Service

Connection Service is a service in IBM Fusion. It's used to setup secure connection between OpenShift Clusters. These Openshfit Cluster can be Hub&Spoke Clusters in Backup and Restore Service or DR clusters in Metro-DR and Regional-DR.

## Reconnecting OpenShift Container Platform cluster

The OpenShift® Container Platform cluster that is connected in a disaster recovery set up or a Backup & Restore service with Hub and Spoke can face disasters, leading to a temporary unusable state. After recovery, it can still display an Unhealthy or UnManaged status in remote clusters. In such instances, this cluster must be reconnected to its corresponding connections.

### About this task

If the cluster recovers before the expiration time, the cluster rejoins the connection automatically and no action is needed. However, if the cluster recovers after the expiration of the client cert, the connection must be cleaned and setup to rejoin the recovered cluster.

### Procedure

1. Save the name of connection CR in cluster-a (a cluster in hub&spoke or DR Clusters), it would be like connection-<id>

2. Clean the connection. For the procedure to clean the connection, see [Disabling the connection](#disabling-the-connection)

3. Setup the connection between clusters.
   
    a. Get the bootstrap token in cluster-b, the version of oc command line should be greater or equal 4.11.
    ```
    oc create token isf-application-operator-cluster-bootstrap -n <Fusion Namespace of cluster-b>
    ```
  
    b. Create init secret in cluster-a with the bootstrap token and API endpoint of cluster-b. For example:
    ```
    apiVersion: v1
    kind: Secret
    metadata:
       name: <Init Secret Name>
       namespace: <Fusion Namespace of cluster-a>
    stringData:
       apiserver: <cluster-b API Endpoint>
       bootstrapToken: <cluster-b Token Generated in Step 3.a>
    ```

    c. Create connection CR in cluster-a with this init secret in spec:
    ```
    apiVersion: application.isf.ibm.com/v1
    kind: Connection
    metadata:
      name: <Connection Name Saved in Step 1>
      namespace: <Fusion Namespace of cluster-a>
    spec:
      remoteCluster:
        apiEndpoint: <cluster-b API Endpoint>
        initSecretName: <Init Secret Name>
        connectionOperatorNamespace: <Fusion Namespace in cluster-b>
    ```

## Disabling the connection

Disable the connection between Hub and Spoke.

### About this task

In Hub and Spoke, there is a connection CR in each cluster that represents the connection to the another(remote) cluster. Delete connection CR in the Spoke or Hub. If the network is down, then use either method 1 or method 2 to manually delete the connection CR in another cluster also.

### Method 1: Using command line or command prompt

1. In the Spoke, use the following command to get all the connection CR:
    ```
    oc get connection -n <fusion-namespace> -o=custom-columns='NAME:.metadata.name,ClusterName:.metadata.    annotations.connection\.isf\.ibm\.com/cluster-name'
    ```

    Example command:
    ```
    oc get connection -n ibm-spectrum-fusion-ns -o=custom-columns='NAME:.metadata.name,ClusterName:.metadata.    annotations.connection\.isf\.ibm\.com/cluster-name'
    ```
    
    Example output:
    ```
    NAME                    ClusterName
    connection-44ae8529aa   cp4d-vpc-98b7318c91b01bd72490e80cc2328915-0000.us-east.containers.appdomain.cloud
    ```

2. Check the ClusterName in connection CR, and delete the connection CR which has the same ClusterName with Hub:
    ```
    oc delete connection <connection-name>  -n <fusion-namespace>
    ```
    
    Example command:
    ```
    oc delete connection connection-44ae8529aa -n ibm-spectrum-fusion-ns
    ```
### Method 2: By using OpenShift Container Platform console 
1. Log in to OpenShift® Container Platform console.
2. Go to Administration > CustomResourceDefinitions > connections.application.isf.ibm.com > Instances.
3. Get the connection instance whose cluster name in the annotation is the same as the hub cluster name.
4. Delete this connection instance.

## Hub and spoke connection issues

Procedure to debug issue in the hub and spoke connections. Backup & Restore service uses connection CR to setup hub and spoke connection.

You might encounter an error when you attempt setup connections between clusters.

### Bootstrap token in init secret is not correct or expired: Unauthorized

#### Problem statement

Connection setup fails with the following message in the connection CR:

```

apiVersion: application.isf.ibm.com/v1
kind: Connection
metadata:
  name: <connection-name>
  namespace: <connection-namespace>
spec:
  remoteCluster:
    apiEndpoint: <cluster api endpoint>
    connectionOperatorNamespace: <connection-namespace>
    heartBeatInterval: 10m
    initSecretName: <init-secret-name>
status:
  conditions:
    - lastTransitionTime: '2023-06-15T02:31:01Z'
      message: 'Bootstrap token in init secret is not correct or expired: Unauthorized'
      reason: CreateBootstrapSecret
      status: 'False'
      type: BootstrapSecretAvaliable
  connectionFromRemoteClusterHealth:
    message: ''
    messageCode: ''
    messageType: ''
  connectionState: Failed
  connectionToRemoteClusterHealth:
    message: ''
    messageCode: ''
    messageType: ''
```

#### Cause

The bootstrap token in the init secret is not correct or expired.

#### Resolution

1. Get the bootstrap token again.
```
oc create token isf-application-operator-cluster-bootstrap -n <connection-namespace>
```

2. Replace the token in init secret:
```
oc edit secret <init-secret-name> -n <connection-namespace>
```

### CA certificate of peer cluster is not correct


#### Problem statement

The CA certificate of peer cluster is not correct error occurs in connection CR.

#### Cause

The CAcert in the configmap kube-root-ca.crt and namespace kube-public of the remote cluster is not correct.


#### Resolution

In the remote cluster, place the right CAcert in the configmap kube-root-ca.crt and namespace kube-public. Connection pkg also provides a customized configmap.

If it is not possible to place the right CAcert in configmap kube-root-ca.crt and namespace kube-public, then place the right CAcert in custom-ca.crt and Fusion namespace:

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: custom-ca.crt
  namespace: <connection-namespace>
data:
  ca.crt: <right CAcert>
```