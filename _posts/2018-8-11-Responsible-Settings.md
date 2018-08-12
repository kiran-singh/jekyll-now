---
layout: post
title: Responsible Settings
---

The .Net config settings are usually left to individual developers and mostly tackled by calling a static config manager or a wrapper.  
  
```csharp
// The old method
var timout = int.Parse(ConfigurationManager.AppSettings["Timeout"]);

// Modern but still not great
var apiId = Guid.Parse(_configManager.AppSetting("ApiId"));
```

In both cases the resulting value is a string and will have to be converted to the type of variable if different.


Ideally this type of manipulation shouldn't be left to the individual classes as they get littered with app setting names, methods to parse the setting values and tackling what to do if any setting is missing or invalid.  


A better way would be a class like the ```Settings``` class in code below that checks for the availability of the settings and sets them up as needed.  


This can either injected or preferably just called on startup by the IOC container and the required value injected into the controller or service that needs them.  


The constructor uses reflection to get the names & types of the class properties and throws an exception if any setting is missing or if any of the values are default for the type.  

```csharp
public class Settings
{
    public Settings(IConfiguration configDictionary) 
    {
        var declaredProperties = GetType().GetTypeInfo().DeclaredProperties.ToList();

        var missingSettings = (from propertyInfo in declaredProperties
            where string.IsNullOrWhiteSpace(configDictionary[propertyInfo.Name])
            select propertyInfo.Name).ToList();

        if (missingSettings.Any())
            throw new Exception(
                $"Cannot start application. Missing environment variables: {string.Join(", ", missingSettings)}");

        var invalidSettings = new List<string>();

        foreach (var propertyInfo in declaredProperties)
        {
            if (propertyInfo.PropertyType == typeof(int))
            {
                var value = configDictionary[propertyInfo.Name].ToInt();
                
                if(value == default(int))
                    invalidSettings.Add(propertyInfo.Name);
                
                propertyInfo.SetValue(this, value);
            }

            // Evaluate bool, Guid etc
            
            else
            {
                var value = configDictionary[propertyInfo.Name];
                
                if(string.IsNullOrWhiteSpace(value))
                    invalidSettings.Add(propertyInfo.Name);
                
                propertyInfo.SetValue(this, value);
            }
            
            if(invalidSettings.Any())
                throw new Exception(
                    $"Cannot start application. Invalid environment variables: {string.Join(", ", invalidSettings)}");
        }
    }

    public Guid ApiId { get; set; }

    public string HostName { get; set; }

    public int Timeout { get; set; }

    public string SecretKey { get; set; }
}
```

```ToInt()``` is a string extension method that uses the ```TryParse()``` method to parse the integer safely as shown below. Similar methods (```ToBool()```, ```ToGuid()``` etc) can be used for other value types.
```csharp
public static int ToInt(this string input)
{
    int.TryParse(input, out var result);

    return result;
}
```
The above code can ensure that the application starts up only when its application settings are setup with the valid values and avoid code duplication, usage of setting names into classes etc.