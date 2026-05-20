To make this scales cleanly across multiple projects and repositories, we will use a **Centralized Template Repository** pattern.

You will create one dedicated Git repository (e.g., named `pipeline-templates`) to hold all your core build and logic engines. Every other application repository will simply reference this central repo, keeping your individual project configurations incredibly lean and easy to maintain.

---

## 1. Central Template Repository (`pipeline-templates`)

Create a dedicated repository in your Azure DevOps organization. Structure it with modular steps so we can swap between `.NET` and `Node.js` processing seamlessly based on parameters.

### `steps/build-dotnet.yml`

This step template builds, packages, and prepares a .NET application.

```yaml
# steps/build-dotnet.yml
parameters:
  - name: projectPath
    type: string
  - name: stagingDirectory
    type: string

steps:
- task: UseDotNet@2
  displayName: 'Install .NET SDK'
  inputs:
    version: '8.x' # Update to your target version

- task: DotNetCoreCLI@2
  displayName: 'Restore & Build .NET'
  inputs:
    command: 'build'
    projects: '${{ parameters.projectPath }}/**/*.csproj'
    arguments: '--configuration Release'

- task: DotNetCoreCLI@2
  displayName: 'Publish .NET Application'
  inputs:
    command: 'publish'
    publishWebProjects: false
    projects: '${{ parameters.projectPath }}/**/*.csproj'
    arguments: '--configuration Release --output ${{ parameters.stagingDirectory }}/package-contents'
    zipAfterPublish: false

```

### `steps/build-nodejs.yml`

This step template installs dependencies and builds a Node.js web application.

```yaml
# steps/build-nodejs.yml
parameters:
  - name: projectPath
    type: string
  - name: stagingDirectory
    type: string

steps:
- task: NodeTool@0
  displayName: 'Install Node.js'
  inputs:
    versionSpec: '20.x'

# Executed inside the specific subfolder via workingDirectory
- script: |
    npm ci
    npm run build --if-present
  workingDirectory: '${{ parameters.projectPath }}'
  displayName: 'Npm Install and Build'

- task: CopyFiles@2
  displayName: 'Stage Node.js Production Artifacts'
  inputs:
    sourceFolder: '${{ parameters.projectPath }}'
    # Copies your build outputs or production files (adjust contents filter as needed)
    contents: |
      dist/**
      build/**
      package.json
      package-lock.json
    targetFolder: '${{ parameters.stagingDirectory }}/package-contents'

```

### `jobs/universal-package-build.yml`

This orchestrator template determines whether it is dealing with a `.NET` or `Node.js` app, triggers the respective engine, zips the outcome, and pushes it to Azure Artifacts.

```yaml
# jobs/universal-package-build.yml
parameters:
  - name: appType
    type: string
    values:
      - dotnet
      - nodejs
  - name: subDirectory
    type: string # Expects 'frontend' or 'backend'
  - name: packageName
    type: string
  - name: feedName
    type: string

jobs:
- job: BuildAndPublish
  displayName: 'Build & Publish ${{ parameters.subDirectory }} (${{ parameters.appType }})'
  pool:
    vmImage: 'ubuntu-latest'
  
  variables:
    stagingDir: '$(Build.ArtifactStagingDirectory)/${{ parameters.subDirectory }}'

  steps:
  # 1. Conditionals dynamically insert the correct build flavor
  - ${{ if eq(parameters.appType, 'dotnet') }}:
    - template: ../steps/build-dotnet.yml
      parameters:
        projectPath: '$(System.DefaultWorkingDirectory)/${{ parameters.subDirectory }}'
        stagingDirectory: '$(stagingDir)'

  - ${{ if eq(parameters.appType, 'nodejs') }}:
    - template: ../steps/build-nodejs.yml
      parameters:
        projectPath: '$(System.DefaultWorkingDirectory)/${{ parameters.subDirectory }}'
        stagingDirectory: '$(stagingDir)'

  # 2. Compress the compiled code 
  - task: ArchiveFiles@2
    displayName: 'Zip Application Output'
    inputs:
      rootFolderOrFile: '$(stagingDir)/package-contents'
      includeRootFolder: false
      archiveType: 'zip'
      archiveFile: '$(stagingDir)/zipped/app.zip'
      replaceExistingArchive: true

  # 3. Ship as Universal Package using automated versioning string
  - task: UniversalPackages@0
    displayName: 'Publish Universal Package'
    inputs:
      command: 'publish'
      publishDirectory: '$(stagingDir)/zipped'
      vstsFeedPublish: '${{ parameters.feedName }}'
      vstsFeedPackagePublish: '${{ parameters.packageName }}'
      versionOption: 'custom'
      versionPublish: '$(Build.BuildNumber)'

```

---

## 2. Application Repositories

Inside **any** application repository across your company, your engineering teams now only need a very small, clean `azure-pipelines.yml` file.

They use the `resources` block to securely link back to your centralized template repository.

### Example Application Configuration

Imagine a repo with a **Node.js React Frontend** and a **.NET Core Backend API**.

```yaml
# [App Repo] /azure-pipelines.yml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - frontend/*
      - backend/*

# Master version format for Universal Packages Compatibility
name: 1.0.$(Rev:r)

resources:
  repositories:
    - repository: shared-templates
      type: git
      name: 'SharedPlatformProject/pipeline-templates' # Format: ProjectName/RepoName
      ref: refs/heads/main # Or a pinned version/tag like refs/tags/v1

variables:
  artifactFeed: 'CompanyGlobalFeed'

stages:
- stage: BuildStage
  displayName: 'Compile and Package'
  jobs:
  
  # Run the Node.js frontend packaging path
  - template: jobs/universal-package-build.yml@shared-templates
    parameters:
      appType: 'nodejs'
      subDirectory: 'frontend'
      packageName: 'retail-app-frontend'
      feedName: '$(artifactFeed)'

  # Run the .NET backend packaging path
  - template: jobs/universal-package-build.yml@shared-templates
    parameters:
      appType: 'dotnet'
      subDirectory: 'backend'
      packageName: 'retail-app-backend'
      feedName: '$(artifactFeed)'

```

---

## 💡 Key Architectural Benefits

* **Subdirectory Isolation:** By passing `frontend` or `backend` as the `subDirectory` parameter, the templates automatically contextualize their paths (`$(System.DefaultWorkingDirectory)/frontend`) without requiring multiple repositories per app.
* **Central Maintenance:** If you need to upgrade from .NET 8 to a newer version or swap out an optimization parameter in npm, you update it **once** inside the `pipeline-templates` repo, and every project pipeline across the organization inherits the update instantly.
* **Path Triggers:** The `paths` block filters at the top of the application pipeline ensure that changes to the frontend directory won't accidentally trigger unnecessary compute if you want to optimize pipeline runs later on.
