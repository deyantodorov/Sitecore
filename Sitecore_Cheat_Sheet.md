# Sitecore - cheat sheet

# Page Tokens

| Token name | Description  |
|---|---|
| $name  | Name of the item |
| $id | ID of the item |
| $parentid | ID of the parent of the item |
| $parentname | Name of the parent of the item |
| $date | Current date (ISO format) |
| $time | Current time (ISO format) |
| $now | Current date and time (ISO format) |

# Image parameters

| Parameter | Property affected  |
|---|---|
| w  | Width  |
| h  | Height  |
| mw  | Max Width  |
| mh  | Max Height  |
| la | Language |
| vs | Version |
| db | Database |
| bc | Background Color |
| as | Allow Stretch |
| sc | Scale (floating point) |

# Definitions

Template - means data template or object. For example object Person with properties name, age, address.

Content - content tree in Sitecore administration

Layout - in other CMS systems this is equal to template, but in Sitecore context Layout means basic structure. It's the same as MVC layout.

# CLEAN OUT EVENT QUEUE, HISTORY TABLE, PUBLISH QUEUE

The following SQL statement will clean out the history table, publish queue and event queue, leaving only 12 hours of history and publish data and 4 hours of events. Replace YOURDATABASE with the name of your database:

```SQL
/****** History ******/
delete FROM [YOURDATABASE_Core].[dbo].[History] where Created < DATEADD(HOUR, -12, GETDATE())
delete FROM [YOURDATABASE_Master].[dbo].[History] where Created < DATEADD(HOUR, -12, GETDATE())
delete FROM [YOURDATABASE_Web].[dbo].[History] where Created < DATEADD(HOUR, -12, GETDATE())
  
/****** Publishqueue ******/
delete FROM [YOURDATABASE_Core].[dbo].[PublishQueue] where Date < DATEADD(HOUR, -12, GETDATE());    
delete FROM [YOURDATABASE_Master].[dbo].[PublishQueue] where Date < DATEADD(HOUR, -12, GETDATE());
delete FROM [YOURDATABASE_Web].[dbo].[PublishQueue] where Date < DATEADD(HOUR, -12, GETDATE());
     
/****** EventQueue ******/
delete FROM [YOURDATABASE_Master].[dbo].[EventQueue] where [Created] < DATEADD(HOUR, -4, GETDATE())
delete FROM [YOURDATABASE_Core].[dbo].[EventQueue] where [Created] < DATEADD(HOUR, -4, GETDATE())
delete FROM [YOURDATABASE_Web].[dbo].[EventQueue] where [Created] < DATEADD(HOUR, -4, GETDATE())
```

# Links
Bundling
- http://www.bugdebugzone.com/2015/07/bundling-and-minification-in-sitecore.html
- http://jockstothecore.com/bundling-with-sitecore-mvc/

# Sitecore 9 topologies
- https://kb.sitecore.net/articles/043375 - great resource if information before you plan your costs.
