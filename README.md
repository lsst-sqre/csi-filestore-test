csi-filestore-test
==================

Setup
------------------

Set up the underlying storage and StorageClass.

1. Set your current working directory to the directory containing this
   [README.md](README.md).

2. Create a Filestore instance for each mountpoint you want (currently:
   `/home`, `/project`, `/scratch`).  We'll Terraform this later.  I
   called each one `csi-<name>` with share of `csi_<name>`.
   
3. Capture the IP address for each filestore.

4. Edit [csi-pvs.yaml](csi-pvs.yaml) and
   [csi-concurrent-pvs.yaml](csi-concurrent-pvs.yaml) replacing
   volumeHandle and volumeAttributes with the correct values for your
   instance.
   
5. Create the `csi-test` namespace: `kubectl create ns csi-test`
   
6. Create the `csi-filestore` storageClass: `kubectl apply -f csi-sc.yaml`.

These resources will not need to be deleted.  When we start using
resources that need cleaning up, we're going to name them all things
starting with `csi` so that we can clean them up with a simple shell
loop.  Obviously, pick something else if you have resources you want to
keep whose names start with `csi`.

Basic functionality
-------------------

This is just a little exercise to test the CSI Filestore driver.

1. Set your current working directory to the directory containing this
   [README.md](README.md).

2. Edit [csi-pod.yaml](csi-pod.yaml), replacing the NFS share details
   for the old filestore as necessary.
   
3. Create the PVs, the PVCs, and then the Pod
```
kubectl apply -f csi-pvs.yaml
kubectl apply -f csi-pvcs.yaml	
kubectl apply -f csi-basic-pod.yaml
```
   
4. `kubectl exec` into the pod `csi-basic-pod` and copy files to their new
   destinations, or whatever else.  For example:
```
for i in home project ; do time (cp -a /orig-share/$i/* /csi/$i/ ; sync); done
```
   This all works swimmingly.

Cleaning up resources
---------------------

Whenever you want to clear the resources, assuming that `csi-test`
contains all your namespaced resources, and that only your testing PVs'
names begin with `csi`:

```
for i in pod pvc pv ; do if [ "${i}" = "pv" ]; then \
    ns="" ; else ns=" -n csi-test" ; \
fi ; 
kubectl get ${i}${ns} | grep ^csi | cut -d ' ' -f 1 \
    | xargs kubectl delete ${i}${ns} ; done
```


Concurrency
-----------

1. Set your current working directory to the directory containing this
   [README.md](README.md).

2. Delete the Basic functionality resources.  You shouldn't have to do
   this, but I'm going to so that we have a minimal replication test for
   the very surprising way this will fail.  See above.

3. Create two concurrent sets of resources and see that both pods come
   up in `Running` state (WOMP WOMP after a failure, they may not; I always
   get at least one so far, though).:
```
for i in 1 2; do
    kubectl apply -f csi-concurrent-pv${i}.yaml
    sleep 1
    kubectl apply -f csi-concurrent-pvc${i}.yaml
    sleep 1
    kubectl apply -f csi-concurrent-pod${i}.yaml
done
sleep 30
kubectl get pods
```

4. If you like, set up a job in each container to verify that writes are
   working.  Exec into each of `csi-concurrent-pod1` and
   `csi-concurrent-pod2` and run something like
```
while : ; do 
    txt="$(date -Ins) $(hostname)"; \
	echo $txt >> /csi/scratch/shared-write.txt; \
	sleep 10; \
done
```
After that runs a bit you can:
```
kubectl exec -t csi-concurrent-pod1 -- cat /csi/scratch/shared-write.txt
```
and validate that you get interleaved output as you would expect.

5. Create a PV/PVC pair for the third pod and verify that the PV and PVC
are bound to one another.
```
kubectl apply -f csi-concurrent-pv3.yaml
sleep 1
kubectl apply -f csi-concurrent-pvc3.yaml
sleep 5
kubectl get pv csi-concurrent-scratch-pv3
kubectl get pvc csi-concurrent-scratch-pvc3
```

6. Attempt to create the third pod for the concurrency test.  It will
fail to start:
```
kubectl apply -f csi-concurrent-pod3.yaml
sleep 30
kubectl describe -n csi-test pod csi-concurrent-pod3
```

The error will look something like

```
  Warning  FailedMount  6s (x7 over 38s)  kubelet            MountVolume.SetUp failed for volume "csi-concurrent-scratch-pv3" : rpc error: code = Internal desc = mount "/var/lib/kubelet/pods/74205c3e-f57c-4bf7-b57f-6a64db788e02/volumes/kubernetes.io~csi/csi-concurrent-scratch-pv3/mount" failed: mount failed: exit status 32
Mounting command: mount
Mounting arguments: -t nfs -o bind /var/lib/kubelet/plugins/kubernetes.io/csi/pv/csi-concurrent-scratch-pv3/globalmount /var/lib/kubelet/pods/74205c3e-f57c-4bf7-b57f-6a64db788e02/volumes/kubernetes.io~csi/csi-concurrent-scratch-pv3/mount
Output: mount: special device /var/lib/kubelet/plugins/kubernetes.io/csi/pv/csi-concurrent-scratch-pv3/globalmount does not exist
```

7. Scratch your head in bafflement.

8. Delete pod1 and its PVC, and then try restarting pod3.  This too will
fail:
```
kubectl delete pod csi-concurrent-pod1
kubectl delete pvc csi-concurrent-scratch-pvc1
```
wait until the pod and PVC are gone.  Then:
```
kubectl delete pod csi-concurrent-pod3
kubectl apply -f csi-concurrent-pod3.yaml
```
The error message will be similar.

9. Delete pod1's PV.  This time creating pod3 will work fine.  This
shows that it's the existence of the third PersistentVolume pointing to
the same filestore that causes trouble.  (NOPE I'M WRONG.)

It turns out to be, the second pod on the same node, is what doesn't
work.  Thanks to Saikat at Google for figuring this out for us.
```
kubectl delete pv csi-concurrent-scratch-pv1
kubectl delete pod csi-concurrent-pod3
kubectl apply -f csi-concurrent-pod3.yaml
```
(WOMP WOMP.  This time it failed.)

10.  Had that worked repeatably, exec into pod3 if you like, and
```
while : ; do 
    txt="$(date -Ins) $(hostname)"; \
	echo $txt >> /csi/scratch/shared-write.txt; \
	sleep 10; \
done
```
Then 
```
kubectl exec -t csi-concurrent-pod2 -- cat /csi/scratch/shared-write.txt
```
and see that the end of the file shows pods 2 and 3 writing happily away.

11.  Weep bitter tears.

Conclusion
----------

The Rubin Observatory use case, for which we were trying to use the CSI
filestore driver, is this:

We have many (potentially hundreds-to-thousands) of simultaneous users
running user pods.  Each user gets their own namespace, to enable us to
restrict their CPU and memory usage by K8s quotas, and to simplify
releasing resources when users log out (by simply tearing down the
namespace).

We are currently using the vanilla NFS driver to map three persistent
storage endpoints (home, project, and scratch) to these user pods.  We
could use a filestore-per-user-home if we wanted, but the minimal
filestore size of 1TiB (2.5TiB if SSD, which we would prefer) is
enormously larger than a typical user will need, and would be extremely
costly on the scales we will require.  That wouldn't help us anyway for
the project and scratch volumes, which are explicitly for collaboration
where multiple users will need to write to the same directories, and,
perhaps, in some cases, to the same files.

Therefore what we need is a small number of persistent filestores that
exist independent of the user pods consuming them.  That already means
we're not using the coolest feature of the CSI Filestore driver, which
is dynamic provisioning of the backing filestore resource.  However, CSI
Filestore also supports a statically-pre-provisioned filestore.

We had hoped to use the CSI Filestore driver and pre-provisioned
filestores in order to implement our shared storage.  However, our
testing revealed the flaw in this plan.  As it turns out, the driver's
scheme for aggregating mountpoints for the volume will only work for the
first pod scheduled on each node.

The intention of the CSI Filestore driver (rather than having multiple
PVs referencing the same filestore, which owing to the driver design
fails as soon as multiple pods referencing the same filestore are
scheduled to the same node) is for many pods to share the same PV/PVC
pair.  Although it is perfectly plausible for multiple PV resources (not
namespaced) to point to the same underlying filestore, it is exactly the
driver's cleverness with bind-mounting to aggregate filesystem access
through a single connection (a perfectly reasonable optimization) that
causes this approach to fail as soon as two consumers of PV/PVC pairs
that reference the same underlying filestore are scheduled on a single
physical node.  This is what we demonstrated in the csi-filestore-test
materials at https://github.com/lsst-sqre/csi-filestore-test, and what
Saikat analyzed for us and explained (neither Dan nor I had realized
that our tests worked only so long as our pods were scheduled on
different nodes).

Now, if we were to run our user pods in the same namespace, we could use
the CSI Filestore driver as intended, and share a small number of PV/PVC
pairs between all pods.  However, since a PVC is a namespaced resource,
this cannot work if our user pods are in individual namespaces.  (The
one-to-one mapping of PVs and PVCs, when PVCs are namespaced and PVs are
not, has been an annoyance to us before; a hypothetical ClusterPVC
resource that behaved like a PVC but was shareable among pods running in
different namespaces, would obviate this problem entirely, because then
we could have one PV/CPVC pair per filestore, rather than one PV/PVC
pair per pod.)

Since we want to retain individual user namespaces, and since a user pod
is going to be enormously smaller than a physical node (meaning
scheduling each user on an individual node would become cost-prohibitive
almost immediately, even leaving aside the problem of prewarming the
image cache with frequently-used images), the CSI driver cannot
currently provide us with the features we need.
