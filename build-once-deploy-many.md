To achieve this, the best practice is to separate your setup into **two distinct YAML pipelines**:

1. **The Build Pipeline (`azure-pipelines-build.yml`)**: Compiles/gathers your code, zips it up, increments the version automatically (using SemVer format like `1.0.buildNumber`), and pushes it to your Azure Artifacts feed.
2. **The CD/Release Pipeline (`azure-pipelines-deploy.yml`)**: A generic, parameter-driven pipeline that downloads that specific Universal Package and deploys it to whatever environment you pass in (Dev, QA, Prod).

Before copying the code below, make sure you have created an **Azure Artifacts Feed** in your project to act as your package repository.

---

## 1. The Build & Publish Pipeline

This pipeline automatically sets a Semantic Versioning scheme (`1.0.$(Build.BuildId)`). Universal Packages **require** a strict `major.minor.patch` version structure.

Notice that the `UniversalPackages@0` task takes a directory, not a raw file. We zip your files first, put that `.zip` file inside a staging directory, and publish *that directory* as the Universal Package.

```yaml
# azure-pipelines-build.yml
trigger:
  - main

# Formats the build number as the 'patch' version (e.g., 1.0.142)
name: 1.0.$(Rev:r)

pool:
  vmImage: 'ubuntu-latest'

variables:
  # The exact name your package will have in Azure Artifacts
  packageName: 'my-app-package'
  # Change this to your Azure DevOps Project Feed Name
  feedName: 'MyProjectArtifactsFeed' 

steps:
- task: ArchiveFiles@2
  displayName: 'Zip application files'
  inputs:
    rootFolderOrFile: '$(System.DefaultWorkingDirectory)' # Or your specific build output folder
    includeRootFolder: false
    archiveType: 'zip'
    archiveFile: '$(Build.ArtifactStagingDirectory)/package-files/app.zip'
    replaceExistingArchive: true

- task: UniversalPackages@0
  displayName: 'Publish Universal Package to Feed'
  inputs:
    command: 'publish'
    publishDirectory: '$(Build.ArtifactStagingDirectory)/package-files'
    vstsFeedPublish: '$(feedName)'
    vstsFeedPackagePublish: '$(packageName)'
    versionOption: 'custom'
    versionPublish: '$(Build.BuildNumber)' # Uses the 1.0.X version generated above
    packagePublishDescription: 'Build package from commit $(Build.SourceVersion)'

```

---

## 2. The Generic Deployment Pipeline

This pipeline uses **Runtime Parameters**. When you trigger it manually, it will prompt you with a dropdown for the environment and a text box to specify exactly which package version you want to deploy.

```yaml
# azure-pipelines-deploy.yml
trigger: none # Disables automatic triggers so it can be run manually/generically

parameters:
  - name: targetEnvironment
    displayName: 'Target Environment'
    type: string
    default: 'Development'
    values:
      - Development
      - QA
      - Staging
      - Production

  - name: packageVersion
    displayName: 'Universal Package Version to Deploy'
    type: string
    default: 'latest' # You can type a specific version like 1.0.12

pool:
  vmImage: 'ubuntu-latest'

variables:
  packageName: 'my-app-package'
  feedName: 'MyProjectArtifactsFeed'

stages:
- stage: Deploy
  displayName: 'Deploy to ${{ parameters.targetEnvironment }}'
  jobs:
  - deployment: RunDeployment
    # Connects to Azure DevOps Environments for tracking, approvals, and checks
    environment: '${{ parameters.targetEnvironment }}' 
    strategy:
      runOnce:
        deploy:
          steps:
          - task: UniversalPackages@0
            displayName: 'Download Universal Package'
            inputs:
              command: 'download'
              downloadDirectory: '$(Pipeline.Workspace)/downloaded-package'
              vstsFeed: '$(feedName)'
              vstsPackage: '$(packageName)'
              vstsnvMode: 'version'
              versionDownload: '${{ parameters.packageVersion }}'

          - task: ExtractFiles@1
            displayName: 'Unzip Package Content'
            inputs:
              archiveFilePattern: '$(Pipeline.Workspace)/downloaded-package/app.zip'
              destinationFolder: '$(Pipeline.Workspace)/extracted-app'
              cleanDestinationFolder: true

          # Replace this script step with your actual deployment logic 
          # (e.g., AzureWebAppDeployment, CopyFiles to server, AWS deploy, etc.)
          - script: |
              echo "Deploying to environment: ${{ parameters.targetEnvironment }}"
              echo "Looking at extracted files in $(Pipeline.Workspace)/extracted-app"
              ls -la $(Pipeline.Workspace)/extracted-app
            displayName: 'Execute Deployment Script'

```

---

## How to use this setup:

1. **Run the Build pipeline:** This packages up your code and increments the version in Azure Artifacts automatically. Take note of the version number generated (e.g., `1.0.4`).
2. **Run the Deployment pipeline:** Click "Run Pipeline", select your environment (e.g., `QA`), type in `1.0.4` (or leave it as `latest`), and hit run.

> **Permission Note:** If your pipelines run into authentication errors while pushing or pulling packages, make sure your Project's Build Service has **Contributor** rights inside your Azure Artifact Feed settings.
