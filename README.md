GitLab CI/CD Pipeline for .NET Core Website Deployment
**Overview**
This GitLab CI/CD pipeline automates the build, testing, and deployment of a .NET Core website. The pipeline consists of three stages:
1.	Build - Restores dependencies and compiles the application.
2.	Test - Runs unit tests to ensure code quality.
3.	Deploy - Publishes the application to an FTP server and restarts the IIS application pool.
**Pipeline Configuration**
**Stages**
stages:
  - build
  - test
  - deploy
**Variables**
The pipeline uses environment variables for configuration, ensuring flexibility and security:
•	DOTNET_VERSION: Specifies the .NET version (e.g., 6.0).
•	PROJECT_NAME: The name of the project.
•	PUBLISH_DIR: The output directory for published files.
•	FTP_SERVER: The FTP server address.
•	FTP_USER: FTP username.
•	FTP_PASSWORD: FTP password (use GitLab CI/CD variables for security).
•	FTP_DEPLOY_PATH: The target path on the FTP server.
•	APP_POOL_NAME: The IIS application pool name.
**Before Script**
Before executing any stage, the pipeline confirms the .NET version:
before_script:
  - echo "Using .NET Core version $DOTNET_VERSION"
  - dotnet --version
**Build Stage**
Compiles the project and creates the deployment package:
build:
  stage: build
  image: mcr.microsoft.com/dotnet/sdk:$DOTNET_VERSION
  script:
    - dotnet restore
    - dotnet publish -c Release -o $PUBLISH_DIR
  artifacts:
    paths:
      - $PUBLISH_DIR
**Test Stage**
Runs unit tests and stores test results:
unit_tests:
  stage: test
  image: mcr.microsoft.com/dotnet/sdk:$DOTNET_VERSION
  script:
    - dotnet test --logger trx
  artifacts:
    when: always
    paths:
      - TestResults/
**Deploy Stage**
**Overview**
The deploy stage is responsible for transferring the published .NET Core application to a remote server via FTP and restarting the IIS application pool to apply the changes.
**Prerequisites**
Before using the deploy stage, ensure the following:
•	The FTP server is set up and accessible with valid credentials.
•	The Windows server hosting IIS allows remote PowerShell execution.
•	The IIS application pool name matches the configured variable (APP_POOL_NAME).
•	WinSCP is installed on the deployment server for FTP file transfers.
•	The deployment script has the necessary permissions to stop and start the IIS application pool.
**Deployment Step**s
1.	Stop the IIS Application Pool 
o	This prevents access to the application while updating files.
o	The script establishes a PowerShell remote session to stop the IIS application pool.
2.	Upload Files via FTP 
o	Uses WinSCP to transfer the published files to the remote FTP server.
o	The pipeline generates a temporary script for WinSCP to execute the file transfer.
3.	Restart the IIS Application Pool 
o	Once the deployment is complete, the application pool is restarted.
o	This allows the updated website to be served to users.
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
**Summary**
•	The pipeline automates the process from build to deployment.
•	Uses GitLab CI/CD variables for configuration security.
•	FTP deployment is handled via WinSCP.
•	IIS app pool is restarted for smooth deployment.
•	Ensures code quality through automated testing.
This pipeline ensures a streamlined deployment process for .NET Core applications.
