# Olive compatibility change log

## 18 Jan 2018
- Add Nuget reference to **Olive.Hangfire** in Website
- In FrontEnd.cshtml change @Html.ResetDatabaseLink() to **@Html.WebTestWidget()** 
- Make your StartUp.cs compatible with the new format:

```csharp
public override void ConfigureServices(IServiceCollection services)
{
    base.ConfigureServices(services);

    AuthenticationBuilder.AddSocialAuth();
    services.AddScheduledTasks();
}

public override void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    app.UseWebTest(ReferenceData.Create, config => config.AddTasks().AddEmail());
    base.Configure(app, env);

    if (Config.Get<bool>("Automated.Tasks:Enabled"))
        app.UseScheduledTasks(TaskManager.Run);
}
```
- From Startup.cs remove RegisterServiceImplementors() and CreateReferenceData() methods

## 17 Jan 2018
- In *#Model\Project.cs* remove all project settings except **Name(...)** and **SolutionFile(...)**.
- In Domain project delete Entities and DAL folders
- Delete App_Start\ViewLocationExpander.cs

# Up to 17 Jan 2018

### Delete these files:
- Website\gulpfile.js
- Website\package.json
- Domain\...\AnonymousUser.cs
- Domain\Services\RoleStore.cs
- Domain\UserStore.cs

### Update javascript structure
* Delete the following from Website
  * ScriptReferences.json
  * node_modules folder
* Update the following files in **Website** from [the latest template](https://github.com/Geeksltd/Olive.MvcTemplate/blob/master/Template/Website/)
  * Views\Layouts\Common.Scripts.cshtml
  * bower.json (after changing, in VS right click it and select Restore Packages)
  * wwwroot\Scripts folder (everything unless you've created your own scripts)
  * tsconfig.json (if you don't have it already, but copy it in)