# How to create and run the GitLab CI/CD Pipeline for .NET Core Web application Deployment on IIS

##Setup
1. Gitlab on premise account e.g https://gitlab.xyz.com (Consider the GitLab is installed on Ubuntu Server/Desktop instance)
2. GitLab Runner : https://docs.gitlab.com/runner/install/windows/
   - Download GitLab Runner from the above link for Windows x64.
   - Extract the .zip file and keep the **gitlab-runner.exe** on the desired location on your computer e.g. D:\Gitlab\gitlab-runner.exe
   NOTE : GitLab Runner is required to run the pipeline.
3. Now go to your GitLab Repository, On left side panel look for **Settings > CI/CD > Runners**.
4. Click on **New Project Runner** button.
5. In the **Tags** add desired name for the runner tag **e.g windows**
6. Check **Run Untagged Jobs** and click on **Create Runner**
7. On next page select the **Windows** as operating system (As our gitlab runner will run on Windows) & copy the **runner authentication token** which looks like --> glrt-t3_Q6xuqabvNynfYKsxE-b2 

## Registering the GitLab repository pipeline to the gitlab-runner
1. Go the location where the gitlab-runner.exe file is saved e.g D:\Gitlab\gitlab-runner.exe
2. Open the command prompt with Administrator privileges & follow the below steps
   - run gitlab-runner.exe register command
   - Enter the Gitlab url in our case it is **https://gitlab.xyz.com**
   - Enter the desired name for the runner e.g "my-runner"
   - Next it will prompt to enter the executor name. In our case we will use **shell** executor as we are working on Windows.
   - Finally a new file will be created next to gitlab-runner.exe with name "config.toml"
   - Open the file with the notepad and replace "pwsh" with "powershell", save the file and restart the gitlab-runner using command **gitlab-runner.exe restart**
   - This will restart the service by applying the changes in the config.toml file.
 
## Overview
This GitLab CI/CD pipeline automates the build, testing, and deployment of a .NET Core website. The pipeline consists of three stages:
1. **Build** - Restores dependencies and compiles the application.
2. **Test** - Runs unit tests to ensure code quality.
3. **Deploy** - Publishes the application to an FTP server and restarts the IIS application pool.

## Pipeline Configuration

### Stages
```yaml
stages:
  - build
  - test
  - deploy
```

### Variables
The pipeline uses environment variables for configuration, ensuring flexibility and security:
- `DOTNET_VERSION`: Specifies the .NET version (e.g., `6.0`).
- `PROJECT_NAME`: The name of the project.
- `PUBLISH_DIR`: The output directory for published files.
- `FTP_SERVER`: The FTP server address.
- `FTP_USER`: FTP username.
- `FTP_PASSWORD`: FTP password (use GitLab CI/CD variables for security).
- `FTP_DEPLOY_PATH`: The target path on the FTP server.
- `APP_POOL_NAME`: The IIS application pool name.

### Before Script
Before executing any stage, the pipeline confirms the .NET version:
```yaml
before_script:
  - echo "Using .NET Core version $DOTNET_VERSION"
  - dotnet --version
```

## Build Stage
Compiles the project and creates the deployment package:
```yaml
build:
  stage: build
  image: mcr.microsoft.com/dotnet/sdk:$DOTNET_VERSION
  script:
    - dotnet restore
    - dotnet publish -c Release -o $PUBLISH_DIR
  artifacts:
    paths:
      - $PUBLISH_DIR
```

## Test Stage
Runs unit tests and stores test results:
```yaml
unit_tests:
  stage: test
  image: mcr.microsoft.com/dotnet/sdk:$DOTNET_VERSION
  script:
    - dotnet test --logger trx
  artifacts:
    when: always
    paths:
      - TestResults/
```

## Deploy Stage

### Overview
The deploy stage is responsible for transferring the published .NET Core application to a remote server via FTP and restarting the IIS application pool to apply the changes.

### Prerequisites
Before using the deploy stage, ensure the following:
- The FTP server is set up and accessible with valid credentials.
- The Windows server hosting IIS allows remote PowerShell execution.
- The IIS application pool name matches the configured variable (`APP_POOL_NAME`).
- WinSCP is installed on the deployment server for FTP file transfers.
- The deployment script has the necessary permissions to stop and start the IIS application pool.

### Deployment Steps
1. **Stop the IIS Application Pool**
   - This prevents access to the application while updating files.
   - The script establishes a PowerShell remote session to stop the IIS application pool.
   
2. **Upload Files via FTP**
   - Uses WinSCP to transfer the published files to the remote FTP server.
   - The pipeline generates a temporary script for WinSCP to execute the file transfer.
   
3. **Restart the IIS Application Pool**
   - Once the deployment is complete, the application pool is restarted.
   - This allows the updated website to be served to users.

```yaml
deploy:
  stage: deploy
  image: mcr.microsoft.com/powershell
  before_script:
    - echo "Deploying to FTP Server: $FTP_SERVER"
  script:
    - |
      echo "Stopping IIS Application Pool: $APP_POOL_NAME"
      $securePassword = ConvertTo-SecureString "YourPassword" -AsPlainText -Force
      $credential = New-Object System.Management.Automation.PSCredential ("Administrator", $securePassword)
      Invoke-Command -ComputerName "YourServerIP" -Credential $credential -ScriptBlock {
          Import-Module WebAdministration
          Stop-WebAppPool -Name "$APP_POOL_NAME"
      }
    - |
      # Define WinSCP path
      $winscpPath = "C:\\Program Files (x86)\\WinSCP\\winscp.com"
      
      # Build the WinSCP command script
      $winscpCommand = @"
      open ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_SERVER}
      put "${CI_PROJECT_DIR}\${PUBLISH_DIR}\*" "${FTP_DEPLOY_PATH}/"
      exit
      "@
      
      # Save the command script to a temporary file
      $tempFile = [System.IO.Path]::GetTempFileName()
      Set-Content -Path $tempFile -Value $winscpCommand
      
      # Execute WinSCP with the command file
      & $winscpPath /script=$tempFile
      
      # Remove the temporary file
      Remove-Item $tempFile
    - |
      echo "Starting IIS Application Pool: $APP_POOL_NAME"
      Invoke-Command -ComputerName "YourServerIP" -Credential $credential -ScriptBlock {
          Import-Module WebAdministration
          Start-WebAppPool -Name "$APP_POOL_NAME"
      }
  only:
    - main  # Deploy only from the main branch
```

## Summary
- The pipeline automates the process from build to deployment.
- Uses GitLab CI/CD variables for configuration security.
- FTP deployment is handled via WinSCP.
- IIS app pool is restarted for smooth deployment.
- Ensures code quality through automated testing.

This pipeline ensures a streamlined deployment process for .NET Core applications.

