# How to create Continuous Integration / Continuous Deployment (CI/CD) for Sitecore 8

## 1. Setup [TeamCity](https://www.jetbrains.com/teamcity/)

### 1.1 Setup build steps:

**Step 1:** If you are using Slack, you could add Slack notification for your team. Here you have two options:

- first write custom PowerShell script;
- second install some TeamCity plugin to help you with this notifications;

I tried some of the most popular plugins, but didn't work out for me and after some discussions decided to keep it simple and wrote a small PowerShell script. *Here is an example*:

```PowerShell
$slackToken = "your-slack-token"
$slackChannel = "#your-slack-channel"
$slackText = "@channel :smile: Start Build & Deploy *%build.number%*"
$slackUserName = "TeamCity"
$slackIcon = "https://pbs.twimg.com/profile_images/880014590427496448/fducIHLi_400x400.jpg"

$postSlackMessage = @{token=$slackToken; channel=$slackChannel; text=$slackText; username=$slackUserName; icon_url=$slackIcon}

Invoke-RestMethod -Uri https://slack.com/api/chat.postMessage -Body $postSlackMessage
```

**Step 2: ** 

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
