# Sitecore 8.2 and setting up custom paths for "Views" location in ViewEngines

1. Spend 2-3 hours or days figting with **Sitecore 8.2** finding a way to modify **ViewEngines.Engines** in ASP.NET MVC application. *Well if you are reading this I've assume you already done this :)*

2. Make a *CustomViewsPathsProcessor.cs* or use another naming convention. Here is my example:
``` C#
using System.Linq;
using System.Web.Mvc;
using Sitecore.Pipelines;
using Sitecore.Mvc.Pipelines.Loader;

namespace ProjectName.Infrastructure
{
    public class CustomViewsPathsProcessor : InitializeRoutes
    {
        public override void Process(PipelineArgs args)
        {
            var razorViewEngine = new RazorViewEngine();

            razorViewEngine.ViewLocationFormats = razorViewEngine.ViewLocationFormats
                    .Concat(new string[] {
                    "~/MyProject/Public/Views/{1}/{0}.cshtml",
                    "~/MyProject/Public/Views/{0}.cshtml",
                    "~/MyProject/Public/Views/Shared/{0}.cshtml"
                    }).ToArray();

            razorViewEngine.PartialViewLocationFormats = razorViewEngine.PartialViewLocationFormats
                    .Concat(new string[] {
                    "~/MyProject/Public/Views/Shared/{0}.cshtml"
                    }).ToArray();

            ViewEngines.Engines.Add(razorViewEngine);
        }
    }
}
```
3. Register **CustomViewsPathsProcessor** in **Sitecore.config**. How?!

In your App_Config\Include folder make a new config file called **Z.Pipelines.config**

``` XML
<sitecore>
    <pipelines>
      <initialize>
        <processor type="ProjectName.Infrastructure.CustomViewsPathsProcessor, ProjectName.Infrastructure" patch:after="processor[@type='Sitecore.Social.Client.Pipelines.Initialize.RemoteEventMap, Sitecore.Social.Client']" />
      </initialize>
    </pipelines>
</sitecore>
```

In my example I initialize this custom view paths after last processor. In your case the last processor could be different. You could find it here "http://yourproject.com/sitecore/admin/showconfig.aspx" and search for **</initialize**. Then you will be able to see your last process and use it in above XML code. 

By the way on this url you could see all Sitecore configurations from all XML configs, but combined in a single file.

*Happy coding ;)*