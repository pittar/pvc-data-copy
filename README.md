# Copy Data to new PersitentVolumeClaim (PVC)

## Background

Sometimes you may need to copy data to a new PVC.  For example, if you have reach the capacity of a PVC and the `StorageClass` you are using doesn't support volume expansion, then you will need to create a new larger PVC, copy the data to it, and mount the new PVC in your `Pod`.

There are a few different ways you could achieve this goal, but a simple Kubernetes `Job` is fast and effective, as well as easy to re-use.

## Resources

There are only two resources in this repository - only one of which you actually need.

* `copy-data-job.yaml` - The `Job` that will mount the old and new PVCs, then copy the data from old to new.
* `nexus-data-pvc.yaml` - An example of a new PVC.  You an create your new PVC however you like.  This file is simply an example.

## Process

### 1. Create the new PVC

First, you will want to create your new `PersistentVolumeClaim`.  You should make sure to create it in the correct namespace.

For this example, I want to expand the storage for my instance of *Nexus* from **10Gi** to **20Gi**.  My nexus instance currently resides in my `cicd` namespace.

The new PVC will need a unique name.  So, first I'll check to see what the current PVC name is:


```
$ oc get pvc -n cicd

NAME                        STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nexus                       Bound     pvc-bc2ab139-f03a-48da-853b-ccb7c51901ae   10Gi       RWO            gp2            108m
postgresql-sonarqube-data   Bound     pvc-9245c7cf-7695-4550-b1bb-62d042d5b89b   1Gi        RWO            gp2            108m
sonarqube-data              Bound     pvc-0f44013b-6485-4def-8f2c-c43128f0c02d   1Gi        RWO            gp2            108m
```

You can see the current name is `nexus`.  So, I'll call my new PVC `nexus-data`.

I'll create the new PVC with the following command, but you can create your own PVC any way you like.

```
$ oc create -f pvc/nexus-data-pvc.yaml -n cicd
```

Once that operation completes, you can confirm that it was created (although it may still be "unbound", which is fine).

```
$ oc get pvc -n cicd

NAME                        STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nexus                       Bound     pvc-bc2ab139-f03a-48da-853b-ccb7c51901ae   10Gi       RWO            gp2            113m
nexus-data                  Bound     pvc-18f9969a-ebaa-4538-b58b-8dc784dd95e9   20Gi       RWO            gp2            68m
postgresql-sonarqube-data   Bound     pvc-9245c7cf-7695-4550-b1bb-62d042d5b89b   1Gi        RWO            gp2            113m
sonarqube-data              Bound     pvc-0f44013b-6485-4def-8f2c-c43128f0c02d   1Gi        RWO            gp2            113m
```

### 2. Scale Down Your Deployment

To make sure you don't corrupt any data and to avoid any issues that can arise from trying to mount `RWO` (Read-Write Once) type PVCs to two pods at the same time, you should be sure to scale your deployment down to 0.  You can do this with the OpenShift UI, or with the `oc` CLI.

For example:
```
# Scale down a "Deployment"
$ oc scale deployment nexus --replicas=0 -n cicd

# Scale down a "DeploymentConfig"
$ oc scale deploymentconfig nexus --replicas=0 -n cicd
```

### 3. Prepare your Job

Make a copy of the `copy-data-job.yaml` file found in the `/pvc` directory of this repository.

You will notice that the `volumes` stanza needs you to add your PVC names.  It currently looks like this:

```
      volumes:
        - name: old-pvc
          persistentVolumeClaim:
            claimName: <old PVC name>
        - name: new-pvc
          persistentVolumeClaim:
            claimName: <new PVC name>
```

For my example, my old PVC name is *nexus* and the new one is *nexus-data*, so the `volumes` stanza of my Job will look like:

```
      volumes:
        - name: old-pvc
          persistentVolumeClaim:
            claimName: nexus
        - name: new-pvc
          persistentVolumeClaim:
            claimName: nexus-data
```

You can take a look at the rest of the Job to see how it works, but there isn't much too it.  It simply mounts both PVCs to different directories (`/data/old-pvc` and `/data/new-pvc`), then calls a resursive copy from old to new.  That's it!

### 3. Run the Job

Now that:
1. You hava create a new (larger) PVC.
2. Stopped the Pod that's conneted to the old PVC.
3. Copied and updated the `Job`.

You're ready to copy some data!

Now, simply apply create the Job (and watch the logs if you want to see the progress).

```
$ oc apply -f copy-data-job.yaml -n cicd
$ oc logs job/copy-data-job -f -n cicd
```

When it completes, you can delete the Job to clean it up.

```
$ oc delete job copy-data-job -n cicd
```

### 4. Update you Deployment or DeploymentConfig

Finally, you need to update the `Deployment` of `DeploymentConfig` to use your new PVC.

**Note:** Make sure you **DO NOT** update the `volumeMounts` stanza in your Deployment/DeploymentConfig.  That will stay the same.

Edit your `Deployment` or `DeploymentConfig` and find the `volumes` stanza.  Simply replace the `claimName` with the name of your new PVC.  Using my example, the "old" PVC name is `nexus` and the "new" PVC name is "nexus-data".  

Old value:
```
      volumes:
        - name: nexus-data
          persistentVolumeClaim:
            claimName: nexus # This is the old PVC name.
```

New value:
```
      volumes:
        - name: nexus-data
          persistentVolumeClaim:
            claimName: nexus-data # This is the new PVC name.
```

Save or apply your `Deployment/DeploymentConfig`.

### 5. Scale Up your Deployment

Everything is set!  All that is left is to scale up your `Deployment` or `DeploymentConfig` and test things out.  You can do this through the OpenShift UI or cli.

For example:
```
# Scale down a "Deployment"
$ oc scale deployment nexus --replicas=1 -n cicd

# Scale down a "DeploymentConfig"
$ oc scale deploymentconfig nexus --replicas=1 -n cicd
```

Done!

### 6. Clean-Up

Once you have determined that the process was successful, you can delete your old PVC to reclaim your data.  Make sure you delete the correct PVC!

```
$ oc delete pvc nexus -n cicd
```



