# Change the default forms root used in the create dialog

*It turns out that the location can be changed in the Core database.
Locate item {5BDD9307-72C7-4831-A1B8-DF4C02399AC2}, which is stored in the path /sitecore/content/Documents and settings/All users/Start menu/Programs/Web Forms for Marketers/Create a New Form.
There you can change the ID in the "Message" field.*

*Source:*
http://www.partech.nl/nl/blog/2013/01/wfm-change-the-default-forms-root-for-create-dialog

# WFFM custom save action not woking on CD environment

We configured CM/CD environment for a website. When using Web Forms for Marketers in a CD environment, We faced an issue on custom save action, It was not working, it did not log any error in sitecore logs and it goes to next action like redirect and success page. 

There is a simple and easy way to resolve this issue, on WFFM custom save actions we need to mark checked on “Client Action” option.

![Client Action](http://sitecorecode.com/wp-content/uploads/2016/05/ClientAction.png)

*Source: http://sitecorecode.com/index.php/2016/05/23/wffm-custom-save-action-not-woking-on-cd-environment/*
