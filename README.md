csi-filestore-test
==================

This is just a little exercise to test the CSI Filestore driver.

1. Create a Filestore instance for each mountpoint you want (currently:
   `/home`, `/project`, `/scratch`).  We'll Terraform this later.  I
   called each one `csi-<name>` with share of `csi_<name>`.
   
2. Capture the IP address for each filestore.

3. Edit [csi-pvs.yaml](csi-pvs.yaml), replacing volumeHandle and
   volumeAttributes with the correct values for your instance.
   
4. Edit [csi-pod.yaml](csi-pod.yaml), replacing the NFS share details
   for the old filestore as necessary.
   
5. In order, create the storage class, the PVs, the PVCs, and then the
   Pod.
   
6. `kubectl exec` into the pod, copy files to their new destinations, or
   whatever else.
