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

You could also use this step to copy some plugin files which will be necessary for proper work of some of installed plugins. You will need to have those plugins in CI/CD pipeline, because you don't want to install them after every build & deployment. Example of Sitecore Powershell plugin which will be added only in CMS folder:

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

**Step 7: TDS package**

I'll use **NuGet Pack** to create an artifact with TDS update package. I'll give path to my NuGet spec file. Here is an example:

```XML
<?xml version="1.0"?>
<package
    xmlns="http://schemas.microsoft.com/packaging/2011/08/nuspec.xsd">
    <metadata>
        <id>MyProject.TDS.Master</id>
        <version>1.0.0</version>
        <title>MyProject.TDS.Master Update Package</title>
        <authors>MyProject</authors>
        <owners>MyProject</owners>
        <requireLicenseAcceptance>false</requireLicenseAcceptance>
        <description>Contains MyProject.TDS.Master.update package</description>
        <copyright>Copyright ©</copyright>
        <tags></tags>
    </metadata>
    <files>
        <file src="..\..\Sources\MyProject.TDS.Master\bin\Package_Release\MyProject.TDS.Master.update" />
    </files>
</package>
```

1. Version - %build.number%
2. Check publish created packages to build artifacts
3. Properties: Configuration=Release
4. Check "Create a tool package"

**Step 8: Clean up CDS folder**

Here agin I'll use PowerShell to clean up CDS folder from information which is not necessary in content deliver environment.

Script arguments: "%teamcity.build.workingDir%\MyProject.CDS"

```PowerShell
param
(
	[Parameter(Mandatory = $true)]
    [ValidateNotNullOrEmpty()]
    [String] $ProjectFolder = $(throw "PROJECT FOLDER parameter is required.")
)

Set-StrictMode -Version Latest

if ((Test-Path -Path $ProjectFolder) -eq $false)
{
    Write-Error "Could not find $ProjectFolder"
	Exit 1
}

Write-Output "Start tweaking content delivery site."

Write-Output "Connection: Removing connection strings ..."

$connectionConfigPath = "$ProjectFolder\App_Config\ConnectionStrings.config"

if ((Test-Path -Path $connectionConfigPath) -eq $false)
{
    Write-Error "Switch: Could not find $connectionConfigPath"
	Exit 1
}

Write-Output "Connection: Open file ..."

$connectionConfig = New-Object -TypeName System.Xml.XmlDocument
$connectionConfig.Load("$connectionConfigPath")

Write-Output "Connection: Removing master connection string ..."

$configKeyMaster = $connectionConfig.SelectSingleNode("/connectionStrings/add[@name = 'master']")

if ($null -ne $configKeyMaster) {
    $configKeyMaster.ParentNode.RemoveChild($configKeyMaster)

    Write-Output "Connection: Master database connection string removed ..."
}
else
{
    Write-Output "Connection: Master database connection string not found ..."
}

Write-Output "Connection: Removing reporting connection string ..."

$configKeyReporting = $connectionConfig.SelectSingleNode("/connectionStrings/add[@name = 'reporting']")

if ($null -ne $configKeyReporting) {
    $configKeyReporting.ParentNode.RemoveChild($configKeyReporting)

    Write-Output "Connection: Reporting database connection string removed ..."
}
else
{
    Write-Output "Connection: Reporting database connection string not found ..."
}

Write-Output "Connection: Removing tracking.history connection string ..."

$configKeyTracking = $connectionConfig.SelectSingleNode("/connectionStrings/add[@name = 'tracking.history']")

if ($null -ne $configKeyTracking) {
    $configKeyTracking.ParentNode.RemoveChild($configKeyTracking)

    Write-Output "Connection: Tracking.history database connection string removed ..."
}
else
{
    Write-Output "Connection: Tracking.history database connection string not found ..."
}

$connectionConfig.Save("$connectionConfigPath")

Write-Output "Connection: File saved ..."

Write-Output "Clean: Starting to clean BIN folder ..."

Write-Output "Clean: Removind App_Config folder ..."

Remove-Item -Path "$ProjectFolder\bin\App_Config" -Verbose -Force -ErrorAction SilentlyContinue

Write-Output "Clean: App_Config folder removed."

Write-Output "Ship: Start removing Sitecore.Ship.AspNet ..."

Write-Output "Ship: Deleting Ship.config ..."

Remove-Item -Path "$ProjectFolder\App_Config\Include\ship.config" -Verbose -Force -ErrorAction SilentlyContinue

Write-Output "Ship: Ship.config deleted."

Write-Output "Ship: Removing Web.config entries ..."

$webConfigPath = "$ProjectFolder\Web.config"

if ((Test-Path -Path $webConfigPath) -eq $false)
{
    Write-Error "Ship: Could not find $webConfigPath"
	Exit 1
}

Write-Output "Ship: Open Web.config file ..."

$webConfig = New-Object -TypeName System.Xml.XmlDocument
$webConfig.Load($webConfigPath)

Write-Output "Ship: File opened for edit."

Write-Output "Ship: Removing Web.config [packageInstallation] section entry ..."

$packageInstallationSectionNode = $webConfig.SelectSingleNode("/configuration/configSections/section[@name = 'packageInstallation']")

if ($null -ne $packageInstallationSectionNode) {
    $packageInstallationSectionNode.ParentNode.RemoveChild($packageInstallationSectionNode)

    Write-Output "Ship: Entry removed."
}
else
{
    Write-Output "Ship: Entry not found."
}

Write-Output "Ship: Removing Web.config [system.web/httpHandlers] entry ..."

$httpHandlersWebNode = $webConfig.SelectSingleNode("/configuration/system.web/httpHandlers/add[@type = 'Sitecore.Ship.AspNet.SitecoreShipHttpHandler, Sitecore.Ship.AspNet']")

if ($null -ne $httpHandlersWebNode) {
    $httpHandlersWebNode.ParentNode.RemoveChild($httpHandlersWebNode)

    Write-Output "Ship: Entry removed."
}
else
{
    Write-Output "Ship: Entry not found."
}

Write-Output "Ship: Removing Web.config [system.webServer/handlers/remove] entry ..."

$httpHandlersServerRemove = $webConfig.SelectSingleNode("/configuration/system.webServer/handlers/remove[@name = 'Sitecore.Ship']")

if ($null -ne $httpHandlersServerRemove) {
    $httpHandlersServerRemove.ParentNode.RemoveChild($httpHandlersServerRemove)

    Write-Output "Ship: Entry removed."
}
else
{
    Write-Output "Ship: Entry not found."
}

Write-Output "Ship: Removing Web.config [system.webServer/handlers/add] entry ..."

$httpHandlersServerAdd = $webConfig.SelectSingleNode("/configuration/system.webServer/handlers/add[@name = 'Sitecore.Ship']")

if ($null -ne $httpHandlersServerAdd) {
    $httpHandlersServerAdd.ParentNode.RemoveChild($httpHandlersServerAdd)

    Write-Output "Ship: Entry removed."
}
else
{
    Write-Output "Ship: Entry not found."
}

Write-Output "Ship: Removing Web.config [packageInstallation] node ..."

$packageInstallationNode = $webConfig.SelectSingleNode("/configuration/packageInstallation")

if ($null -ne $packageInstallationNode) {
    $packageInstallationNode.ParentNode.RemoveChild($packageInstallationNode)

    Write-Output "Ship: Node removed."
}
else
{
    Write-Output "Ship: Node not found."
}

Write-Output "Ship: Saving Web.config file ..."

$webConfig.Save($webConfigPath)

Write-Output "Ship: Web.config file saved."

Write-Output "Ship: Removing DLL files ..."

Remove-Item -Path "$ProjectFolder\bin\Sitecore.Ship.AspNet.dll" -Verbose -Force -ErrorAction SilentlyContinue

Write-Output "Ship: Removed Sitecore.Ship.AspNet.dll."

Remove-Item -Path "$ProjectFolder\bin\Sitecore.Ship.Core.dll" -Verbose -Force -ErrorAction SilentlyContinue

Write-Output "Ship: Removed Sitecore.Ship.Core.dll."

Remove-Item -Path "$ProjectFolder\bin\Sitecore.Ship.Infrastructure.dll" -Verbose -Force -ErrorAction SilentlyContinue

Write-Output "Ship: Removed Sitecore.Ship.Infrastructure.dll."

Write-Output "Ship: Sitecore.Ship.AspNet removed."

$PowerShellConfig = "$ProjectFolder\App_Config\Include\Z.MyProject\PowerShell"

if ((Test-Path -Path $PowerShellConfig) -eq $false)
{
    Write-Error "PowerShell configs: Could not find $PowerShellConfig"
	Exit 1
}
else {
    Remove-Item -Recurse -Force $PowerShellConfig
    Write-Output "Smartling: Removed $PowerShellConfig"
}

Write-Output "Content management site tweaked."
```

As you could see here we're doing some simple steps:

- remove connection string to master database
- remove Sitecore.Ship, because we don't need it on CDS environment
- remove Powershell config files and so on.

**Step 9: Clean up CMS folder**

Script arguments: "%teamcity.build.workingDir%\MyProject.CMS"

Here some sample PowerShell script which will clean up CMS folder:

```PowerShell
param
(
	[Parameter(Mandatory = $true)]
    [ValidateNotNullOrEmpty()]
    [String] $ProjectFolder = $(throw "PROJECT FOLDER parameter is required.")
)

Set-StrictMode -Version Latest

if ((Test-Path -Path $ProjectFolder) -eq $false)
{
    Write-Error "Could not find $ProjectFolder"
	Exit 1
}

Write-Output "Start tweaking content management site."

Write-Output "Clean: Starting to clean BIN folder ..."

Write-Output "Clean: Removing App_Config folder ..."

Remove-Item -Path "$ProjectFolder\bin\App_Config" -Verbose -Force -ErrorAction SilentlyContinue

Write-Output "Clean: App_Config folder removed."

Write-Output "Start tweaking headers section."

$webConfigPath = "$ProjectFolder\Web.config"

if ((Test-Path -Path $webConfigPath) -eq $false)
{
    Write-Error "Headers: Could not find $webConfigPath"
	Exit 1
}

Write-Output "Headers: Open Web.config file ..."

$webConfig = New-Object -TypeName System.Xml.XmlDocument
$webConfig.Load($webConfigPath)

$responseHeadersNode = $webConfig.SelectSingleNode("/configuration/system.webServer/rewrite/rules/rule[@name = 'Setup response headers']")

if ($null -ne $responseHeadersNode) {
    $responseHeadersNode.ParentNode.RemoveChild($responseHeadersNode)
    Write-Output "Headers: Removed response headers rule."
}
else
{
	Write-Output "Headers: Response headers rule not found."
}

Write-Output "Headers: Saving Web.config file ..."

$webConfig.Save($webConfigPath)

Write-Output "Headers: Web.config file saved."

Write-Output "Headers section tweaked."

Write-Output "Content management site tweaked."
```

**Step 10: CDS pack**

Here I'll create a NuGet package from CDS folder.

1. Runner type: NuGet pack
2. Step name: CDS pack
3. NuGet.exe: 4.5.0
4. Path to my NuGet specification file. For example something like - %teamcity.build.workingDir%\DevOps\MyProject.CDS.nuspec

```XML
<?xml version="1.0"?>
<package
    xmlns="http://schemas.microsoft.com/packaging/2011/08/nuspec.xsd">
    <metadata>
        <id>MyProject.CDS</id>
        <version>1.0.0</version>
        <title>MyProject.CDS Update package</title>
        <authors>MyProject</authors>
        <owners>MyProject</owners>
        <requireLicenseAcceptance>false</requireLicenseAcceptance>
        <description>Contains MyProject.CDS package</description>
        <copyright>Copyright Â©</copyright>
        <tags></tags>
    </metadata>
    <files>
        <file src="..\..\MyProject.CDS\**\*.*" />
    </files>
</package>
```

5. Version: %build.number%
6. Check this option "Publish created packages to build artifacts"
7. Check this option "Create tool package"

**Step 11: CMS pack**

Here I'll create a NuGet package from CMS folder.

1. Runner type: NuGet pack
2. Step name: CMS pack
3. NuGet.exe: 4.5.0
4. Path to my NuGet specification file. For example something like - %teamcity.build.workingDir%\DevOps\MyProject.CMS.nuspec

```XML
<?xml version="1.0"?>
<package
    xmlns="http://schemas.microsoft.com/packaging/2011/08/nuspec.xsd">
    <metadata>
        <id>MyProject.CMS</id>
        <version>1.0.0</version>
        <title>MyProject.CMS Update package</title>
        <authors>MyProject</authors>
        <owners>MyProject</owners>
        <requireLicenseAcceptance>false</requireLicenseAcceptance>
        <description>Contains MyProject.CMS package</description>
        <copyright>Copyright Â©</copyright>
        <tags></tags>
    </metadata>
    <files>
        <file src="..\..\MyProject.CMS\**\*.*" />
    </files>
</package>
```

5. Version: %build.number%
6. Check this option "Publish created packages to build artifacts"
7. Check this option "Create tool package"

**Step 12: Clean up CDS/CMS/Publish folder**

After all build, package job is done it's time to clean up and apply [boy scout rule](http://deviq.com/boy-scout-rule/). To do this I'll use again sample PowerShell script.

```PowerShell
Remove-Item -Recurse -Force MyProject.CDS
Remove-Item -Recurse -Force MyProject.CMS
Remove-Item -Recurse -Force MyProject.Published
```

For the next two steps I will use Octopus plugin for TeamCity.

**Step 13: OctopusDeploy: Release CMS**

1. Runner type: OctopusDeploy: Create release
2. Octopus url: path to your octopus deploy instance
3. Octopus API key
4. Octopus version
5. Project name
6. Release number - %build.number%
7. Environment(s)
8. Check option "Show deployment progress"
9. Additional command line arguments:

```PowerShell
--packageversion=%build.number% --package=Sitecore.Base.Data:8 --package=Sitecore.Base.Website.CMS:8 --deploymenttimeout=00:30:00
```

With the above **octo.exe** arguments I'm solving some of the issues I had before I add them:
- CMS instance always used the previous version. Never mind that the new version is available in the TeamCity NuGet feed every time use the previous version. So if you don't fix it you should always trigger two builds & deploy to have the latest version. This argument fix it "--packageversion=%build.number%".
- previous argument fix double build issue, but create another one. Sitecore.Base.Data and Sitecore.Base.Website.CMS was triggered with version %build.number%. Those two packages are constant and I'll tell later what I mean by base package. To fix this issue I fix it with "--package=Sitecore.Base.Data:8 --package=Sitecore.Base.Website.CMS:8"
- TeamCity has deploy time out limit to 10 minutes, so I extend it with this argument "--deploymenttimeout=00:30:00"

**Step 14: OctopusDeploy: Release CDS1**

The only difference from the previous step is the following:
- Step name
- Project name
- Additional command line arguments:

```PowerShell
--packageversion=%build.number% --package=Sitecore.Base.Data:8 --package=Sitecore.Base.Website.CDS:8 --deploymenttimeout=00:30:00
```

**Step 15: OctopusDeploy: Release CDS2**

The same as **Step 14**, but with different values for:
- Step name
- Project name

**Step 16: Notify Slack Channel for Success**

*This step will execute only if all previous steps finished successfully.*

Again I will use PowerShell as I do it in **Step 1** and notify my team, but with different message. Here is an example:

```PowerShell
$slackToken = "your-slack-token"
$slackChannel = "#your-slack-channel"
$slackText = "Success Build & Deploy *%build.number%* - CMS -> https://cms.your-domain.com/ CDS -> https://your-domain.com/"
$slackUserName = "TeamCity"
$slackIcon = "https://pbs.twimg.com/profile_images/880014590427496448/fducIHLi_400x400.jpg"

$postSlackMessage = @{token=$slackToken; channel=$slackChannel; text=$slackText; username=$slackUserName; icon_url=$slackIcon}

Invoke-RestMethod -Uri https://slack.com/api/chat.postMessage -Body $postSlackMessage
```

I'm not going to explain how you could get Slack Token, because I'm sure that you're smart enough to do it :)

We're done with the build. The result of the following steps will be artifacts for CDS/CD, CMS/CM and TDS update packages. I'll use those artifacts in Octopus Deploy.

## 2. Setup [Octopus Deploy](https://octopus.com/)

To be continued...

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