When deploying a compiled `.zip` artifact to a remote physical or virtual IIS server (whether it is on-premise or an Azure VM), you will run into **file locking conflicts** if the application is actively serving traffic.

The industry best practice to achieve **minimal downtime** and ensure file locks are released gracefully without doing an aggressive, server-wide `iisreset` is utilizing an **`app_offline.htm` shadow file** paired with targeting an Azure DevOps **Self-Hosted Agent or Deployment Group** on that server.

Here is the step-by-step modular strategy to add to your centralized deployment templates.

---

## The Zero-Lock Strategy

Instead of stopping the whole World Wide Web Publishing Service (`w3svc`), we drop a literal file named `app_offline.htm` into the root folder of your web directory.

* **For .NET:** The IIS ASP.NET Core module intercepts this instantly, stops accepting new requests, gracefully finishes in-flight requests, freezes the worker process pool, and releases all locks on binaries (`.dll` files).
* **For Node.js (iisnode):** It allows you to safely stop the Node background process or recycle the App Pool without error codes.

While this file exists, visitors see a fast, lightweight "Under Maintenance" page. The instant you delete the file, the app pool wakes up automatically.

---

## Step 1: Update the Central Deployment Template

We will update your shared `jobs/universal-package-deploy.yml` template to include a step that runs directly on the remote machine using a **PowerShell Target** or an agent running on that target pool.

Here is how to extract your unzipped artifact directly into the IIS live path safely:

```yaml
# Add this task sequence inside your central deployment template
# targeting your Windows/IIS Environment machine.

parameters:
  - name: iisWebsitePath
    type: string
    default: 'C:\inetpub\wwwroot\my-app'
  - name: iisAppPoolName
    type: string
    default: 'MyAppPool'

steps:
# 1. Gracefully shut down the app domain & display a maintenance layout to incoming traffic
- powershell: |
    $targetPath = "${{ parameters.iisWebsitePath }}"
    if (!(Test-Path $targetPath)) { New-Item -ItemType Directory -Path $targetPath -Force }
    
    # Create the app_offline interceptor file
    $htmlContent = "<html><body style='font-family:sans-serif;text-align:center;padding:100px;'><h2>System Update In Progress</h2><p>We are applying a quick update. This page will refresh automatically.</p><script>setInterval(() => { fetch('/').then(r => { if(r.status===200) location.reload(); }) }, 3000);</script></body></html>"
    Set-Content -Path "$targetPath\app_offline.htm" -Value $htmlContent
    
    Write-Host "Dropped app_offline.htm. Waiting 5 seconds for worker processes to release file locks..."
    Start-Sleep -Seconds 5
  displayName: 'Graceful IIS Stop (Drop app_offline)'

# 2. Extract and overwrite the new contents safely
- task: ExtractFiles@1
  displayName: 'Extract Universal Package Content directly to IIS Site'
  inputs:
    archiveFilePattern: '$(Pipeline.Workspace)/**/*-pkg/app.zip' # Grabs unzipped package matching previous steps
    destinationFolder: '${{ parameters.iisWebsitePath }}'
    cleanDestinationFolder: false # Set to false so we don't accidentally wipe log directories or the app_offline file mid-flight

# 3. Recycle the app pool to completely clear memory leaks and pick up fresh environment adjustments
- powershell: |
    Import-Module WebAdministration
    $pool = "${{ parameters.iisAppPoolName }}"
    
    Write-Host "Recycling Application Pool: $pool"
    Restart-WebAppPool -Name $pool
  displayName: 'Recycle IIS Application Pool'

# 4. Bring the application back online
- powershell: |
    $offlineFile = "${{ parameters.iisWebsitePath }}\app_offline.htm"
    if (Test-Path $offlineFile) {
        Remove-Item -Path $offlineFile -Force
        Write-Host "Removed app_offline.htm. The site is now LIVE."
    }
  displayName: 'Bring Site Online (Remove app_offline)'

```

---

## 💡 Pro-Tips for Zero Downtime Optimization

If your service requirements dictate **true zero-downtime** (less than 1 second of interruption), `app_offline.htm` isn't enough on its own since users will still see a maintenance message for a brief window. To get around this entirely, adopt one of these structural architectures:

### Approach A: The Side-by-Side Directory Swap (Highly Recommended)

Instead of overwriting files in `C:\inetpub\wwwroot\my-app`, you deploy to timestamped directories and update the IIS pointer via script:

1. Extract the zip to a fresh path: `C:\inetpub\wwwroot\build-1.0.45`
2. Run a post-deployment script to alter the physical path component inside IIS manager:
```powershell
Set-ItemProperty "IIS:\Sites\MyWebSite" -Name physicalPath -Value "C:\inetpub\wwwroot\build-1.0.45"

```


3. Recycle the app pool. The cutover is atomic and immediate.

### Approach B: IIS Application Request Routing (ARR) / Blue-Green

If your server has a load balancer or Application Request Routing (ARR) enabled locally:

1. Bind your app to two ports (e.g., Port `8081` [Blue] and Port `8082` [Green]).
2. Deploy your zip package to the offline port, run health-check smoke tests locally against it, and then instruct ARR or your reverse proxy to swing production traffic over to the upgraded port.

```

```
