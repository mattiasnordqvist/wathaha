---
layout: post
title: "User Secrets"
date: 2023-09-23 13:37:48 +0200
categories: net7 user-secrets visual-studio
---

While `appsettings.json` and `appsettings.Development.json` can contain base configuration and development-specific settings, as soon as it comes to sensitive values, like connectionstrings etc, you shouldn't commit them to your repo.

Some put their secrets in `appsettings.Development.json` and de-lists it from git, but there is already a feature in VS and .Net for this. Its called UserSecrets. UserSecrets are stored in you AppData/Roaming-folder which is of course not tracked in git.

As with other common "non-committed-file"-based solutions to this problem, it is quite hard to keep user secrets updated among developers, and we constantly get errors because someone else added some code that expects a secret value in the configuration, which of course can't be found in the User Secrets on your computer.

This proposal here is not a solution, but it helps a bit.

1. Make the secrets.json show up in visual studio. This makes it more apparent to developers that user secrets should be configured for this project. Place this code in a file called `Directory.Build.Props` next to your solution file, and the secrets-file will be linked in for all projects that has user-secrets configured.

```xml
<Project>
	<ItemGroup Condition="'$(UserSecretsId)' != ''">
		<None Include="$([System.Environment]::GetFolderPath(SpecialFolder.ApplicationData))\Microsoft\UserSecrets\$(UserSecretsId)\secrets.json" Link="secrets.json" />
		<None Include="secrets.template.json" />
	</ItemGroup>
</Project>
```

2. Add a `usersecrets.template.json` to all projects with usersecrets configured. This file can contain examples or nonsensitive default values expected to be found in `secrets.json`. Now a developer can check if their `secrets.json` is symmetrical to the template. To make this file show up nicely nested under the `secrets.json`, add the following code to a file called `.filenesting.json` next to your solution file.

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

![Alt text](/_posts/2023-09-23_UserSecrets.png)
