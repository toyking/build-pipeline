# How to use the Pipeline CRD

* [How do I create a new Pipeline?](#creating-a-pipeline)
* [How do I make a Task?](#creating-a-task)
* [How do I make Resources?](#creating-resources)
* [How do I run a Pipeline?](#running-a-task)
* [How do I run a Task on its own?](#running-a-task)
* [How do I troubleshoot a PipelineRun?](#troubleshooting)

## Creating a Pipeline

1. Create or copy [Task definitions](#creating-a-task) for the tasks you’d like to run.
   Some can be generic and reused (e.g. building with Kaniko) and others will be
   specific to your project (e.g. running your particular set of unit tests).
2. Create a `PipelineParams` definition which includes parameters such as what repos
   to run against, where to store results, etc.
3. Create a `Pipeline` which expresses the Tasks you would like to run and what
   [Resources](#creating-resources) the Tasks need.
   Use [`providedBy`](#providedBy) to express the order the `Tasks` should run in.

See [the example guestbook Pipeline](../examples/pipelines/guestbook.yaml) and
[the example kritis Pipeline](../examples/pipelines/kritis.yaml).

### ProvidedBy

When you need to execute `Tasks` in a particular order, it will likely be because they
are operating over the same `Resources` (e.g. your unit test task must run first against
your git repo, then you build an image from that repo, then you run integration tests
against that image).

We express this ordering by adding `providedBy` on `Resources` that our `Tasks`
need.

* The (optional) `providedBy` key on an `input source` defines a set of previous
  task names.
* When the `providedBy` key is specified on an input source, only the version of
  the resource that is provided by the defined list of tasks is used.
* The `providedBy` allows for `Tasks` to fan in and fan out, and ordering can be
  expressed explicitly using this key since a task needing a resource from a another
  task would have to run after.
* The name used in the `providedBy` is the name of `PipelineTask`

## Creating a Task

To create a Task, you must:

* Define [parameters](task-parameters.md) (i.e. string inputs) for your `Task`
* Define the inputs and outputs of the `Task` as [`Resources`](./Concepts.md#pipelineresources)
* Create a `Step` for each action you want to take in the `Task`

`Steps` are images which comply with the [image contract](#image-contract).

### Image Contract

Each container image used as a step in a [`Task`](#task) must comply with a specific
contract.

When containers are run in a `Task`, the `entrypoint` of the container will be
overwritten with a custom binary that redirects the logs to a separate location
for aggregating the log output. As such, it is always recommended to explicitly
specify a command.

When `command` is not explicitly set, the controller will attempt to lookup the
entrypoint from the remote registry.

Due to this metadata lookup, if you use a private image as a step inside a
`Task`, the build-pipeline controller needs to be able to access that registry.
The simplest way to accomplish this is to add a `.docker/config.json` at
`$HOME/.docker/config.json`, which will then be used by the controller when
performing the lookup

For example, in the following Task with the images, `gcr.io/cloud-builders/gcloud`
and `gcr.io/cloud-builders/docker`, the entrypoint would be resolved from the
registry, resulting in the tasks running `gcloud` and `docker` respectively.

```yaml
spec:
  steps:
  - image: gcr.io/cloud-builders/gcloud
    command: [gcloud]
  - image: gcr.io/cloud-builders/docker
    command: [docker]
```

However, if the steps specified a custom `command`, that is what would be used.

```yaml
spec:
  steps:
  - image: gcr.io/cloud-builders/gcloud
    command:
    - bash
    - -c
    - echo "Hello!"
```

You can also provide `args` to the image's `command`:

```yaml
steps:
- image: ubuntu
  command: ['/bin/bash']
  args: ['-c', 'echo hello $FOO']
  env:
  - name: 'FOO'
    value: 'world'
```

### Images Conventions

 * `/workspace`: If an input is provided, the default working directory will be
   `/workspace` and this will be shared across `steps` (note that in
   [#123](https://github.com/knative/build-pipeline/issues/123) we will add supprots for multiple input workspaces)
 * `/builder/home`: This volume is exposed to steps via `$HOME`.
 * Credentials attached to the Build's service account may be exposed as Git or
   Docker credentials as outlined
   [in the auth docs](https://github.com/knative/docs/blob/master/build/auth.md#authentication).

### Templating

Tasks support templating using values from all `inputs` and `outputs`. Both
`Resources` and `Params` can be used inside the `Spec` of a `Task`.

`Resources` can be referenced in a `Task` spec like this, where `NAME` is the
Resource Name and `KEY` is one of `name`, `url`, `type` or `revision`:

```shell
${inputs.resources.NAME.KEY}
```

To access a `Param`, replace `resources` with `params` as below:

```shell
${inputs.params.NAME}
```

## Running a Pipeline

1. To run your `Pipeline`, create a new `PipelineRun` which links your `Pipeline` to the
   `PipelineParams` it will run with.
2. Creation of a `PipelineRun` will trigger the creation of [`TaskRuns`](#running-a-task)
   for each `Task` in your pipeline.

See [the example PipelineRun](../examples/invocations/kritis-pipeline-run.yaml).

## Running a Task

1. To run a `Task`, create a new `TaskRun` which defines all inputs, outputs
   that the `Task` needs to run.
2. The `TaskRun` will also serve as a record of the history of the invocations of the
   `Task`.

See [the example TaskRun](../examples/invocations/run-kritis-test.yaml).

## Creating Resources

### Git Resource

Git resource represents a [git](https://git-scm.com/) repository, that containes
the source code to be built by the pipeline. Adding the git resource as an input
to a task will clone this repository and allow the task to perform the required
actions on the contents of the repo.

Use the following example to understand the syntax and strucutre of a Git Resource

1. Create a git resource using the `PipelineResource` CRD

    ```
    apiVersion: pipeline.knative.dev/v1alpha1
    kind: PipelineResource
    metadata:
      name: wizzbang-git
      namespace: default
    spec:
      type: git
      params:
      - name: url
        value: https://github.com/wizzbangcorp/wizzbang.git
      - name: Revision
        value: master
    ```

   Params that can be added are the following:

   1. URL: represents the location of the git repository
   1. Revision: Git [revision](https://git-scm.com/docs/gitrevisions#_specifying_revisions ) (branch, tag, commit SHA or ref) to clone. If no revision is specified, the resource will default to `latest` from `master`

2. Use the defined git resource in a `Task` definition:

    ```
    apiVersion: pipeline.knative.dev/v1alpha1
    kind: Task
    metadata:
      name: build-push-task
      namespace: default
    spec:
      inputs:
        resources:
        - name: wizzbang-git
          type: git
        params:
        - name: pathToDockerfile
      outputs:
        resources:
        - name: builtImage
          type: image
      steps:
      - name: build-and-push
        image: gcr.io/my-repo/my-imageg
        args:
        - --repo=${inputs.resources.wizzbang-git.url}
    ```

3. And finally set the version in the `TaskRun` definition:

    ```
    apiVersion: pipeline.knative.dev/v1alpha1
    kind: TaskRun
    metadata:
      name: build-push-task-run
      namespace: default
    spec:
      taskRef:
        name: build-push-task
      inputs:
        resourcesVersion:
        - resourceRef:
            name: wizzbang-git
            apiVersion: HEAD
      outputs:
        resources:
        - name: builtImage
          type: image
      # Optional, indicate a serviceAccount for authentication.
      serviceAccount: build-bot
    ```

### Cluster Resource

Cluster Resource represents a Kubernetes cluster other than the current cluster the pipleine 
CRD is running on. A common use case for this resource is to deploy your application/function 
on different clusters. 

The resource will use the provided parameters to create a [kubeconfig](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) file 
that can be used by other steps in the pipeline task to access the target cluster. The kubeconfig will be 
placed in `/workspace/<your-cluster-name>/kubeconfig` on your task container

The Cluster resource has the following parameters:
 * Name: The name of the Resource is also given to cluster, will be used in the kubeconfig and also as part of the path to the kubeconfig file 
 * URL (required): Host url of the master node         
 * Username (required): the user with access to the cluster 
 * Password: to be used for clusters with basic auth
 * Token: to be used for authenication, if present will be used ahead of the password
 * Insecure: to indicate server should be accessed without verifying the TLS certificate. 
 * CAData (required): holds PEM-encoded bytes (typically read from a root certificates bundle).

Note: Since only one authentication technique is allowed per user, either a token or a password should be provided, if both are provided, the password will be ignored.

The following example shows the syntax and structure of a Cluster Resource

```
apiVersion: pipeline.knative.dev/v1alpha1
kind: PipelineResource
metadata:
  name: test-cluster
spec:  
  type: cluster
  params:
  - name: url
    value: https://10.10.10.10    # url to the cluster master node   
  - name: cadata
    value: LS0tLS1CRUdJTiBDRVJ.....
  - name: token
    value: ZXlKaGJHY2lPaU....
```

For added security, you can add the sensetive information in a Kubernetes [Secret](https://kubernetes.io/docs/concepts/configuration/secret/) and populate the kubeconfig from them.

For example, create a secret like the following example

```
apiVersion: v1
kind: Secret
metadata:
  name: target-cluster-secrets
data:
  cadatakey: LS0tLS1CRUdJTiBDRVJUSUZ......tLQo=
  tokenkey: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbX....M2ZiCg==
```

and then apply secrets to the cluster resource 

```
apiVersion: pipeline.knative.dev/v1alpha1
kind: PipelineResource
metadata:
  name: test-cluster
spec:  
  type: cluster
  params:
  - name: url
    value: https://10.10.10.10 
  - name: username
    value: admin  
  secrets:  
  - fieldName: token
    secretKey: tokenKey
    secretName:  target-cluster-secrets
  - fieldName: cadata
    secretKey: cadataKey
    secretName:  target-cluster-secrets    
```

Example usage of the cluster resource in a task

```
apiVersion: pipeline.knative.dev/v1alpha1
kind: Task
metadata:
  name: deploy-image
  namespace: default
spec:
  inputs:
    resources:
    - name: workspace
      type: git
    - name: dockerimage
      type: image
    - name: testcluster
      type: cluster    
  steps:
  - name: deploy
    image: image-wtih-kubectl
    command: ['bash']
    args:
    - '-c'
    - kubectl --kubeconfig /workspace/${inputs.resources.testCluster.Name}/kubeconfig --context ${inputs.resources.testCluster.Name} apply -f /workspace/service.yaml'
```

## Troubleshooting

All objects created by the build-pipeline controller show the lineage of where
that object came from through labels, all the way down to the individual build.

There are a common set of labels that are set on objects. For `TaskRun` objects,
it will receive two labels:

* `pipeline.knative.dev/pipeline`, which will be set to the name of the owning pipeline
* `pipeline.knative.dev/pipelineRun`, which will be set to the name of the PipelineRun

When the underlying `Build` is created, it will receive each of the `pipeline`
and `pipelineRun` labels, as well as `pipeline.knative.dev/taskRun` which will
contain the `TaskRun` which caused the `Build` to be created.

In the end, this allows you to easily find the `Builds` and `TaskRuns` that are
associated with a given pipeline.

For example, to find all `Builds` created by a `Pipeline` named "build-image",
you could use the following command:

```shell
kubectl get builds --all-namespaces -l pipeline.knative.dev/pipeline=build-image
```
