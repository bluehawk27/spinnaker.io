---
title: "Plugin User Guide"
no_list: true
description: >
  Add, configure, and deploy plugins.
---

## Overview

<<<<<<< HEAD:content/en/docs/guides/user/plugins/user-guide.md
Spinnaker uses [PF4J-Update](https://github.com/pf4j/pf4j-update) to load and manage plugins. These plugins can implement a PF4J extension point or be Spring components. See the [Plugin Creators Guide](/docs/guides/developer/plugin-creators/overview/) for details.
=======
Spinnaker uses [PF4J-Update](https://github.com/pf4j/pf4j-update) to load and manage plugins. These plugins can implement a PF4J extension point or be Spring components. See the [Plugin Creators Guide]({{< ref "plugin-creator" >}}) for details.
>>>>>>> b3e7770 (Port PR 2142):content/en/docs/guides/user/plugins-users/_index.md

## Terms

**plugins.json**

* required file
* defines one to many plugins in a plugin repository
* each plugin definition has an id, description, provider, and a collection of releases (version, date, requires, sha512sum, state, url)
* the plugin developer provides access to this file

**repositories.json**

* optional file
* defines one to many plugin repositories
* each repository definition consists of a unique identifier and the path to a `plugins.json` file
* the plugin developer may supply this file

## Plugin requirements

* The plugin is either a [Plugin Framework for Java](https://github.com/pf4j/pf4j)(PF4J) plugin or a Spring plugin
* The plugin repository is a web location that Spinnaker can access, like a GitHub repository

Spinnaker environment:

* Spinnaker v1.20.6, v1.21.0 (assumption is v1.21.x but not validated)
* Halyard v1.36 to deploy Spinnaker (assumption is v1.36+ but not validated)


## How to add a plugin to Spinnaker

- [Overview](#overview)
- [Terms](#terms)
- [Plugin requirements](#plugin-requirements)
- [How to add a plugin to Spinnaker](#how-to-add-a-plugin-to-spinnaker)
- [Add a plugin repository using Halyard](#add-a-plugin-repository-using-halyard)
- [List, edit, and delete repositories](#list-edit-and-delete-repositories)
- [Add a plugin using Halyard](#add-a-plugin-using-halyard)
  - [Plugin configuration without Halyard](#plugin-configuration-without-halyard)
- [List, edit, and delete repositories](#list-edit-and-delete-repositories-1)
- [Add a Deck proxy to Gate](#add-a-deck-proxy-to-gate)
- [Redeploy Spinnaker](#redeploy-spinnaker)
- [Deployment example](#deployment-example)
- [Resources](#resources)

## Add a plugin repository using Halyard

_Note: Your plugins.json and repository.json files must be in a location that Spinnaker can access. Token authentication to private repositories is not supported. Consider storing your plugins and repository files in an AWS S3 bucket (or similar) instead of a private repository._

When you configure a repository, you tell Spinnaker where to find the `plugins.json` file that defines the plugins you want to use.  Each plugin repository entry in Spinnaker consists of a unique name and a URL.

If you want a repository to point to a single `plugins.json` file, you add it like this:

```bash
hal plugins repository add <unique-repo-name> --url=<path-to-plugins.json-file>
```

For example:

```
hal plugins repository add spinnaker-plugin-examples \
    --url=https://raw.githubusercontent.com/spinnaker-plugin-examples/examplePluginRepository/master/plugins.json
```

This action creates an entry in your Halconfig:

```yaml
repositories:
  spinnaker-plugin-examples:
    id: spinnaker-plugin-examples
    url: https://raw.githubusercontent.com/spinnaker-plugin-examples/examplePluginRepository/master/plugins.json
```

If you want a single plugin repository entry to point to multiple `plugins.json` files, you need to create a `repositories.json` file that defines a collection of plugin repositories. The file format is:

```json
[
  {
    "id": "<unique-repo-name>",
    "url": "<url-of-plugins.json-file>"
  }
]
```

For example:

```json
[
  {
    "id": "spinnaker-plugin-examples",
    "url": "https://raw.githubusercontent.com/spinnaker-plugin-examples/examplePluginRepository/master/plugins.json"
  },
  {
    "id": "my-company-internal-plugins",
    "url": "https://<my-company-internal-github>/<repo-name>/plugins.json"
  },
  {
    "id": "my-plugins",
    "url": "https://github.com/aimeeu/pluginRepository/blob/master/plugins.json"
  }
]
```

Save your `repositories.json` file in a web location that Spinnaker can access. Then you can add a new plugin repository using the `repositories.json` file:

```
hal plugins repository add all-the-plugins \
    --url=https://raw.githubusercontent.com/aimeeu/all-the-plugins/master/repositories.json
```

You can also list, edit, and delete repositories. See the command [reference](/docs/reference/halyard/commands/#hal-plugins-repository) for a complete list of parameters.

Don't forget to `hal deploy apply` to apply your configuration changes.

## List, edit, and delete repositories

See the command [reference](/docs/reference/halyard/commands/#hal-plugins-repository) to list, edit, or delete repositories.

## Add a plugin using Halyard

> Note: When Halyard adds a plugin to a Spinnaker installation, it adds the plugin repository information to each service. This means that when you restart Spinnaker, each service restarts, downloads the plugin, and checks if an extension exists for that service. Each service restarting is not ideal for large Spinnaker installations due to service restart times. Clouddriver can take an hour or more to restart if you have many accounts configured. Engineers are working to shorten restart times. See the [Plugin configuration without Halyard](#plugin-configuration-without-halyard) section for how to avoid each service restarting.

After you have added your plugin repository, you can add your plugin to Spinnaker. The Halyard [command](/docs/reference/halyard/commands/#hal-plugins-add) is:

```bash
hal plugins add <unique-plugin-id> --extensions=<extension-name> \
--version=<version> --enabled=true
```

The plugin distributor should provide you with the `unique-plugin-id`, `extensions`, and `version` values as well as any plugin configuration details. If you have to hunt for these values, you can find `unique-plugin-id` and `version` in the `plugins.json` file, but you have to look at the code to find the value for `extensions`. Search for the deprecated `@ExtensionConfiguration` or the current `@PluginConfiguration` annotation. Both take a value, which is the extension name.

Example of the deprecated `@ExtensionConfiguration` annotation in the [pf4jStagePlugin](https://github.com/spinnaker-plugin-examples/pf4jStagePlugin/blob/master/random-wait-orca/src/main/kotlin/io/armory/plugin/stage/wait/random/RandomWaitConfig.kt), which is written in Kotlin:

```kotlin
package io.armory.plugin.stage.wait.random

import com.netflix.spinnaker.kork.plugins.api.ExtensionConfiguration

@ExtensionConfiguration("armory.randomWaitStage")
data class RandomWaitConfig(var defaultMaxWaitTime: Int)
```

Example of the `@PluginConfiguration` annotation in the [Notification Plugin](https://github.com/spinnaker-plugin-examples/notificationPlugin/blob/master/notification-agent-echo/src/main/kotlin/io/armory/plugin/example/echo/notificationagent/HTTPNotificationConfig.kt#L5), which is also written in Kotlin:

```kotlin
package io.armory.plugin.example.echo.notificationagent

import com.netflix.spinnaker.kork.plugins.api.PluginConfiguration

@PluginConfiguration("armory.httpNotificationService")
data class HTTPNotificationConfig(val url: String)
```

Plugin configuration variables are passed into the primary class constructor. If the plugin developer doesn't specify configuration details, you can find key and type, or a configuration tree, by looking at the primary class constructor.

You add the `pf4jStagePlugin` to Spinnaker like this:

```bash
hal plugins add Armory.RandomWaitPlugin --extensions=armory.randomWaitStage \
--version=1.1.4 --enabled=true
```

Halyard adds the plugin configuration to the `.hal/config` file. Note the plugin's empty `config` collection.

```yaml
spinnaker:
  extensibility:
    plugins:
      Armory.RandomWaitPlugin:
        id: Armory.RandomWaitPlugin
        enabled: true
        version: 1.1.14
        extensions:
          armory.randomWaitStage:
            id: armory.randomWaitStage
            enabled: true
            config: {}
```

Halyard _does not_ support configuring plugins. You should manually edit the  Halconfig file for custom values. For example, `pf4jStagePlugin` has a configurable `defaultMaxWaitTime`, so you add that parameter to the plugin's configuration in the `config` collection section:

```yaml
spinnaker:
  extensibility:
    plugins:
      Armory.RandomWaitPlugin:
        id: Armory.RandomWaitPlugin
        enabled: true
        version: 1.1.14
        extensions:
          armory.randomWaitStage:
            id: armory.randomWaitStage
            enabled: true
            config:
              defaultMaxWaitTime: 60
```

Note: `hal plugins enable` and `hal plugins disable` enable or disable _all_ plugins, so use with caution.

### Plugin configuration without Halyard

To avoid each service restarting and downloading the plugin, _do not_ add the plugin using Halyard. Instead, configure the plugin in the service's local file. For example, if your plugin extends Orca, add configuration to your `orca-local.yml` file.

```yaml
spinnaker:
  extensibility:
    plugins:
      <unique-plugin-id>:
        id: <unique-plugin-id>
        enabled: <true-false>
        version: <version>
        extensions:
          <extension-name>:
            id: <extension-name>
            enabled: <true-false>
            config: {}
```

The plugin developer should provide configuration details in YAML format. If not:

1. Add the plugin using Halyard.
1. Do not restart Spinnaker.
1. Copy the plugin configuration from the Halconfig file.
1. Paste the plugin configuration into the relevant service's local file. Make sure configuration is in the format detailed above.
1. [Delete](https://spinnaker.io/reference/halyard/commands/#hal-plugins-delete) the plugin by executing `hal plugins delete <unique-plugin-id>`.
1. Restart Spinnaker

## List, edit, and delete repositories

See the command [reference](/docs/reference/halyard/commands/#hal-plugins-repository) to list, edit, or delete repositories.

## Add a Deck proxy to Gate

If your plugin has a Deck component, you need to configure a `deck-proxy` so Gate knows where to find the plugin.

You can create or find the `gate-local.yml` in the same place as the other Halyard configuration files. This is usually `~/.hal/default/profiles` on the machine where Halyard is running.

```yaml
spinnaker:
   extensibility:
     deck-proxy:
       enabled: true
       plugins:
         <unique-plugin-id>:
           enabled: true
           version: <version>
       repositories:
         <unique-repo-name>:
           url: <url-to-repositories.json-or-plugins.json>
```

* `unique-plugin-id`: the plugin ID you used when you added the plugin to Spinnaker ([Add a plugin using Halyard](#add-a-plugin-using-halyard) section)
* `unique-repo-name`: the plugin repository ID you used when you added the repository to Spinnaker ([Add a plugin repository using Halyard](#add-a-plugin-repository-using-halyard) section)
* `url`: the location of the plugin repository ([Add a plugin repository using Halyard](#add-a-plugin-repository-using-halyard) section)

## Redeploy Spinnaker

Remember to `hal deploy apply` after you have finished configuring your plugin.

## Deployment example

<<<<<<< HEAD:content/en/docs/guides/user/plugins/user-guide.md
See the [pf4jStagePlugin Deployment Example](/docs/guides/user/plugins/deploy-example/) page for a walkthrough and troubleshooting.
=======
See the [pf4jStagePlugin Deployment Example]({{< ref "plugin-deploy-example" >}}) page for a walkthrough and troubleshooting.
>>>>>>> b3e7770 (Port PR 2142):content/en/docs/guides/user/plugins-users/_index.md

## Resources

You can ask for help with plugins in the [Spinnaker Slack's](https://join.spinnaker.io/) `#plugins` channel.
