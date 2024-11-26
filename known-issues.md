# Known Issues and Troubleshooting

## During cluster creation with mwcsim

### Unable to list buckets

```
mwcsim 2024/11/22 08:24:29 error: Unable to list buckets, operation error S3: ListBuckets, https response error StatusCode: 400, RequestID: XXXXXXXXXXXXXXXX, HostID: XXXXXXXXXXXXXXXX/XXXXXXXXXXXXXXXX, api error ExpiredToken: The provided token has expired.
```

This means that `mwcsim` is unable to access AWS resources.  Please login again to AWS using the AWS command line interface.


```
# the exact AWS CLI login procedures might be different depending on your IAM settings
aws configure
```

Another form of the `Unable to list buckets` error can occur if the region is provided and it is not valid.  Please double check your AWS region and try again.

### Timed out waiting for HTTPS endpoint

```
mwcsim 2024/11/25 11:45:55 waiting for HTTPS ready and retrying (timeout 5 mins)
mwcsim 2024/11/25 11:46:00 waiting for HTTPS ready and retrying (timeout 5 mins)
mwcsim 2024/11/25 11:46:05 waiting for HTTPS ready and retrying (timeout 5 mins)
mwcsim 2024/11/25 11:46:10 waiting for HTTPS ready and retrying (timeout 5 mins)
mwcsim 2024/11/25 11:46:10 fatal error occurred, releasing lock then exiting
mwcsim 2024/11/25 11:46:10 configuration lock released by current host "host-name"
mwcsim 2024/11/25 11:46:10 error, timeout exceeded while trying to send HTTPS request
```

This error can occur when there are delays while provisioning the cluster loadbalancer endpoint or there are HTTPS cert delays.  Please retry the operation.  The creation command will pick up where it left off.

### No endpoints available for service

```
mwcsim 2024/11/21 17:25:15 Error from server (InternalError): Internal error occurred: failed calling webhook "validate.nginx.ingress.kubernetes.io": failed to call webhook: Post "https://ingress-nginx-controller-admission.ingress-nginx.svc:443/networking/v1/ingresses?timeout=10s": no endpoints available for service "ingress-nginx-controller-admission"
```

This error can occur when there are delays while provisioning the cluster loadbalancer endpoint.  Wait a few minutes and then retry the operation.  The creation command will pick up where it left off.

### Failed to send keycloak request

```
mwcsim 2024/11/21 17:27:16 failed to send keycloak request: Post "https://mydomainasdjwomfx.mwcsim.io/auth/realms/master/protocol/openid-connect/token": EOF
```

This error can occur when there are delays while provisioning the cluster HTTPS certificates.  Wait a few minutes and then retry the operation.  The creation command will pick up where it left off.

### Access token request failed

```
mwcsim 2024/11/21 17:40:09 access token request failed with status: 401 Unauthorized
```

The Keycloak deployment may not have started correctly.  Re-initialize Keycloak with the following commands.

```
# remove keycloak and databases
mwcsim delete cluster my-cluster -x kubeapply1

# recreate keycloak and databases
mwcsim create cluster my-cluster
```

### Cannot access keycloak admin console with admin credentials

The Keycloak deployment may not have started correctly.  Re-initialize Keycloak with the following commands.

```
# remove keycloak and databases
mwcsim delete cluster my-cluster -x kubeapply1

# recreate keycloak and databases
mwcsim create cluster my-cluster
```

### Timed out trying to reach CDDS

You'll get this error if you try to use the domain auto-provisioning feature. Please contact us if you're interested in this beta feature.

## On MATLAB client connecting to the cluster

### 503 error while running addCluster

```
Error using simulink.cloud.internal.Authenticator/getResource
ERROR: HTTP/1.1 503 Service Temporarily Unavailable

Error in simulink.cloud.internal.Authenticator/login

Error in simulink.cloud.internal.Authenticator

Error in simulink.cloud.internal.addCluster

Error in simulink.cloud.addCluster
```

A cluster user may not be created yet.  Create a user and then try again.  You can use the Keycloak user management console or use the convenience command below.

```
# credentials will be returned if successful
mwcsim create user my-username -c my-cluster
```

## During cluster operation

### Cluster error with paramservsvc

```
>> job
 
job =
 
  Job with properties:
 
               State: Failed

            Progress: Completed 0 simulations

      SubmitDateTime: 22-Nov-2024 10:40:40

     RunningDuration: 3.40970129s
 
               Error: Cluster configuration or resource error encountered. Contact cluster admin and provide the information in the error cause.
 
Caused by:

    Service "paramservsvc-24016ffd" not found

    Error using paramprocessing-operator/controllers/paramprocessor.(*ParamProcessorReconciler).Reconcile in

    /workspace/controllers/paramprocessor/reconcile.go (line 277)
```

We're actively looking into this issue. As a temporary workaround, this sporadic error can be resolved by the cluster administrator by deleting the paramprocessing controller and letting it restart.

```
kubectl delete pod -l control-plane=controller-manager -n paramprocessing-operator-system
```

### Last few simulation units in a job may stall

Occasionally, the last few simulations in a job may stall and the job may be in `Running` state indefinitely.

```
>> job

job = 

  Job with properties:

               State: Running
            Progress: Completed 24 of 25 simulations
      SubmitDateTime: 25-Nov-2024 12:09:15
     RunningDuration: 7m43.422085247s
```

We're actively looking into this issue. As a temporary workaround, the cluster administrator can delete the `simsvc` pods on the cluster to progress and finish up the job. The following command will loop over all the pods and delete them so they're force restarted.

```
kubectl get pods -n cloudsim-simservices -l app=simsvc,cloudsim-pod-type=simsvc-pod --no-headers -o custom-columns=":metadata.name" | while read -r simsvcpod; do
    kubectl delete pod $simsvcpod -n cloudsim-simservices
done
```

## During cluster deletion with mwcsim

### Errors while deleting kubernetes resources

```
deploy0": the server could not find the requested resource (delete servers.policy.linkerd.io prometheus-admin)
mwcsim 2024/11/21 21:40:37 Error from server (NotFound): error when deleting "/home/user/.mwcsim/manifests/v1/ganymede-eosybokg/deploy0": the server could not find the requested resource (delete servers.policy.linkerd.io tap-api)
mwcsim 2024/11/21 21:40:37 Error from server (NotFound): error when deleting "/home/user/.mwcsim/manifests/v1/ganymede-eosybokg/deploy0": the server could not find the requested resource (delete servers.policy.linkerd.io tap-injector-webhook)
mwcsim 2024/11/21 21:40:37 Error from server (NotFound): error when deleting "/home/user/.mwcsim/manifests/v1/ganymede-eosybokg/deploy0": pods "kube-prometheus-stack-grafana-test" not found
```

The above types of `not found` errors are benign and can be ignored during deletion.
