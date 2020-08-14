

# NFS Server

for the NFS server we will need to NFS utils packages , setup the required directories and files and make sure the firewalld + SElinux are configured correctly.
This tutorial already assume that you have already pre installed the OS so I will not go into that in this tutorial.

## NFS Packages

First we will install the necessary packages for which we require :

```
$ yum install -y nfs-utils policycoreutils-python-utils policycoreutils-python
```

now for the NFS configuration we will create a new directory

```
$ mkdir /var/nfsshare
```

## Optional

Create an LVM device and mount it into the new directory :

```
$ lvcreate -L50G -n nfs vg00
$ mkfs.xfs /dev/vg00/nfs
$ mount -t xfs /dev/vg00/nfs /var/nfsshare
```

and add it to your fstab (or autofs) file to make sure it is consistent at boot time :

```
$ cat >> /etc/fstab << EOF
/dev/vg00/nfs /var/nfsshare   xfs  defaults 0 0
EOF
```

## Continue…

Now we to make sure everybody has write permissions on the directory (I know this is bad security wise) so we will change it to 777

```
$ chmod 777 /var/nfsshare
```

Edit the /etc/exports file to publish the nfs share directory

```
$ cat > /etc/exports << EOF
/var/nfsshare     *(rw,sync,no_wdelay,root_squash,insecure,fsid=0)
EOF
```

Start the services :

```
$ systemctl start rpcbind
$ systemctl start nfs-server
$ systemctl start nfs-lock
$ systemctl start nfs-idmap
```

Make sure they run on boot time :

```
$ systemctl enable rpcbind
$ systemctl enable nfs-server
$ systemctl enable nfs-lock
$ systemctl enable nfs-idmap
```

## Firewall

For the firewalld we need to make sure the following services are open :

```
$ export FIREWALLD_DEFAULT_ZONE=`firewall-cmd --get-default-zone`
$ echo ${FIREWALLD_DEFAULT_ZONE}
public
$ firewall-cmd --permanent --zone=${FIREWALLD_DEFAULT_ZONE} --add-service=rpc-bind
$ firewall-cmd --permanent --zone=${FIREWALLD_DEFAULT_ZONE} --add-service=nfs
$ firewall-cmd --permanent --zone=${FIREWALLD_DEFAULT_ZONE} --add-service=mountd
```

Now we will reload the firewall configuration :

```
$ firewall-cmd --reload
$ firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: ens192
  sources: 
  services: dhcpv6-client dns ftp http https kerberos ldap ldaps mountd nfs ntp ssh tftp
  ports: 80/tcp 443/tcp 389/tcp 636/tcp 53/tcp 88/udp 464/udp 53/udp 123/udp 88/tcp 464/tcp 123/tcp 9000/tcp 8000/tcp 8080/tcp 111/tcp 54302/tcp 20048/tcp 2049/tcp 46666/tcp 42955/tcp 875/tcp
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules:
```

## SElinux

for the SElinux part we need to set two (2) boolean roles and set the nfs directory with semanage:

```
$ setsebool -P nfs_export_all_rw 1
$ setsebool -P nfs_export_all_ro 1
```

And for the Directory content switch :

```
$ semanage fcontext -a -t public_content_rw_t  "/var/nfsshare(/.*)?"
$ restorecon -R /var/nfsshare
```

## Testing

On another machine , make sure you have “nfs-utils” installed and mount the directory from the server :

```
$ mount -t nfs nfs-server:/var/nfsshare /mnt
$ touch /mnt/1 && rm -f /mnt/1
```

If the command was successful and you can create a file and delete it , we are good to go


# Kubernetes NFS-Client Provisioner

[![Docker Repository on Quay](https://quay.io/repository/external_storage/nfs-client-provisioner/status "Docker Repository on Quay")](https://quay.io/repository/external_storage/nfs-client-provisioner)

**nfs-client** is an automatic provisioner that use your *existing and already configured* NFS server to support dynamic provisioning of Kubernetes Persistent Volumes via Persistent Volume Claims. Persistent volumes are provisioned as ``${namespace}-${pvcName}-${pvName}``.

# How to deploy nfs-client to your cluster.

To note again, you must *already* have an NFS Server.

## With Helm

Follow the instructions for the stable helm chart maintained at https://github.com/helm/charts/tree/master/stable/nfs-client-provisioner

The tl;dr is

```bash
$ helm install stable/nfs-client-provisioner --set nfs.server=x.x.x.x --set nfs.path=/exported/path
```

## Without Helm

**Step 1: Get connection information for your NFS server**. Make sure your NFS server is accessible from your Kubernetes cluster and get the information you need to connect to it. At a minimum you will need its hostname.

**Step 2: Get the NFS-Client Provisioner files**. To setup the provisioner you will download a set of YAML files, edit them to add your NFS server's connection information and then apply each with the ``kubectl`` / ``oc`` command. 

Get all of the files in the [deploy](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client/deploy) directory of this repository. These instructions assume that you have cloned the [external-storage](https://github.com/kubernetes-incubator/external-storage) repository and have a bash-shell open in the ``nfs-client`` directory.

**Step 3: Setup authorization**. If your cluster has RBAC enabled or you are running OpenShift you must authorize the provisioner. If you are in a namespace/project other than "default" edit `deploy/rbac.yaml`.

Kubernetes:

```sh
# Set the subject of the RBAC objects to the current namespace where the provisioner is being deployed
$ NS=$(kubectl config get-contexts|grep -e "^\*" |awk '{print $5}')
$ NAMESPACE=${NS:-default}
$ sed -i'' "s/namespace:.*/namespace: $NAMESPACE/g" ./deploy/rbac.yaml ./deploy/deployment.yaml
$ kubectl create -f deploy/rbac.yaml
```

OpenShift:

On some installations of OpenShift the default admin user does not have cluster-admin permissions. If these commands fail refer to the OpenShift documentation for **User and Role Management** or contact your OpenShift provider to help you grant the right permissions to your admin user. 

```sh
# Set the subject of the RBAC objects to the current namespace where the provisioner is being deployed
$ NAMESPACE=`oc project -q`
$ sed -i'' "s/namespace:.*/namespace: $NAMESPACE/g" ./deploy/rbac.yaml
$ oc create -f deploy/rbac.yaml
$ oadm policy add-scc-to-user hostmount-anyuid system:serviceaccount:$NAMESPACE:nfs-client-provisioner
```

**Step 4: Configure the NFS-Client provisioner**

Note: To deploy to an ARM-based environment, use: `deploy/deployment-arm.yaml` instead, otherwise use `deploy/deployment.yaml`.

Next you must edit the provisioner's deployment file to add connection information for your NFS server. Edit `deploy/deployment.yaml` and replace the two occurences of <YOUR NFS SERVER HOSTNAME> with your server's hostname.

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: <YOUR NFS SERVER HOSTNAME>
            - name: NFS_PATH
              value: /var/nfs
      volumes:
        - name: nfs-client-root
          nfs:
            server: <YOUR NFS SERVER HOSTNAME>
            path: /var/nfs
```

You may also want to change the PROVISIONER_NAME above from ``fuseim.pri/ifs`` to something more descriptive like ``nfs-storage``, but if you do remember to also change the PROVISIONER_NAME in the storage class definition below:

This is `deploy/class.yaml` which defines the NFS-Client's Kubernetes Storage Class:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
provisioner: fuseim.pri/ifs # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false" # When set to "false" your PVs will not be archived
                           # by the provisioner upon deletion of the PVC.
```

**Step 5: Finally, test your environment!**

Now we'll test your NFS provisioner.

Deploy:

```sh
$ kubectl create -f deploy/test-claim.yaml -f deploy/test-pod.yaml
```

Now check your NFS Server for the file `SUCCESS`.

```sh
kubectl delete -f deploy/test-pod.yaml -f deploy/test-claim.yaml
```

Now check the folder has been deleted.

**Step 6: Deploying your own PersistentVolumeClaims**. To deploy your own PVC, make sure that you have the correct `storage-class` as indicated by your `deploy/class.yaml` file.

For example:

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```



1. To configure your registry to use storage, change the `spec.storage.pvc` in the `configs.imageregistry/cluster` resource.

   |      | When using shared storage such as NFS, it is strongly recommended to use the `supplementalGroups` strategy, which dictates the allowable supplemental groups for the Security Context, rather than the `fsGroup` ID. Refer to the NFS **Group IDs** documentation for details. |


2. Verify you do not have a registry Pod:

   

   ```
   $ oc get pod -n openshift-image-registry
   ```

   |      | If the storage type is `emptyDIR`, the replica number cannot be greater than `1`.If the storage type is `NFS`, you must enable the `no_wdelay` and `root_squash` mount options. For example:`# cat /etc/exports`Example output`/mnt/data *(rw,sync,no_wdelay,root_squash,insecure,fsid=0)``sh-4.2# exportfs -rv`Example output`exporting *:/mnt/data` |


3. Check the registry configuration:

   

   ```
   $ oc edit configs.imageregistry.operator.openshift.io
   ```

   Example registry configuration

   

   ```
   storage:
     pvc:
       claim:
   ```

   Leave the `claim` field blank to allow the automatic creation of an `image-registry-storage` PVC.

4. Optional: Add a new storage class to a PV:

   1. Create the PV:

      

      ```
      $ oc create -f -
      ```

      

      ```
      apiVersion: v1
      kind: PersistentVolume
      metadata:
        name: image-registry-pv
      spec:
        accessModes:
          - ReadWriteMany
        capacity:
            storage: 100Gi
        nfs:
          path: /registry
          server: 172.16.231.181
        persistentVolumeReclaimPolicy: Retain
        storageClassName: nfs01
      ```

      

      ```
      $ oc get pv
      ```

   2. Create the PVC:

      

      ```
      $ oc create -n openshift-image-registry -f -
      ```

      

      ```
      apiVersion: "v1"
      kind: "PersistentVolumeClaim"
      metadata:
        name: "image-registry-pvc"
      spec:
        accessModes:
          - ReadWriteMany
        resources:
          requests:
            storage: 100Gi
        storageClassName: nfs01
        volumeMode: Filesystem
      ```

      

      ```
      $ oc get pvc -n openshift-image-registry
      ```

      Finally, add the name of your PVC:

      

      ```
      $ oc edit configs.imageregistry.operator.openshift.io -o yaml
      ```

      

      ```
      storage:
        pvc:
          claim: image-registry-pvc 
      ```

      |      | Creating a custom PVC allows you to leave the `claim` field blank for default automatic creation of an `image-registry-storage` PVC. |

5. Check the `clusteroperator` status:

   

   ```
   $ oc get clusteroperator image-registry
   ```

###### Configuring block registry storage for VMware vSphere

