# Db2 Log Archive and Backup to Azure Blob Storage on AKS

Db2 does not directly support Azure Blob Storage because it is not S3 compatible. However, you can use a program s3proxy to convert the S3 API calls to Azure Blob Storage API calls. Microsoft published a document describing the setup here:

https://devblogs.microsoft.com/cse/2016/05/22/access-azure-blob-storage-from-your-apps-using-s3-api/

You will need to use a customised version of s3proxy which has some fixes to allow it to work with Db2 here:

https://github.com/thaughbaer/s3proxy

There is a prebuilt container image which if desired you can push to a private registry here:

https://quay.io/repository/thaughbaer/s3proxy

You will need to know:
- The storage account name <accountmame>
- The storage account key <accountkey>
- The Db2u cluster name <clustername>

Create a new namespace for the s3proxy deployment:
```
kubectl create namespace s3proxy
```

Create a secret with your Azure Storage Account credentials:
```
kubectl create secret generic azure-<accountname>-secret --from-literal=azurestorageaccountname=<accountname> --from-literal=azurestorageaccountkey='<accountkey>'
```

Update the deployment yaml with:
- The storage account secret name ( multiple places )
- The endpoint for your storage account	
- If you are using a private registry the image

Create the deployment:
```
kubectl create -f s3proxy.Deployment.yml
```

Create the service:
```
kubectl create -f s3proxy.Service.yml
```

Db2 will only use a trusted HTTPS connection so you need to have a trusted certficate. One way to achieve this is to have the s3proxy pod create it's own Certificate Authority and issue it's own certificate and then add the CA public key to the Db2 pods. If no CA keys exist the pod will create them. The keys will then exist in /opt/s3proxy/certs and can be copied out of the pod and used to create the config artifacts:
```
kubectl get pods
kubectl cp s3proxy-65d5575fbc-94wzj:/opt/s3proxy/certs/rootCA.pem rootCA.pem
kubectl cp s3proxy-65d5575fbc-94wzj:/opt/s3proxy/certs/rootCA.key rootCA.key

kubectl create configmap s3proxy-rootca-crt --from-file=rootCA.pem=./rootCA.pem
kubectl create secret s3proxy-rootca-key --from-file=rootCA.key=./rootCA.key
```

The s3proxy-rootca-crt configmap will also need to be copied to the namespace that the Db2u pods are located.

In the Db2uCluster CRD add a volume source for the CA public key:
```
volumeSources:
- visibility:
  - db2u
  volumeSource:
    secret:
      secretName: s3proxy-rootca-crt
```

In the Db2 instance userprofile ( add the code to import the CA public key:
```
kubectl exec -it c-<clustername>-db2u-0
$ sudo su - db2inst1
$ vi ~/sqllib/userprofile
```

Add:
```
if [[ ! -f /db2u/.db2u_initialized ]]; then
    sudo cp /secrets/external/s3proxy-rootca-crt/rootCA.pem /etc/pki/ca-trust/source/anchors
    sudo update-ca-trust 
fi

```

Create the storage access alias in Db2:
```
kubectl exec -it c-<clustername>-db2u-0
sudo su - db2inst1
$ db2 "catalog storage access alias <accountname> vendor S3 server s3proxy.s3proxy.svc.cluster.local user <accountname> password <accountkey> container <clustername> dbuser db2inst1"

$ db2 "list storage access"

 Node Directory

Node 1 entry:

ALIAS=<accountname>
VENDOR=s3
SERVER=s3proxy.s3proxy.svc.cluster.local
USERID=<accountname>
CONTAINER=<clustername>
OBJECT=
DBUSER=db2inst1
DBGROUP=

 Number of entries in the directory = 1

DB20000I  The LIST STORAGE ACCESS command completed successfully.
```

Update the Db2 configuration to use S3 for log archiving:
```
db2 "update database configuration for BLUDB using LOGARCHMETH1 db2remote://<accountname>/<clustername>/archive_log/"
```

Run a backup to S3:
```
db2 "backup database BLUDB online to db2remote://dtaeunndevblbedw02//backup/"
```
