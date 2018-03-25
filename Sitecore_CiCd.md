# How to create Continuous Integration / Continuous Deployment (CI/CD) for Sitecore 8

## 1. Setup [TeamCity](https://www.jetbrains.com/teamcity/)

### 1.1 Setup build steps:

**Step 1: Notify Slack channel for start** 

If you are using Slack, you could add Slack notification for your team. Here you have two options:

- first write custom PowerShell script;
- second install some TeamCity plugin to help you with this notifications;

I tried some of the most popular plugins, but didn't work out for me and after some discussions decided to keep it simple and wrote a small PowerShell script. 

*Here is an example*:

```PowerShell
$slackToken = "your-slack-token"
$slackChannel = "#your-slack-channel"
$slackText = "@channel :smile: Start Build & Deploy *%build.number%*"
$slackUserName = "TeamCity"
$slackIcon = "https://pbs.twimg.com/profile_images/880014590427496448/fducIHLi_400x400.jpg"

$postSlackMessage = @{token=$slackToken; channel=$slackChannel; text=$slackText; username=$slackUserName; icon_url=$slackIcon}

Invoke-RestMethod -Uri https://slack.com/api/chat.postMessage -Body $postSlackMessage
```

**Step 2: Restore NuGet packages**

What you should do before you try to build? Yes, you should **restore NuGet packages**. 

1. Choose Runner Type: NuGet Installer.
2. Path to solution file.
3. If you didn't specify NuGet packages source in the solution you could add the following lines in **Packages sources** field:
    - https://api.nuget.org/v3/index.json
    - https://sitecore.myget.org/F/sc-packages - this is Sitecore NuGet, from where you could reference desired packages with Noreference type. *Keep in mind that sometimes it's unavailable and your build could fail, because it's not able to download used packages. For this reason I use **"Restore"** which cache downloaded packages in the build machine and use them instead of downloading again all of them.*
4. Restore mode: Restore

**Step 3: Visual Studio build**

1. Runner type: "Visual Studio (sln)"
2. Step name - specify some descriptive name like "Visual Studio build"
3. Select path to solution file from repository
4. Targets - enter targets separated by space or semicolon. Build, Rebuild, Clean, Publish targets are supported by default
5. Configuration - Release 
6. Platform - Any CPU

*You could use "Build Features" AssemblyInfo patcher to change dlls info. You could setup it from "Build Features". Make sure that every AssemblyInfo.cs file has the following attributes:*

```C#
[assembly: AssemblyVersion("1.0.0")]
[assembly: AssemblyFileVersion("1.0.0")]
[assembly: AssemblyInformationalVersion("1.0.0")]
```

**Step 4: Run unit tests**

1. Runner type: NUnit
2. Step name
3. NUnit runner: NUnit 3
4. Run test from: path to your test project

**What's next?**

Before I proceed with next steps I would like to explain what is the final goal. 

In Sitecore world we have different "Servers" - [configuring servers](https://doc.sitecore.net/sitecore_experience_platform/81/setting_up_and_maintaining/xdb/configuring_servers/configuring_servers).
- Content delivery server
- Content management server
- Processing/aggregation server
- Reporting Service server
- Collection database server
- Reporting database server
- Session database server 

In my case I'll use simple configuration with 2 Content delivery server, 1 Content management server, 1 MS SQL Server. 

After build & deployment I should have 2 Content delivery servers, 1 Content management server and content will be deployed and synced with [Team Development for Sitecore (TDS)](https://www.teamdevelopmentforsitecore.com/). 

To be able to do this during the build I'll prepare 1 artifact for content delivery server (CDS/CD), 1 artifact for content management server (CMS/CM) and 1 or more TDS packages which will contain our content. Why I need to do this? I must have different artifacts/packages which will deployed on different environments - CDS/CD configuration and CMS/CM configuration. 

**Step 5: Publish solution**

I must publish the solution with Release profile to prepare the files which I'll use to create CDS and CMS artifacts.

1. Runner type: Visual Studio (sln)
2. Step name: Publish solution
3. Solution file path
4. Targets: Build
5. Configuration: Release
6. Command line parameters

```PowerShell
/p:DeployOnBuild=true
/p:PublishProfile=%teamcity.build.checkoutDir%\MyProject\Properties\PublishProfiles\Release.pubxml
```

Here are details of my publish profile:

```XML
<Project ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <PropertyGroup>
    <WebPublishMethod>FileSystem</WebPublishMethod>
    <PublishProvider>FileSystem</PublishProvider>
    <LastUsedBuildConfiguration>Release</LastUsedBuildConfiguration>
    <LastUsedPlatform>Any CPU</LastUsedPlatform>
    <SiteUrlToLaunchAfterPublish />
    <LaunchSiteAfterPublish>False</LaunchSiteAfterPublish>
    <ExcludeApp_Data>False</ExcludeApp_Data>
    <publishUrl>$(MSBuildThisFileDirectory)..\..\..\..\MyProject.Published</publishUrl>
    <DeleteExistingFiles>False</DeleteExistingFiles>
  </PropertyGroup>
</Project>
```

As you can see after this step finish successfully it will produce this folder **MyProject.Published**. I will use the content of this folder to create CDS and CMS packages.

**Step 6: Copy CDS/CMS folder**

1. Runner type: PowerShell

I will use this step and PowerShell to prepare the CDS and CMS artifacts. Here is my PowerShell script:

```PowerShell
$pathToCds = "MyProject.CDS"
$pathToCms = "MyProject.CMS"
$pathToPublish = "MyProject.Published\*"

# Create folders
New-Item -ItemType Directory -Force -Path $pathToCds
Write-Output("Folder $pathToCds created.")

New-Item -ItemType Directory -Force -Path $pathToCms
Write-Output("Folder $pathToCms created.")

# Copy publish folders content
Copy-Item $pathToPublish -Recurse $pathToCds 
Write-Output("Folder $pathToPublish copied to $pathToCds.")

Copy-Item $pathToPublish -Recurse $pathToCms
Write-Output("Folder $pathToPublish copied to $pathToCms.")
```

You could also use this step to copy some plugins files which will be necessary for proper work of some of installed plugins. You will need to have those plugins in CI/CD pipeline, because you don't want to install them after every build & deployment. Example of Sitecore Powershell plugin which will be added only in CMS folder:

```PowerShell
# Copy Sitecore.PowerShell.Extensions-4.3.for.Sitecore.8 dlls only on CMS bin
$SitecorePowerShell  = "Plugins\SitecorePowerShell\dll\*"
if ((Test-Path -Path $SitecorePowerShell) -eq $false) 
{
    Write-Error "Switch: Could not find $SitecorePowerShell"
	Exit 1
}
else {    
    Copy-item -Force -Recurse -Verbose $SitecorePowerShell -Destination $pathToPublishBin
    Write-Output("Sitecore.PowerShell.Extensions dlls copied to $pathToCms\bin.")
}

# Copy Sitecore PowerShell Extensions for Sitecore dlls only on CMS bin
$SitecorePowerShell  = "Plugins\SitecorePowerShell\dll\*"
if ((Test-Path -Path $SitecorePowerShell) -eq $false) 
{
    Write-Error "Switch: Could not find $SitecorePowerShell"
	Exit 1
}
else {    
    Copy-item -Force -Recurse -Verbose $SitecorePowerShell -Destination $pathToPublishBin
    Write-Output("Sitecore.PowerShell.Extensions dlls copied to $pathToCms\bin.")
}

# Copy Sitecore PowerShell Extensions for Sitecore files to sitecore
$SitecorePowerShellFiles1  = "Plugins\SitecorePowerShell\sitecore\shell\Themes\Standard\*"
if ((Test-Path -Path $SitecorePowerShellFiles1) -eq $false) 
{
    Write-Error "Switch: Could not find $SitecorePowerShellFiles1"
	Exit 1
}
else {    
    Copy-item -Force -Recurse -Verbose $SitecorePowerShellFiles1 -Destination "$pathToCms\sitecore\shell\Themes\Standard"
    Write-Output("Sitecore.PowerShell.Extensions files copied to $pathToCms\sitecore\shell\Themes\Standard.") 
}

# Copy Sitecore PowerShell Extensions for Sitecore files to sitecore modules
$SitecorePowerShellFiles2  = "Plugins\SitecorePowerShell\sitecore modules\*"
if ((Test-Path -Path $SitecorePowerShellFiles2) -eq $false) 
{
    Write-Error "Switch: Could not find $SitecorePowerShellFiles2"
	Exit 1
}
else {    
    Copy-item -Force -Recurse -Verbose SitecorePowerShellFiles2 -Destination "$pathToCms\sitecore modules"
    Write-Output("Sitecore.PowerShell.Extensions files copied to $pathToCms\sitecore modules.")
}
```


## 2. Setup [Octopus Deploy](https://octopus.com/)

## Tools:
--
- Octopus Deploy
- TeamCity
- Sitecore.Ship
- Sitecore 8

## Links and materials:
--
https://blog.wesleylomax.co.uk/2016/04/19/continuous-delivery-sitecore-tds-git-team-city-octopus-sitecore-ship-part-2/

http://goblinrockets.com/2014/01/16/update-continuous-integration-deployment-sitecore/

https://thesoftwaredudeblog.wordpress.com/2016/07/11/building-a-continuous-integration-environment-for-sitecore-part-10-config-transformations-using-octopus-deploy/

https://naveed-ahmad.com/2016/12/07/sitecore-devop-series-part-1-continuous-integration-why-your-sitecore-project-deployments-must-be-autmated/

Sitecore configuration roles to help with environment configuration:
https://github.com/Sitecore/Sitecore-Configuration-Roles - didn't work as expected

Update package content:
https://github.com/kevinobee/Sitecore.Ship    
