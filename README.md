# Data Converter App


## Prepare the environment

* Install [OpenShift Pipelines Operator](https://docs.openshift.com/container-platform/4.13/cicd/pipelines/installing-pipelines.html).

* Install [App Connect Operator](https://www.ibm.com/docs/en/app-connect/containers_cd?topic=access-installing-app-connect-operator)

* Define [ibm-entitlement-key](https://www.ibm.com/docs/en/app-connect/containers_cd?topic=resources-obtaining-applying-your-entitlement-key) secret

* **[Optional]** Define a shared workspace volume, using the yaml content below:

    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
    name: source-pvc
    spec:
    accessModes:
        - ReadWriteOnce
    volumeMode: Filesystem
    resources:
        requests:
        storage: 1Gi
    ```


## Define OpenShift Pipelines Task

The pipeline uses three tasks, which two of them are already defined as `ClusterTasks`:

1. git-clone
2. buildah

The third task, `deploy-is` is customized for `IntegrationServer` deployment.

* Install the `deploy-is` Task using the yaml content below:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: deploy-is
spec:
  params:
    - description: reference of the image buildah produced.
      name: IMAGE
      type: string
    - default: is-quickstart
      description: IntegrationServer CR name
      name: deployment-name
      type: string
  steps:
    - image: 'quay.io/openshift/origin-cli:latest'
      name: deploy
      resources: {}
      script: |
        cat << EOF | oc apply -f -
          apiVersion: appconnect.ibm.com/v1beta1
          kind: IntegrationServer
          metadata:
            name: $(params.deployment-name)
          spec:
            license:
              accept: true
              license: L-MJTK-WUU8HE
              use: AppConnectEnterpriseNonProductionFREE
            pod:
              containers:
                runtime:
                  image: $(params.IMAGE)
                  imagePullPolicy: Always
                  resources:
                    limits:
                      cpu: 300m
                      memory: 368Mi
                    requests:
                      cpu: 300m
                      memory: 368Mi
            router:
              timeout: 120s
            service:
              endpointType: http
            adminServerSecure: true
            createDashboardUsers: true
            designerFlowsOperationMode: disabled
            enableMetrics: true
            replicas: 1
            version: 12.0.8.0-r1
        EOF
```



## Define OpenShift Pipelines Pipeline


* Install `ace-build-and-deploy` Pipeline using the yaml content below:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: ace-build-and-deploy
spec:
  params:
    - description: name of the deployment to be patched
      name: deployment-name
      type: string
    - description: url of the git repo for the code of deployment
      name: git-url
      type: string
    - default: master
      description: revision to be used from repo of the code for deployment
      name: git-revision
      type: string
    - description: image to be build from the code
      name: IMAGE
      type: string
  tasks:
    - name: fetch-repository
      params:
        - name: url
          value: $(params.git-url)
        - name: subdirectory
          value: ''
        - name: deleteExisting
          value: 'true'
        - name: revision
          value: $(params.git-revision)
      taskRef:
        kind: ClusterTask
        name: git-clone
      workspaces:
        - name: output
          workspace: shared-workspace
    - name: build-image
      params:
        - name: IMAGE
          value: $(params.IMAGE)
      runAfter:
        - fetch-repository
      taskRef:
        kind: ClusterTask
        name: buildah
      workspaces:
        - name: source
          workspace: shared-workspace
    - name: deploy-is
      params:
        - name: IMAGE
          value: $(params.IMAGE)
        - name: deployment-name
          value: $(params.deployment-name)
      runAfter:
        - build-image
      taskRef:
        kind: Task
        name: deploy-is
  workspaces:
    - name: shared-workspace
```


You will notice that the `Pipeline` uses the three `Tasks` defined earlier.


## Start the Pipeline

You can start the pipeline manually in two way:

1. Using the OpenShift console - similar to the recording demo
2. Define `PipelineRun` Custom Resource that refers to the `Pipeline`.

Here's an example of starting the `Pipeline` using `PipelineRun` Custom Resource:

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: ace-build-and-deploy-1
  labels:
    tekton.dev/pipeline: ace-build-and-deploy
spec:
  params:
    - name: deployment-name
      value:  <deployment-name> # exmaple: data-converter-app
    - name: git-url
      value: <git-url> # exmaple: 'https://github.com/RashidAljohani/data-converter-app.git'
    - name: git-revision
      value: main
    - name: IMAGE
      value: <container-image-tag> # exmaple: image-registry.openshift-image-registry.svc:5000/integration-platofrm-dev/data-converter-service
  pipelineRef:
    name: ace-build-and-deploy
  serviceAccountName: pipeline
  timeout: 1h0m0s
  workspaces:
    - name: shared-workspace
      persistentVolumeClaim:
        claimName: source-pvc
```

Pay attention to the `spec.params` values to meet your deployment target.