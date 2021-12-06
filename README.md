csi-filestore-test
==================

Setup
------------------

Set up the underlying storage and StorageClass.

1. Create a Filestore instance for each mountpoint you want (currently:
   `/home`, `/project`, `/scratch`).  We'll Terraform this later.  I
   called each one `csi-<name>` with share of `csi_<name>`.
   
2. Capture the IP address for each filestore.

3. Edit [csi-pvs.yaml](csi-pvs.yaml) and
   [csi-concurrent-pvs.yaml](csi-concurrent-pvs.yaml) replacing
   volumeHandle and volumeAttributes with the correct values for your
   instance.
   
4. Create the `csi-test` namespace: `kubectl create ns csi-test`
   
5. Create the `csi-filestore` storageClass: `kubectl apply -f csi-sc.yaml`.


Basic functionality
-------------------

This is just a little exercise to test the CSI Filestore driver.

1. Edit [csi-pod.yaml](csi-pod.yaml), replacing the NFS share details
   for the old filestore as necessary.
   
2. Create the PVs, the PVCs, and then the Pod
```
kubectl apply -f csi-pvs.yaml
kubectl apply -f csi-pvcs.yaml	
kubectl apply -f csi-pod.yaml
```
   
3. `kubectl exec` into the pod `csi-basic`, copy files to their new
   destinations, or whatever else.

Concurrency
-----------

1. Delete the Basic functionality resources.  You shouldn't have to do
   this, but here we are.
```
kubectl delete -n csi-test pod csi-basic
kubectl delete -n csi-test pvc csi-home-pvc csi-project-pvc csi-scratch-pvc
kubectl delete pv csi-home-pv csi-project-pv csi-scratch-pv
```
   
2. Create the PVs for the concurrency test and see that they are created and
   in `Available` state:
```
kubectl apply -f csi-concurrent-pvs.yaml
kubectl get pv
```

3. Create the PVCs for the concurrency test and see that they are created and
   in `Bound` state, and that the PVs are now `Bound` as well:
```
kubectl apply -f csi-concurrent-pvcs.yaml
kubectl get -n csi-test pvc
kubectl get pv
```

4. Create the first pod for the concurrency test.  Be amazed that it does not start.
```
kubectl apply -f csi-concurrent1-pod.yaml
sleep 200
kubectl get -n csi-test pod csi-concurrent1
```

The error will look something like

```
  Warning  FailedMount  4s               kubelet            MountVolume.MountDevice failed for volume "csi-concurrent-scratch-pv1" : rpc error: code = DeadlineExceeded desc = context deadline exceeded
  Warning  FailedMount  2s (x2 over 3s)  kubelet            MountVolume.MountDevice failed for volume "csi-concurrent-scratch-pv1" : rpc error: code = Aborted desc = An operation with the given volume key modeInstance/us-central1-b/csi-scratch/csi_scratch already exists
```

It appears that the mere existence of multiple PV/PVC pairs pointing to
the same underlying filestore will prevent the CSI driver from mounting
any of those PVCs to any pod.

5. Attempt to create the second pod and see what happens.  It too is going to get
   stuck in `ContainerCreating`:
```
kubectl apply -f csi-concurrent2-pod.yaml
kubectl get pod -n csi-test csi-concurrent2
```
   
6. Weep bitter tears.
