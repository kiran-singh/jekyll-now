---
layout: post
title: Responsible Settings
---

The .Net config settings are usually left to individual teams/developers and mostly tackled by calling a static config manager or a wrapper where the setting is required.  
  
```csharp
// The old way
var timout = int.Parse(ConfigurationManager.AppSettings["Timeout"]);

// Newer
var apiId = Guid.Parse(_configManager.AppSetting("ApiId"));
```

As shown above, the resulting string value has to be converted to the type of variable if different.


Ideally this manipulation shouldn't be left to individual classes as they get littered with app setting names, methods to parse the setting values and tackling what to do if any setting is missing or invalid.  
There is also the possibility that a setting not required immediately would be missing or misconfigured when the code is deployed to live.  


A better way would be a class like the ```Settings``` class in code below that validates the settings and sets them up as needed.   
This is for **.Net Core** but similar class can be created for other versions of .Net.   
[Code on github](https://github.com/kiran-singh/ResponsibleSettings/)   


This class can be either injected or preferably just called on startup by the IOC container and the required config value injected into the controller or service that needs them.  


__The constructor uses reflection to get the names & types of the its class' properties and throws an exception if any setting is missing or if any of the values are default for the type.__

```csharp
public class Settings
{
    public Settings(IConfiguration configuration) 
    {
        var declaredProperties = 
            GetType().GetTypeInfo().DeclaredProperties.ToList();

        var missingSettings = 
            (from propertyInfo in declaredProperties
            where string.IsNullOrWhiteSpace(configuration[propertyInfo.Name])
            select propertyInfo.Name).ToList();

        if (missingSettings.Any())
            throw new Exception(
                $"Cannot start application. Missing environment variables: {string.Join(", ", missingSettings)}");

        var invalidSettings = new List<string>();

        foreach (var propertyInfo in declaredProperties)
        {
            var configValue = configuration[propertyInfo.Name];

            if (propertyInfo.PropertyType == typeof(int))
            {
                if(int.TryParse(configValue, out var value))
                    propertyInfo.SetValue(this, value);
                else
                    invalidSettings.Add(propertyInfo.Name);
            }  
            // Parse other types required: 
            // else if(propertyInfo.PropertyType == typeof(bool))...
            // else if(propertyInfo.PropertyType == typeof(Guid))...
            // ...

            else // strings
                propertyInfo.SetValue(this, configValue);
        }

        if(invalidSettings.Any())
                throw new Exception(
                    $"Cannot start application. Invalid environment variables: {string.Join(", ", invalidSettings)}");
    }

    public Guid ApiId { get; set; }

    public string HostName { get; set; }

    public int Timeout { get; set; }

    public string SecretKey { get; set; }
}
```

The above code ensures that the application starts up only when its application settings are setup with the valid values and avoids anti-patterns like code duplication, usage of setting names into classes etc.