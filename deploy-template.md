To allow independent deployments where you can selectively deploy the **frontend**, **backend**, or **both** for a specific project/version, we will build a dedicated, **interactive deployment pipeline** inside your central template repository.

This uses a combination of Azure DevOps **Runtime Parameters** (for the UI selection) and **Environments** (to handle gates and manual approvals).

---

## Step 1: Add the Deployment Template to the Central Repo

Add this new file to your `pipeline-templates` repository. It accepts parameters from the user interface and dynamically builds deployment jobs for only the components selected.

### `jobs/universal-package-deploy.yml`

```yaml
# jobs/universal-package-deploy.yml
parameters:
  - name: targetEnvironment
    type: string
  - name: feedName
    type: string

  # Controls which applications are deployed
  - name: deployFrontend
    type: boolean
  - name: frontendPackageName
    type: string
  - name: frontendVersion
    type: string

  - name: deployBackend
    type: boolean
  - name: backendPackageName
    type: string
  - name: backendVersion
    type: string

stages:
- stage: DeployToEnv
  displayName: 'Deploying to ${{ parameters.targetEnvironment }}'
  jobs:

  # --- FRONTEND JOB ---
  - ${{ if eq(parameters.deployFrontend, true) }}:
    - deployment: DeployFrontendJob
      displayName: 'Deploy Frontend (${{ parameters.frontendVersion }})'
      pool:
        vmImage: 'ubuntu-latest'
      environment: '${{ parameters.targetEnvironment }}' # Triggers approvals configured in ADO
      strategy:
        runOnce:
          deploy:
            steps:
            - task: UniversalPackages@0
              displayName: 'Download Frontend Package'
              inputs:
                command: 'download'
                downloadDirectory: '$(Pipeline.Workspace)/frontend-pkg'
                vstsFeed: '${{ parameters.feedName }}'
                vstsPackage: '${{ parameters.frontendPackageName }}'
                vstsnvMode: 'version'
                versionDownload: '${{ parameters.frontendVersion }}'

            - task: ExtractFiles@1
              displayName: 'Unzip Frontend'
              inputs:
                archiveFilePattern: '$(Pipeline.Workspace)/frontend-pkg/app.zip'
                destinationFolder: '$(Pipeline.Workspace)/frontend-extracted'
                cleanDestinationFolder: true

            # Replace with your actual frontend infrastructure deployment task
            - script: echo "Deploying frontend files to ${{ parameters.targetEnvironment }}..."
              displayName: 'Execute Frontend Push'

  # --- BACKEND JOB ---
  - ${{ if eq(parameters.deployBackend, true) }}:
    - deployment: DeployBackendJob
      displayName: 'Deploy Backend (${{ parameters.backendVersion }})'
      pool:
        vmImage: 'ubuntu-latest'
      environment: '${{ parameters.targetEnvironment }}' # Triggers approvals configured in ADO
      strategy:
        runOnce:
          deploy:
            steps:
            - task: UniversalPackages@0
              displayName: 'Download Backend Package'
              inputs:
                command: 'download'
                downloadDirectory: '$(Pipeline.Workspace)/backend-pkg'
                vstsFeed: '${{ parameters.feedName }}'
                vstsPackage: '${{ parameters.backendPackageName }}'
                vstsnvMode: 'version'
                versionDownload: '${{ parameters.backendVersion }}'

            - task: ExtractFiles@1
              displayName: 'Unzip Backend'
              inputs:
                archiveFilePattern: '$(Pipeline.Workspace)/backend-pkg/app.zip'
                destinationFolder: '$(Pipeline.Workspace)/backend-extracted'
                cleanDestinationFolder: true

            # Replace with your actual backend infrastructure deployment task
            - script: echo "Deploying backend binary files to ${{ parameters.targetEnvironment }}..."
              displayName: 'Execute Backend Push'

```

---

## Step 2: The Interactive UI Pipeline (One Per Project)

In your specific project/application repository, create a `deploy-pipeline.yml`. This file defines the explicit UI configurations for that project's artifacts.

When you click **"Run Pipeline"**, Azure DevOps turns these parameters into form fields, checkboxes, and dropdowns.

```yaml
# [App Repo] /deploy-pipeline.yml
trigger: none # Purely manual interactive trigger

parameters:
  - name: env
    displayName: '1. Target Environment'
    type: string
    default: 'Development'
    values:
      - Development
      - QA
      - Production

  # Frontend Selections
  - name: deploy_fe
    displayName: '2. Deploy Frontend?'
    type: boolean
    default: true
  - name: fe_version
    displayName: '   ↳ Frontend Version (Ignore if unchecked)'
    type: string
    default: 'latest'

  # Backend Selections
  - name: deploy_be
    displayName: '3. Deploy Backend?'
    type: boolean
    default: true
  - name: be_version
    displayName: '   ↳ Backend Version (Ignore if unchecked)'
    type: string
    default: 'latest'

resources:
  repositories:
    - repository: shared-templates
      type: git
      name: 'SharedPlatformProject/pipeline-templates'
      ref: refs/heads/main

variables:
  artifactFeed: 'CompanyGlobalFeed'
  fePackage: 'retail-app-frontend'
  bePackage: 'retail-app-backend'

stages:
- template: jobs/universal-package-deploy.yml@shared-templates
  parameters:
    targetEnvironment: '${{ parameters.env }}'
    feedName: '$(artifactFeed)'
    
    deployFrontend: ${{ parameters.deploy_fe }}
    frontendPackageName: '$(fePackage)'
    frontendVersion: '${{ parameters.fe_version }}'
    
    deployBackend: ${{ parameters.deploy_be }}
    backendPackageName: '$(bePackage)'
    backendVersion: '${{ parameters.be_version }}'

```

---

## Step 3: Enforcing Approvals in Azure DevOps

To ensure that deployments stop and wait for human sign-off (especially for Production), you must configure the environment targets natively within your Azure DevOps Project UI.

1. **Navigate to Environments:** In Azure DevOps.
Go to **Pipelines** ➔ **Environments** inside your project web portal.


2. **Create target environments:** Match YAML inputs.
Click **New Environment** and create environments matching your names exactly: `Development`, `QA`, and `Production`.


3. **Add Approval Gates:** Protect the environment.
Click into an environment (like `Production`), select the **three dots menu (⋮)** in the upper right-hand corner, and choose **Approvals and checks**.


4. **Assign Approvers:** Final step.
Click **Approvals**, add the specific engineering leads or QA team members required to sign off, and click **Save**.


---

### How this works in practice:

1. **Granular Control:** If you only changed a typo on your web app, run this deployment pipeline, uncheck **Deploy Backend**, paste your frontend version number, and hit run. Only the frontend job will execute.
2. **Approval Handling:** When the pipeline encounters the environment parameter (e.g., `Production`), it halts execution immediately, flags the run as "Pending Approval," and pings your designated approvers via email/Teams before any steps run.
