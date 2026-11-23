# Session 4: the generator on GKE

## kubectl get pods -o wide
```
NAME                        READY   STATUS    RESTARTS   AGE   IP           NODE                             NOMINATED NODE   READINESS GATES
generator-78898f6ff-lp6kp   1/1     Running   0          87s   10.72.0.19   gke-freight-main-8ea75779-nfr5   <none>           <none>
```

## kubectl rollout history deployment/generator
```
deployment.apps/generator 
REVISION  CHANGE-CAUSE
2         <none>
3         <none>

```
## Deleting the pod

When I deleted the pod generator-78898f6ff-44zhq, a new pod
(generator-78898f6ff-2gdz7) was Running within about a second.
I didn't restart anything myself: the Deployment declares one replica,
so when the controller saw zero pods it created a new one to match
the desired state.
