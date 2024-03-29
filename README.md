# Tekton - Cloud Native CI/CD

This is the repo of the project for the course Special Topics Cloud Computing at the JKU Linz.

The original proposal for this project can be found in [PROPOSAL.md](PROPOSAL.md).


## Learnings and project structure

### Tekton Basics
Tekton is a powerful yet flexible Kubernetes-native open source framework for creating continuous integration and delivery (CI/CD) systems. The goal is to make CI/CD easier for everyone. It does this by giving you maximum flexibility, meaning, it abstracts the underlying implementation details and specific concepts of Kubernetes. The users can focus on creating powerful pipelines and choose the build, test and deploy workflows based on their team's requirements. All the building blocks are basically Kubernetes CRDs (Custom Resource Definition). 

#### Building blocks
`Step`: the smallest unit in Tekton. Each step is equivalent to a container. Can have specified inputs and outputs.  
`Task`: sequence of steps running a sequence of containers. It executes a pod on the K8s cluster. All steps inside a task have access to a shared workspace which is mounted to the pod as an implicit volume. Tekton tasks are a K8s CRD of kind Task, therefore they have a specified name so that they can be referenced (kubectl get tasks) and reused. Lastly, they can be executed with the TaskRun CRD, which provides parameters needed by the task.  
`Pipeline`: sequence of tasks running a sequence of K8s pods. They receive input in the form of parameters which are predetermined by the tasks' parameters + produce output in the form of results. Tasks inside a pipeline can communicate via results or share lots of data in a shared persitent volume (workspace). Tekton pipelines are also a K8s CRD of kind Pipeline, must have a name and they are executed with the help of PipeRuns (CRD).  
`Trigger`: help to automatically invoke a pipeline (e.g. after push on repo). Triggers consist of three CRD: `TriggerBinding`, `TriggerTemplate`, `Eventlistener`. The binding extracts relevant information inside the event payload and sets some of the later used parameters. The template is a blueprint for creating pipelines and/or tasks, i.e., it defines the PipelineRuns and/or TaskRuns. The eventlistner K8s object is basically a service that encompasses both the TriggerTemplates and TriggerBindings and listens for events.

### Concrete project structure
![project_structure](project_structure.png)

## Summary of research
To get familiar with Tekton implementing the Getting Started with [Tasks](https://tekton.dev/docs/getting-started/tasks/), with [Pipelines](https://tekton.dev/docs/getting-started/pipelines/) and with [Triggers](https://tekton.dev/docs/getting-started/triggers/) from the Tekton documentation helps a lot. 

Further resources are: [Tekton Documentation](https://tekton.dev/docs/),
[Tekton Youtube Videos](https://tekton.dev/docs/getting-started/), [Code in Action](https://www.youtube.com/playlist?list=PLeLcvrwLe185j54ZV-CoPjeixY8GjEwVB) and [Sebastian Daschner](https://www.youtube.com/playlist?list=PLEV9ul4qfGOYLooAW9hnekIOyCMtI7zaZ)

## Tutorial

### Requirements
- App repository with a main (live) and dev branch
- kubectl installed

### Commands
First, we install the Tekton Pipeline. This basically includes the main building blocks/CRDs, including: Pipeline, PipelineRun, Task and TaskRun.  
`kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml` 

Install all that resources that are needed for triggers + the different kinds of interceptors.  
`kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml`      
`kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml`

Install Tekton Dashboard, which is a general purpose, web-based UI for Tekton Pipelines and Tekton triggers resources. It allows users to manage and view Tekton resource creation, execution, and completion.  
`kubectl apply -f https://github.com/tektoncd/dashboard/releases/latest/download/tekton-dashboard-release.yaml`   

Now we create a namespace for our project and its resources (to have them nicely seperated from the default namespace).   
`kubectl apply -f namespace.yaml`    

Define the workspace which will later be shared among tasks of the same pipeline.  
`kubectl apply -f tekton-demo/workspace.yaml`  

After we push to the dev branch, we have a task that logs information from the push event.  
`kubectl apply -f task-logger-dev.yaml`  

After a successful pull request, we log additional information (merged by & branch)  
`kubectl apply -f task-logger-live.yaml`

The following four tasks are all from the Tekton Hub. From the Tekton Hub, developers can access predefined Tasks and Pipelines (for commonly occuring tasks) that were shared by the community.   
`kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.9/git-clone.yaml -n tekton-demo`    
`kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/golangci-lint/0.2/golangci-lint.yaml -n tekton-demo`    
`kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/golang-test/0.2/golang-test.yaml -n tekton-demo`    
`kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/golang-build/0.3/golang-build.yaml -n tekton-demo`   

Then we create one dev and one live Pipeline resource, which are composed of previously created task definions.   
`kubectl apply -f pipeline-dev.yaml`      
`kubectl apply -f pipeline-live.yaml`  

In order to extract specific payload information, we create two seperate bindings (one for dev, one for live).   
`kubectl apply -f trigger-binding-dev.yaml`    
`kubectl apply -f trigger-binding-live.yaml`  

The TriggerTemplates defines what happens when a push event is detected or a pull request, respectively.    
`kubectl apply -f trigger-template-dev.yaml`    
`kubectl apply -f trigger-template-live.yaml`  

The EventListener requires a service account to run, i.e., this service account allows the EventListener to create PipelineRuns. The role bindings basically define what the eventlistener is allowed to do and cannot do, respectively.   
`kubectl apply -f rbac.yaml`    

```
serviceaccount/triggers-sa created
rolebinding.rbac.authorization.k8s.io/triggers-example-eventlistener-binding created
clusterrolebinding.rbac.authorization.k8s.io/triggers-example-eventlistener-clusterbinding unchanged
```

The interceptor of type "github" (definied in the eventlistener.yaml) checks if various constraints are met, including the check for the correct secret sent by GitHub. The secret is definied here:  
`kubectl apply -f secret.yaml`  

Last but not least, we create an eventlistener of service type "load balancer" (so we can access it from the outside rather than only from localhost with port forwarding).  
`kubectl apply -f eventlistener.yaml`  

Start the Tekton Dashboard using port forwarding.  
`kubectl -n tekton-pipelines port-forward svc/tekton-dashboard 9097:9097`  

### GitHub Webhook
Using GitHub Webhooks, we can create http requests after a specific event has happend. In our case, we got two  webhooks, one for pushes on the dev branch and one for pull requests. 

First we need to get the IP and port from the eventlistener:  

`kubectl get svc/el-cd-listener -n tekton-demo`    
```
NAME             TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)                         AGE
el-cd-listener   LoadBalancer   10.76.8.90   35.226.176.142   8080:32540/TCP,9000:30752/TCP   48s
``` 

Then we go to our app repo, under "Settings > Webhook" and enter the "Payload URL", set the content type to JSON, enter the previously definied secret, and select "Just the push events".

![webhook](webhook.jpg)

For the pull request, do the same steps as before, just switch from push events to "Let me select individual events > Pull Requests". 

### GitHub Action
GitHub Actions can be used by creating a YAML-File int the .github/workflows/ directory. All YAML-Files in this directory will automatically be executed as actions. 

The code below shows the content of our merge-schedule.yml File. Actions start with a name to different them in the action view. The on: sectionis to define on what the action should listen to and what should trigger it. This can be a pull or push or in our case a schedule. We want to create a new version every month (by automatically merging the dev into the main) like a lot of companies have. For test reasons we changed that to once an hour. This can easily be done with crone.

The jobs contain what the action should do once it is triggered. In our case it builds on the latest version of ubuntu. It checks out the code, sets a git config (necessary to automatically merge), it fetches and checks out the main branch. Then it merges the main with the origin/dev with the option --allow-unrelated-histories, to make sure it merges even if there are differences. Lastly it pushes the merged version. 

The GITHUB_TOKEN is needed for merging and pushing by the GitHub bot that runs in the action. To give the bot these rights, it must be activated in the project settings. (Actions/General/Workflow permissions set to "Read and write permissions")

```yml
name: Merge Dev into Main every hour

on:
  schedule:
    # - cron: '0 0 1 * *'   # runs every month on the first
    - cron: '0 * * * *'     # runs every hour

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Set Mail
      run: git config --global user.email "timseferagic@gmail.com"
    
    - name: Set Name
      run: git config --global user.name "Tim"

    - name: Fetch branches
      run: git fetch

    - name: Checkout main branch
      run: git checkout main

    - name: Merge dev branch
      run: git merge origin/dev --allow-unrelated-histories
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

    - name: Push changes
      run: git push
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

This image shows the Actions window from our GitHub project. It shows that our action was executed, that all jobs run through and what it did.
![image](https://user-images.githubusercontent.com/25606213/213167445-8860fe2f-94fa-463d-8c13-603b5302ebb8.png)


## Demo App
The demo Go program sets up a basic HTTP server that listens on port 8080. When a request is made to the server a response is sent to the client with the message "Hello, you've requested: [requested URL path]". The main function of the app sets up the server to handle requests and begins listening for incoming requests on port 8080. It uses log.Fatal function to start the server, that will cause the program to exit if an error occurs while starting the server.

The following image shows the response of the server after making a request to it.
![image](https://user-images.githubusercontent.com/25606213/213170536-ad2e8bac-3dcd-473e-b653-b43a6ac1d713.png)

To run this go app you have to create a docker image and run it with the option to use the port 8080. 

`docker build -t my-go-app .` 
`docker run -p 8080:8080 my-go-app`


## Demo Procedure
### 1. Commit to the dev branch of this [Repo](https://github.com/danielraso7/cloud-computing-project-demoapp)
### 2. Check out the recent push webhook http request 

![webhook-dev](webhook-dev.jpg)

### 3. Open the Tekton Dashboard in the Browser with localhost:9097. In PipelineRuns you should now see 

![dev-pipeline](dev-pipeline.png)

### 4. The Github Action starts the merg of dev into main after a specified time
### 5. Check out the recent pull request webhook http request

![webhook-live](webhook-live.jpg)

### 6. Open the Tekton Dashboard again to also see this pipeline run 

![live-pipeline](live-pipeline.jpg)



## Problems & Results
During the realization of our project we encountered many bumps on the road. Here is a brief description of the hard times Tekton can give you when working with it for the first time:

### 1. tkn has a bug
The Tekton Pipelines CLI (tkn) has a [bug](https://github.com/tektoncd/cli/issues/1837) in its latest version, which makes it impossible to install tasks from Tekton Hub via tkn. 

Solution: Since tkn is not working to this day, we used kubectl.

### 2. Run the go project in Docker
In order to build a Dockerimage and run it, an authentication for e.g. DockerHub is needed. Tekton implements this with Tekton secrets. Sadly, time got too short to understand how to include the DockerHub authentification and therefore build and run the Image.

Solution: We built the project as a golang project, which we could not run in the end. 
Although we did not get the last task of the pipeline running, we learned a great deal about all the other concepts of Tekton.

### 3. Line Indentations of YAML
Sometimes nothing other than the line indentation is wrong with your project and sometimes it takes hours to get it right.

### 4. GitHub Raw
GitHub hosts raw files at https://raw.githubusercontent.com. You can use that link to download the raw contents of the file or use it after `kubectl apply -f ...`. When you push a change to the file, it doesn't update instantly, because raw files are updated after 5 minutes. This fact was pretty annoying when trying to create the same resource after a short period of time, because you had to wait for those ~5 minutes.  
