# Dokumentation

## Authenticate cluster access
- go to cluster in google cloud, press the three dots on the right of the cluster and click "Connect". Copy the given command into your command line ;)

---
## Task Tutorial 
https://tekton.dev/docs/getting-started/pipelines/

### Install tekton on cluster
`kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml`

### Apply hello world task to cluster
`kubectl apply --filename hello-world.yaml`

output:
`task.tekton.dev/hello created`

### Apply task run
`kubectl apply --filename hello-world-run.yaml`

output: 
`taskrun.tekton.dev/hello-task-run created`

### Verify that it worked
`kubectl get taskrun hello-task-run`

output: 
```
NAME             SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
hello-task-run   True        Succeeded   2m40s       2m30s
```

### Take a look at the logs
`kubectl logs --selector=tekton.dev/taskRun=hello-task-run`

Displays: `Hello World`

## Pipelines Tutorial
https://tekton.dev/docs/getting-started/pipelines/

### Install tkn 
https://tekton.dev/docs/cli/