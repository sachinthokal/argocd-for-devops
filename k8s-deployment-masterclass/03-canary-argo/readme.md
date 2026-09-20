# Argo Rollouts Complete Reference Guide
```bash
# Get rollout details and watch live progress
kubectl argo rollouts get rollout canary-app-rollout -w -n canary
kubectl argo rollouts get rollout canary-app-rollout -n canary

# Start the Argo Rollouts web dashboard UI locally
kubectl argo rollouts dashboard

# Pause an active rollout progress
kubectl argo rollouts pause canary-app-rollout -n canary

# Promote a rollout to push traffic to the next step / version
kubectl argo rollouts promote canary-app-rollout -n canary

# Abort a rollout to immediately roll back to the stable version
kubectl argo rollouts abort canary-app-rollout -n canary

# Retry a failed or aborted rollout
kubectl argo rollouts retry rollout canary-app-rollout -n canary

# Restart all the pods belonging to a rollout
kubectl argo rollouts restart canary-app-rollout -n canary

# Update container image value on a rollout resource
kubectl argo rollouts set image canary-app-rollout web=hashicorp/http-echo:0.2.3 -n canary

# Undo a rollout to revert directly back to a previous revision
kubectl argo rollouts undo canary-app-rollout --to-revision=1 -n canary

# Lint and validate a rollout manifest file for errors
kubectl argo rollouts lint -f rollout.yaml

```