---
layout: post
title: "User Secrets"
date: 2023-09-23 13:37:48 +0200
tags: net7 user-secrets visual-studio
---

While `appsettings.json` and `appsettings.Development.json` can contain base configuration and development-specific settings, as soon as it comes to sensitive values, like connectionstrings etc, you shouldn't commit them to your repo.

For production scenarios, you might get your secrets from a keyvault, but for development, some teams put their secrets in `appsettings.Development.json` and de-lists it from git. However, there is already a feature in VS and .Net for this. Its called UserSecrets. UserSecrets are stored in you AppData/Roaming-folder which is of course not tracked in git.

As with other common "non-committed-file"-based solutions to this problem, it is quite hard to keep user secrets in sync among developers. Every other day, some developer can't start the application on their machine because someone else added some code that expects a secret value in the configuration, which of course can't be found in the User Secrets on that developers computer.

This proposal here doesnt really solve the problem, but it helps a bit.

## Make the secrets.json show up in visual studio.

The user secrets are hidden behind a right click on the csproj-file, under the `Manage User Secrets`-menu item. There is no easy way to tell the file is there by just looking at the project in visual studio. I want the secrets file to show up next to `appsettings.json`. This makes it more apparent to developers that user secrets should be configured for this project. By placing this code in a file called `Directory.Build.Props` next to your solution file, the `secrets.json`-file will be automatically linked in for all projects that has user-secrets configured.

```xml
<Project>
	<ItemGroup Condition="'$(UserSecretsId)' != ''">
		<None Include="$([System.Environment]::GetFolderPath(SpecialFolder.ApplicationData))\Microsoft\UserSecrets\$(UserSecretsId)\secrets.json" Link="secrets.json" />
		<None Include="secrets.template.json" />
	</ItemGroup>
</Project>
```

## A template file

As you can see, there's also a reference to something called `secrets.template.json` in the file above. (You could also call it maybe `secrets.example.json` or whatever you like.) The idea here is to show developers what their `secrets.json` should look like. It should contain all keys that the application expects `secrets.json` to contain. Values in this file could be example values or just empty values. Don't store actual secrets here. This way, a developer can take a look in the template file, to see if anything is missing in his or her own `secrets.json`. Add a `secrets.template.json` to all projects with usersecrets configured.

To make this file show up nicely nested under the `secrets.json`, add the following code to a file called `.filenesting.json` next to your solution file.

```json
{
  "help": "https://go.microsoft.com/fwlink/?linkid=866610",
  "dependentFileProviders": {
    "add": {
      "fileToFile": {
        "add": {
          "secrets.json": ["appsettings.json"]
        }
      }
    }
  }
}
```

Restart vs and your projects with usersecrets enabled should look like this

![Alt text](/wathaha/assets/2023-09-23-User-Secrets.png)
