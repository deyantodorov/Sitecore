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
