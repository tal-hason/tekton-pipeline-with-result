# tekton-pipeline-with-result

a Sample Tekton Pipeline, to create a Pipeline Configuration in the Git and allocate it to Task results
![Pipeline](https://github.com/tal-hason/tekton-pipeline-with-result/blob/main/Artifacts/Pipeline.png?raw=true)

## This sample Pipeline demonstrate how to extract content from a file in a Git repo (Using YQ) and using them as Parameters in another task

1. The 1st Step is to clone our git repository with the Config file that we want to get it content from, for this we will use the "Git-Clone" task from Tekton Hub, it's a simple task Just follow the required details. [Git-Clone Tekton Hub](https://hub.tekton.dev/tekton/task/git-clone).

2. The 2nd is a Sample task that i have created, it relays that in the cloned Git Repository there is a folder named ".Pipeline" and in it there is a file named *"Properties.yaml"*, for my demo need the file have the following content:

    ```YAML
    pipelineConfig:
    Project:
        Name: Test-project
    Type: Polyrepo
    Build:
        Main: true
        Tests: true
    GitOps:
        RepoURL: "https://github.com/tal-hason/cd-repo-demo.git"
        ApplicationPath: Application/
    ```

3. The *"Get-Pipeline-Config.yaml"* task uses "YQ" to extract the key values from the YAML file of the *"Properties.yaml"* and store them as a tekton task result:

    ```YAML
    apiVersion: tekton.dev/v1beta1
    kind: Task
    metadata:
    name: get-pipeline-config
    spec:
    results:
        - name: project-name
        description: the project name from the config file.
        - name: type
        description: The Type of the Repo (i.e Polyrepo or Monorepo)
        - name: build-main
        description: To build the main Conatainer (True Or False)
        - name: build-tests
        description: To build the Tests Conatainer (True Or False)
        - name: gitops-url
        description: The Full GitOps Repository Url
        - name: gitops-applicationpath
        description: The Full GitOps Repository Url
    steps:
        - name: get-pipeline-config
        image: 'quay.io/thason/pipeline-tools:latest'
        args:
            - '-c'
            - >
            set -ex

            yq eval '.pipelineConfig.Project.Name' $(workspaces.input.path)/.Pipeline/Properties.yaml | tee $(results.project-name.path)

            yq eval '.pipelineConfig.Type' $(workspaces.input.path)/.Pipeline/Properties.yaml | tee $(results.type.path)

            yq eval '.pipelineConfig.Build.Main' $(workspaces.input.path)/.Pipeline/Properties.yaml | tee $(results.build-main.path)

            yq eval '.pipelineConfig.Build.Tests' $(workspaces.input.path)/.Pipeline/Properties.yaml | tee $(results.build-tests.path)

            yq eval '.pipelineConfig.GitOps.RepoURL' $(workspaces.input.path)/.Pipeline/Properties.yaml | tee $(results.gitops-url.path)

            yq eval '.pipelineConfig.GitOps.ApplicationPath' $(workspaces.input.path)/.Pipeline/Properties.yaml | tee $(results.gitops-applicationpath.path)

        command:
            - /bin/bash
        resources: {}
    workspaces:
        - description: The git repo will be read from this workspace
        name: input        
    ```

    In the task the workspace that we create in the pipelineRun will be transfered to the *get-pipeline-config* task and then runs on all the known keys and the file and set them to their related results.

4. The Last task is Just a validation Tasks to Check that our results are accessible in other tasks and we can use them.

    ```YAML
    apiVersion: tekton.dev/v1beta1
    kind: Task
    metadata:
    name: print-pipeline-config
    spec:
    params:
        - name: project-name
        description: The Project Name from the Config
        type: string
        - name: project-type
        description: The Project type from the Config
        type: string
        - name: build-main
        description: Parameter for Build main Repo image
        type: string
        - name: build-tests
        description: Parameter for Build tests Repo image
        type: string
        - name: gitops-url
        description: the GitOps repo Full URL
        type: string
        - name: gitops-applicationpath
        description: the GitOps repo application Path
        type: string
    steps:
        - name: get-results-from-pipeline
        image: 'quay.io/thason/pipeline-tools:latest'
        args:
            - '-c'
            - >
            set -x

            echo "The Project Name from the Config: $(params.project-name)"

            echo "The Project type from the Config: $(params.project-type)"
            
            echo "Parameter for Build main Repo image: $(params.build-main)"

            echo "Parameter for Build tests Repo image: $(params.build-tests)"

            echo "the GitOps repo Full URL: $(params.gitops-url)"

            echo "the GitOps repo application Path: $(params.gitops-applicationpath)"

        command:
            - /bin/bash
        resources: {}
    ```

    This simple task just prints the Params values of the task.

5. In our Pipeline we will map to each Parameter object in the task the tasts results, as follows:

    | indicats the tasks | Name of the Task | result of the task | the name of the result as set in the Task |
    |--------------------|------------------|--------------------|-------------------------------------------|
    |tasks|get-pipeline-config|results|gitops-applicationpath|

    ```YAML
    apiVersion: tekton.dev/v1
    kind: Pipeline
    metadata:
    name: get-results
    spec:
    tasks:
        - name: git-clone
        params:
            - name: url
            value: 'https://github.com/tal-hason/tekton-pipeline-with-result.git'
            - name: revision
            value: main
            - name: refspec
            value: ''
            - name: submodules
            value: 'true'
            - name: depth
            value: '1'
            - name: sslVerify
            value: 'true'
            - name: crtFileName
            value: ca-bundle.crt
            - name: subdirectory
            value: ''
            - name: sparseCheckoutDirectories
            value: ''
            - name: deleteExisting
            value: 'true'
            - name: httpProxy
            value: ''
            - name: httpsProxy
            value: ''
            - name: noProxy
            value: ''
            - name: verbose
            value: 'true'
            - name: gitInitImage
            value: >-
                gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/git-init:v0.40.2
            - name: userHome
            value: /home/git
        taskRef:
            kind: Task
            name: git-clone
        workspaces:
            - name: output
            workspace: repository
        - name: get-pipeline-config
        runAfter:
            - git-clone
        taskRef:
            kind: Task
            name: get-pipeline-config
        workspaces:
            - name: input
            workspace: repository
        - name: print-pipeline-config
        params:
            - name: project-name
            value: $(tasks.get-pipeline-config.results.project-name)
            - name: project-type
            value: $(tasks.get-pipeline-config.results.type)
            - name: build-main
            value: $(tasks.get-pipeline-config.results.build-main)
            - name: build-tests
            value: $(tasks.get-pipeline-config.results.build-tests)
            - name: gitops-url
            value: $(tasks.get-pipeline-config.results.gitops-url)
            - name: gitops-applicationpath
            value: $(tasks.get-pipeline-config.results.gitops-applicationpath)
        runAfter:
            - get-pipeline-config
        taskRef:
            kind: Task
            name: print-pipeline-config
    workspaces:
        - name: repository
    ```
