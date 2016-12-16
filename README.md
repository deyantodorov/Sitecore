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

# WFFM custom save action not woking on CD environment

We configured CM/CD environment for a website. When using Web Forms for Marketers in a CD environment, We faced an issue on custom save action, It was not working, it did not log any error in sitecore logs and it goes to next action like redirect and success page. 

There is a simple and easy way to resolve this issue, on WFFM custom save actions we need to mark checked on “Client Action” option.

Source: http://sitecorecode.com/index.php/2016/05/23/wffm-custom-save-action-not-woking-on-cd-environment/

http://sitecorecode.com/wp-content/uploads/2016/05/ClientAction.png
