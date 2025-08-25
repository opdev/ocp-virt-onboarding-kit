# Configuring a Default Storage Class for OpenShift Virtualization

This tutorial provides a step-by-step guide on how to configure a default storage class for OpenShift Virtualization. Setting a default storage class ensures that virtual machines and data volumes automatically use the specified storage class when no storage class is explicitly specified, simplifying VM deployment and management.

## Prerequisites

To get started, you'll need the following:

- **OpenShift Cluster**: An OpenShift cluster with OpenShift Virtualization installed
- **Cluster Admin Access**: Access to the cluster with cluster-admin privileges
- **Storage Classes**: At least one StorageClass configured in your cluster
- **Storage Provisioner**: A storage provisioner that supports dynamic provisioning

Versions tested:
```
OCP 4.19
```

## Step 1: View Available Storage Classes and Check Current Default

First, check which storage classes are already available in your cluster and identify if there's already a default storage class configured. You can do this using the `oc` command-line tool.

```bash
oc get sc
```

This command will list all the storage classes, showing their names and provisioners. Look for any storage class that has `(default)` next to its name, which indicates it's currently set as the default.

Example output showing no default storage class:
```
NAME                    PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ceph-rbd-sc            rbd.csi.ceph.com           Delete          Immediate              true                   2d
hostpath-provisioner   kubevirt.io/hostpath-provisioner   Delete   WaitForFirstConsumer   false                  1d
local-storage          kubernetes.io/no-provisioner       Delete   WaitForFirstConsumer   false                  1d
```

Example output showing an existing default storage class:
```
NAME                    PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ceph-rbd-sc (default)  rbd.csi.ceph.com           Delete          Immediate              true                   2d
hostpath-provisioner   kubevirt.io/hostpath-provisioner   Delete   WaitForFirstConsumer   false                  1d
local-storage          kubernetes.io/no-provisioner       Delete   WaitForFirstConsumer   false                  1d
```

Take note of:
1. The name of the storage class you want to set as the default
2. Whether there's already a default storage class configured (you'll need to remove it first if you want to change it)

## Step 2: Remove an Existing Default (If Necessary)

If another storage class was already set as the default, you'll need to remove that annotation first before setting a new one. A cluster can only have one default storage class. To remove the default annotation from a storage class, use the following command:

```bash
oc annotate sc <the-old-default-storage-class-name> storageclass.kubernetes.io/is-default-class-
```

The hyphen at the end of the annotation key is what removes the annotation. After running this, you can then proceed with Step 3 to set the new default.

## Step 3: Annotate the Storage Class

Next, you need to add an annotation to the storage class you've chosen. This annotation tells OpenShift that this particular storage class should be the default one. The annotation is `storageclass.kubernetes.io/is-default-class: "true"`.

Use the `oc annotate` command to apply this. Replace `<your-storage-class-name>` with the name you noted in the previous step.

```bash
oc annotate sc <your-storage-class-name> storageclass.kubernetes.io/is-default-class="true"
```

For example, if your storage class is named `ceph-rbd-sc`, the command would be:

```bash
oc annotate sc ceph-rbd-sc storageclass.kubernetes.io/is-default-class="true"
```

## Step 4: Verify the Default Storage Class

After applying the annotation, you can verify that the change has taken effect. Run the `oc get sc` command again. You should see a new column, `DEFAULT`, and the storage class you just modified should have `(default)` next to its name.

```bash
oc get sc
```

Example output after setting the default:
```
NAME                    PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ceph-rbd-sc (default)  rbd.csi.ceph.com           Delete          Immediate              true                   2d
hostpath-provisioner   kubevirt.io/hostpath-provisioner   Delete   WaitForFirstConsumer   false                  1d
local-storage          kubernetes.io/no-provisioner       Delete   WaitForFirstConsumer   false                  1d
```

## Step 5: Test the Default Storage Class

Create a simple test to verify that the default storage class is working correctly. You can create a PVC without specifying a storage class to test this:

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-default-storage
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```

Then verify that the PVC was created with the correct storage class:

```bash
oc get pvc test-default-storage -o yaml | grep storageClassName
```

You should see that the PVC was automatically assigned to your default storage class.

## Step 6: Deploy a VM Using Default Storage

Now you can deploy a virtual machine that will automatically use the default storage class for its data volumes:

```bash
oc apply -f - <<EOF
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: test-vm-default-storage
  namespace: default
spec:
  running: true
  dataVolumeTemplates:
  - metadata:
      name: test-vm-disk
    spec:
      pvc:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
      sourceRef:
        kind: DataSource
        name: fedora
        namespace: openshift-virtualization-os-images
  template:
    metadata:
      labels:
        kubevirt.io/vm: test-vm-default-storage
    spec:
      domain:
        devices:
          disks:
          - name: datavolumedisk
            disk:
              bus: virtio
        resources:
          requests:
            memory: 1Gi
            cpu: 1
      networks:
      - name: default
        pod: {}
      volumes:
      - name: datavolumedisk
        dataVolume:
          name: test-vm-disk
EOF
```

## Troubleshooting

### Common Issues

1. **Multiple Default Storage Classes**: If you see multiple storage classes marked as default, remove the annotation from all but one:
   ```bash
   oc get sc -o custom-columns=NAME:.metadata.name,DEFAULT:.metadata.annotations.storageclass\.kubernetes\.io/is-default-class
   ```

2. **Storage Class Not Found**: Ensure the storage class exists before trying to annotate it:
   ```bash
   oc get sc <storage-class-name>
   ```

3. **Permission Denied**: Make sure you have cluster-admin privileges:
   ```bash
   oc auth can-i create storageclass --all-namespaces
   ```

### Verification Commands

- Check all storage classes and their default status:
  ```bash
  oc get sc -o wide
  ```

- View detailed information about a specific storage class:
  ```bash
  oc describe sc <storage-class-name>
  ```

- List PVCs and their storage classes:
  ```bash
  oc get pvc --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,STORAGECLASS:.spec.storageClassName
  ```

## Best Practices

1. **Choose Appropriate Storage**: Select a storage class that provides the performance and features needed for your VMs (e.g., fast I/O, snapshots, encryption).

2. **Consider Workload Requirements**: Different workloads may require different storage characteristics. Consider using specific storage classes for critical workloads rather than relying on defaults.

3. **Monitor Storage Usage**: Regularly monitor storage usage and performance to ensure the default storage class meets your needs.

4. **Document Your Choice**: Document why you chose a particular storage class as the default for future reference and troubleshooting.

## References

### OpenShift Documentation
- [OpenShift - Storage Classes](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/storage/understanding-persistent-storage#persistent-storage-overview_storage-classes)
- [OpenShift Virtualization - Storage](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/virtual-machines/virt-storage-overview)
