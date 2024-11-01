# envTrack VSCode Extension Configuration Guide

## Table of Contents
1. [Introduction](#introduction)
2. [Configuration File Structure](#configuration-file-structure)
3. [Key Components](#key-components)
   1. [External Config](#external-config)
   2. [Paths](#paths)
   3. [Commands](#commands)
      1. [Command Name](#command-name)
      1. [Execution Settings](#Execution-Settings)
      1. [Output Handling](#JSON-outputs)
      1. [Session Management](#Session-Management)
      1. [Command Behavior](#Command-Behavior)
      1. [Configuration and Parameters](#Configuration-and-Parameters)
      1. [Session-Specific Configurations](#Session-Specific-Configurations)
      1. [Template Usage](#Template-Usage)
      1. [Subcommands](#Subcommands)
   4. [Configs](#configs)
   5. [Command Templates](#command-templates)
   6. [Observable Commands](#observable-commands)
   7. [Autostart](#autostart)
4. [Advanced Features](#advanced-features)
5. [Best Practices](#best-practices)

## Introduction
The envTrack VSCode extension allows you to manage and execute commands across different environments. This guide will help you understand and configure the extension effectively.

## Configuration File Structure
The configuration is typically stored in a YAML file. The main components of the configuration are:

- External Config
- Paths
- Commands
- Configs
- Command Templates
- Observable Commands
- Autostart

> Example yaml

```yaml
externalConfigPath: ./external_config.yaml

paths:
  deploy-config: ./../../deployment-configs
  local-scripts: ./scripts

commands:
  Deploy:Frontend:
    background: true
    shell: bash
    forceEnv: true
    isJSONOutput: true
    hasTemplate: true
    defaultFileName: ${env}_frontend_deploy_${curDateTime}
    templateName: frontend_deploy_template
    command: npm run deploy
    killCmd: pkill -f "npm run deploy"
    defaultSession: staging
    autoSelectDefaultSession: true
    importFrom:
      - Common:Utilities
    configMap:
      envConfig: staging-fe
    params:
      buildType: production
      cacheEnabled: true
    sessions:
      staging:
        configMap:
          envConfig: staging-fe
      production:
        configMap:
          envConfig: prod-fe
        params:
          buildType: production-optimized

  Database:Backup:
    background: false
    shell: bash
    command: pg_dump -U ${DB_USER} -d ${DB_NAME} > backup_${curDateTime}.sql
    defaultSession: development
    protectionOnEnv: production
    params:
      DB_USER: configs.dbConfig.username
      DB_NAME: myapp_db
    sessions:
      development:
        configMap:
          dbConfig: dev-db
      production:
        configMap:
          dbConfig: prod-db

configs:
  staging-fe:
    location: ${paths.deploy-config}/staging-frontend.yaml
  prod-fe:
    location: ${paths.deploy-config}/production-frontend.yaml
  dev-db:
    location: ${paths.local-scripts}/dev-database.yaml
  prod-db:
    location: ${paths.local-scripts}/prod-database.yaml

commandTemplatesPaths:
  - ./templates/commands

commandFolderPaths:
  - ./custom-commands

observableCommands:
  - Deploy:Frontend
  - Database:Backup

autostart:
  Deploy:Frontend: staging
  Database:Backup: development
```

## Key Components

### External Config

The `externalConfigPath` feature allows you to extend your current configuration with values from an external file. This powerful functionality enables modular and reusable configuration setups.

Key points:
- Specified using the `externalConfigPath` property in the main configuration file.
- The path is relative to the main configuration file's location.
- Supports both absolute and relative paths.
- Can use path aliases defined in the `paths` section, e.g., `${paths.alias}`.

Example:
```yaml
externalConfigPath: ./external_config.yaml
```

#### Behavior:

1. The system reads the external config file specified by externalConfigPath.
2. The external config is merged with the main config, with the external config taking precedence for overlapping fields.
3. Paths in the external config are adjusted relative to the main config file's location.
4. The merged configuration includes all settings from both files, creating a comprehensive setup.

#### Advanced usage:

- Circular dependencies are detected and prevented to avoid infinite loops.
- Path resolution ensures that all paths in the merged config are valid and accessible.

#### Future enhancements:

- Support for multiple external configs using an array structure is planned for future versions.

#### Using external configs allows for:

- Separating sensitive information (e.g., API keys, credentials) from the main config.
- Sharing common configurations across multiple projects.
- Creating environment-specific configurations that can be easily swapped.
- Remember to use this feature judiciously to maintain clarity and avoid overly complex configuration structures.

### Paths

The `paths` section in the configuration file allows you to define key-value pairs for commonly used paths in your project. This feature provides a convenient way to reference and manage paths throughout your configuration.

Key features of the `paths` section:

1. **Path Aliases**: Define aliases for frequently used paths in your project.

2. **Variable Substitution**: Use path aliases in other parts of the configuration using the `${paths.alias}` syntax.

3. **Relative Path Resolution**: Paths are resolved relative to the location of the configuration file.

4. **Absolute Path Conversion**: The system automatically converts relative paths to absolute paths for consistency.

5. **Cross-Section Usage**: Path aliases can be used in various sections of the configuration, including `externalConfigPath`, `commandTemplatesPaths`, `commandFolderPaths`, and `configs`.

Example usage:

```yaml
paths:
  deploy-config: ./../../deployment-configs
  local-scripts: ./scripts
  LOCAL_CONFIG: ./local_configs

externalConfigPath: ${paths.deploy-config}/external_config.yaml

commandTemplatesPaths:
  - ${paths.local-scripts}/templates

configs:
  xyz:
    location: ${paths.LOCAL_CONFIG}/creds.yaml
```

In this example, `${paths.deploy-config}` will be replaced with the absolute path to `./../../deployment-configs` relative to the config file's location.

The `ConfigManager` class in `config.ts` handles the resolution and validation of these paths:

- The `resolvePathAliases` method converts relative paths to absolute paths.
- The `applyPathAliases` method replaces path aliases in the configuration with their resolved absolute paths.
- The `replacePathAliases` method handles the actual string replacement of path aliases.

By using the paths section, you can maintain a centralized list of important directories in your project, making it easier to update and manage paths across your entire configuration.

### Commands

The commands section in the configuration file allows you to define a key value pair of commands with various properties. Each command is identified by a unique name (it's key) and can have multiple attributes to customize its behavior.

#### Command Name

- The name of the command serves as its identifier.
- If the name contains colons (:), it will be used to create a hierarchical structure in the UI tree.
> Example: `Test:School:1` will be displayed as
* Test
   * School
      * 1 (> run)

#### Command Display

- **color**: Customize the command's display color in the command tree. Available colors:
  - `red`: Error foreground color
  - `green`: Test passed color
  - `blue`: Debug start color
  - `yellow`: Warning color
  - `orange`: Warning notification color
  - `purple`: Text link color
  - `gray`: Description text color

> Example:

```yaml
commands:
  Deploy:Frontend:
    color: green
    command: npm run deploy

  Database:Backup:
    color: orange
    command: pg_dump -U ${DB_USER}
```

#### Execution Settings

- **background**: Boolean flag. If set to true, the command will run in the background.
- **shell**: Specifies the shell or runtime environment for executing the command. Supports:
  - `"bash"`: Starts bash with your profile (`bash -l -c`)
  - `"vscode"`: Uses VS Code integrated terminal
  - `"external"`: Uses system default or custom external terminal
- **externalShellLocation**: When using external shells, specifies a custom terminal location
- **forceEnv**: Boolean flag. If true, the command will not run without selecting a session.
- **command**: The actual command string to be executed.
- **killCmd**: A command to run before starting your command (will be renamed in future versions).
- **pipe**: Used when using a different shell and you want to pipe outside the selected shell.

#### JSON outputs

- **isJSONOutput**: Boolean flag used to indicate if the command output is in JSON format.
- **hasTemplate**: Boolean flag used for report generation with JSON output.
- **defaultFileName**: Specifies the default filename for saving command output or logs. Supports variable substitution.
- **templateName**: Specifies the name of the template used for report generation.

#### Session Management

- **defaultSession**: Preselected session when you are asked which session to use for running the command.
- **autoSelectDefaultSession**: Boolean flag. If true, it will preselect the default session without asking.
- **protectionOnEnv**: Adds a confirmation step when running the command in specific environments (e.g., production). `(In development)`

#### Command Behavior

- `autocloseCommand`: Boolean flag. If true, the terminal will close as soon as the command is done (planned feature).
- `importFrom`: An array of commands to import parameters, configuration maps, or sessions from.
- `extend`: Extend the command with additional settings from another command. All fields will be loaded from the extended command and overwritten by the current command.
- `skipTerminalAutofocus`: Boolean flag. If true, the terminal or background logs will not automatically gain focus when the command starts.
- `enableLog`: Boolean flag. If true, command output will be saved to log files in the configured log directory.
- `informationalCommandSettings`: Configure settings for informational commands
  - `fileWatcher`: Array of file patterns to watch for changes. When matching files are created or modified, the command automatically re-executes.
  - `interval`: The interval in seconds for refreshing the command.
  - `commandTrigger`: Watch for triggers of the command to re-execute.
  - `order`: The order in which the command should be shown in the UI.

> Example:

```yaml
commands:
  System:Status:
    isInformational: true
    command: systemctl status myservice
    informationalCommandSettings:
      fileWatcher:
        - "**/status/*.log"
        - "**/health/*.json"
```

#### Configuration and Parameters

- `configMap`: Allows the use of dynamic config files. Values can be referenced in the command or params.
- `params`: Defines parameters for the command. Supports simple values and complex references using configs.
- `requiredParams`: List of parameters that must be provided for the command to execute.

#### Parameter Options

The `paramOptions` field allows you to define metadata and validation rules for command parameters:

```yaml
commands:
  Deploy:App:
    command: deploy.sh --env ${environment} --mode ${mode} --debug ${enableDebug}
    paramOptions:
      environment:
        type: enum
        description: Target deployment environment
        enumValues: ['dev', 'staging', 'prod']
        required: true
      mode:
        type: string
        description: Deployment mode
        default: 'standard'
      enableDebug:
        type: boolean
        description: Enable debug logging
        default: false
```

Supported options for each parameter:

- type: Defines the parameter type ('string' | 'number' | 'boolean' | 'enum')
- description: Optional description of the parameter's purpose
- enumValues: Array of allowed values (only for enum type)
- default: Default value if parameter is not provided
- required: If true, parameter must be specified before command execution

#### Session-Specific Configurations

- `sessions`: Defines environment-specific configurations that will overwrite configs or config maps when running with a particular session.

#### Template Usage

- `template`: Optional template name that this command is based on.

#### Subcommands

Commands can include subcommands, which are executed before the main command. This feature allows for creating complex command structures and workflows.

Key aspects of subcommands:

1. **Definition**: Subcommands are defined within a command using the `subCommands` key, which contains an array of subcommand definitions.

2. **Structure**: Each subcommand is defined with the following properties:
   - `commandName`: The name of the existing command to be executed as a subcommand.
   - `params` (optional): Parameters specific to this subcommand execution.
   - `configMap` (optional): Configuration map specific to this subcommand execution.
   - `session` (optional): Session information for this subcommand.

3. **Execution Order**: Subcommands are executed in the order they are listed, before the main command.

4. **Parameter and Config Inheritance**: Subcommands inherit and can override parameters and configurations:
   - Original command <- Subcommand definition <- Individual subcommand entry

Example:

```yaml
Subcommand Command:
  command: echo "Started group"
  params:
    a: b
  configMap:
    cnf: cool
  subCommands:
    - commandName: Test:Echo config map
    - commandName: Test:Echo config map
      configMap:
        cnf: bar
    - commandName: Test:Echo var
    - commandName: Test:Echo var
      params:
        echo: bar
    - commandName: Test:Echo
    - commandName: Inherit:1
      session:
      params:
      configMap:
```

In this example:

- The main command "Subcommand Command" will be executed after all subcommands.
- Subcommands like "Test:Echo config map" will be executed with inherited and potentially overridden parameters and configurations.
- The last subcommand "Inherit:1" demonstrates how session, params, and configMap can be explicitly set or cleared for a specific subcommand.

This subcommand feature enables the creation of complex command sequences and allows for fine-grained control over parameter and configuration inheritance within command groups.

### Configs

The `configs` section in the configuration file allows you to define external configuration files that can be used across your commands. This feature provides a flexible way to manage environment-specific settings and variables.

#### Key features of the configs section:

1. **Structure**: Each entry in the configs section is a key-value pair, where the key is a unique identifier for the config, and the value is an object with the following properties:
   - `location`: The path to the configuration file. This can be an absolute path or a relative path from the main config file's location.
2. **Path Resolution**: The `location` can use path aliases defined in the paths section, allowing for more flexible and maintainable configurations.
3. **Variable Loading**: When a config is referenced, the system automatically loads the variables defined in the specified file.
4. **Usage in Commands**: Config variables can be accessed in command parameters and configuration maps using the syntax `configs.<config-name>.<variable-name>`.
5. **Dynamic Configuration**: This feature allows for environment-specific settings to be easily swapped by changing the referenced config file.

> Example usage:

```yaml
configs:
  staging-fe:
    location: ${paths.deploy-config}/staging-frontend.yaml
  prod-fe:
    location: ${paths.deploy-config}/production-frontend.yaml
  dev-db:
    location: ${paths.local-scripts}/dev-database.yaml
  prod-db:
    location: ${paths.local-scripts}/prod-database.yaml

commands:
  Deploy:Frontend:
    command: npm run deploy
    configMap:
      envConfig: staging-fe
    params:
      DB_USER: configs.envConfig.DB_USERNAME

  Database:Backup:
    command: pg_dump -U ${DB_USER} -d ${DB_NAME} > backup_${curDateTime}.sql
    configMap:
      dbConfig: dev-db
    params:
      DB_USER: configs.dbConfig.username
      DB_NAME: myapp_db
```

In this example, different configuration files are specified for staging and production environments. The `Deploy:Frontend` command uses the `staging-fe` config by default, while the `Database:Backup` command uses the `dev-db` config.

By using the configs section, you can maintain separate configuration files for different environments or components of your project, making it easier to manage complex setups and switch between different configurations as needed.

#### Using Configs with Sessions and Config Maps

Configs can be dynamically used in combination with sessions and config maps to create flexible, environment-specific command configurations. This powerful feature allows you to easily switch between different settings based on the execution context.

##### Key concepts:

1. *Config Maps*: Allow you to alias config names, making it easy to switch between different configurations.
2. **Sessions**: Define environment-specific settings, including which configs to use.
3. **Variable Resolution**: Config variables are resolved at runtime, allowing for dynamic configuration based on the selected session.

> Example usage:

```yaml
commands:
  Deploy:Frontend:
    command: npm run deploy --env=${ENV_TYPE}
    configMap:
      envConfig: staging-fe
    params:
      ENV_TYPE: configs.envConfig.environment
    sessions:
      staging:
        configMap:
          envConfig: staging-fe
      production:
        configMap:
          envConfig: prod-fe

configs:
  staging-fe:
    location: ${paths.deploy-config}/staging-frontend.yaml
  prod-fe:
    location: ${paths.deploy-config}/production-frontend.yaml

# Content of staging-frontend.yaml
variables:
  environment: staging

# Content of production-frontend.yaml
variables:
  environment: production
```

In this example:

1. The `Deploy:Frontend` command uses a `configMap` to alias `envConfig` to `staging-fe` by default.
2. The `ENV_TYPE` parameter is set to `configs.envConfig.environment`, which will resolve to the environment variable in the selected config file.
3. When running the command with the `staging` session, it uses the `staging-fe` config.
4. When running with the `production` session, it switches to the `prod-fe` config.
5. The `ENV_TYPE` parameter will resolve to "staging" or "production" based on the selected session, without changing the command definition.

This setup allows you to easily switch between different environments by simply changing the session, while keeping the command structure consistent. It provides a powerful way to manage environment-specific configurations and promotes code reuse across different deployment scenarios.

### Command Templates

Command Templates provide a powerful way to define reusable command structures that can be inherited by other commands. This feature allows for efficient configuration management and promotes code reuse across different command definitions.

Key aspects of Command Templates:

1. **Definition**: Command templates are defined in separate YAML files located in directories specified by the `commandTemplatesPaths` configuration.

2. **Structure**: A command template has the same fields as a regular command but may omit certain fields. It serves as a base structure for other commands.

3. **Inheritance**: Commands can inherit from templates using the template field, which specifies the name of the template to use.

4. **Overriding**: Commands that use a template can override any field defined in the template, allowing for customization.

5. **Loading**: Templates are loaded automatically from the specified `commandTemplatesPaths` directories.

6. **Flexibility**: Templates can define partial command configurations, which can be completed or modified by the inheriting commands.

> Example of a Command Template:

```yaml
DB:Gen Connection String URL:
  command: |-
    echo ${URL};
  requiredParams:
    - PORT
  params:
    ORB_DB_USERNAME: configs.envConfig.values.ORB_DB_USERNAME
  sessions:
    shadow:
      configMap:
        envConfig: ShadowFe
    qa:
      configMap:
        envConfig: QaFe
    prod:
      params:
        envConfig: ProdFe
```

> Usage in a Command:

```yaml
commands:
  Generate:DB:URL:
    template: DB:Gen Connection String URL
    params:
      PORT: 5432
      URL: "postgresql://${ORB_DB_USERNAME}:${ORB_DB_PASSWORD}@${ORB_DB_HOST}:${PORT}/${ORB_DB_NAME}"
```

#### Benefits:

1. Reduces configuration duplication
2. Ensures consistency across similar commands
3. Simplifies maintenance and updates of common command structures
4. Allows for creating specialized commands from general templates

By leveraging Command Templates, you can create a more modular and maintainable command configuration system, promoting best practices and reducing potential errors in command definitions.

### Observable Commands

The Observable Commands feature allows you to specify a list of commands that can be monitored and tracked in real-time through the VSCode status bar. This feature provides quick visibility into the state of important processes in your project.

#### Key aspects of Observable Commands:

1. **Definition**: Observable Commands are defined in the main configuration file under the `observableCommands` key. This is an array of command names that you want to monitor.
2. **Real-time Updates**: The status of Observable Commands is updated in real-time, reflecting whether they are running, stopped, or have exited.
3. **Quick Access**: Clicking on the status bar item opens a quick pick menu, allowing you to view and interact with all registered processes, including Observable Commands.
4. **Visual Indicators**: Each Observable Command is displayed with an icon indicating its current status:

   - Stopped: $(circle-outline)
   - Running: $(pass-filled)
   - Exited: $(error)

5. **Interaction**: From the quick pick menu, you can start or stop Observable Commands directly, providing easy control over these important processes.

> Example configuration:

```yaml
observableCommands:
  - Deploy:Frontend
  - Database:Backup
  - API:HealthCheck
```

By utilizing Observable Commands, you can keep track of critical processes in your project without the need to manually check their status repeatedly. This feature enhances your workflow by providing at-a-glance information about key commands and allowing quick interaction when needed.

### Autostart

The Autostart feature allows you to configure commands to start automatically when the application launches. This is particularly useful for setting up your development environment or initializing key processes without manual intervention.

#### Configuration

Autostart is defined in the main configuration file as a key-value pair object:
```yaml
autostart:
  CommandName1: SessionName
  CommandName2: null
```

- The key is the name of the command to be auto-started.
- The value is either:
   - A specific session name to use when starting the command.
   - null or an empty string, indicating that the command should start for all available sessions.

#### Behavior

1. When the application starts, it checks the autostart configuration.
2. For each command listed in autostart:
   - If a session is specified, the command is started using that session.
   - If null or an empty string is specified, the command is started for all available sessions.
   - If autoSelectDefaultSession is set to true for the command, it will use the defaultSession without prompting.

> Example

```yaml
autostart:
  Deploy:Frontend: staging
  Database:Backup: null
```

In this example:

- The `Deploy:Frontend` command will automatically start with the staging session.
- The `Database:Backup` command will start for all available sessions.

#### Best Practices

1. Use autostart for commands that are essential to your development workflow.
2. Be cautious when auto-starting commands that make significant changes or require careful monitoring.
3. Consider using autoSelectDefaultSession in conjunction with autostart to streamline the startup process.
4. Regularly review and update your autostart configuration to ensure it aligns with your current project needs.

#### Related Configuration
- `defaultSession`: Specifies the default session to use when none is explicitly set.
- `autoSelectDefaultSession`: When set to true, automatically selects the default session without prompting.

By utilizing the Autostart feature, you can significantly streamline your development workflow and ensure that critical processes are initialized consistently across your team or deployment environments.

## Advanced Features

## Logging Configuration

The extension supports comprehensive logging capabilities for command outputs:

### Command-Level Logging

> NOTE: This feature is NOT available for background commands. Use outputs tab for background commands.

Enable logging for specific commands by setting the `enableLog` flag in the command configuration:

```yaml
commands:
  Deploy:Frontend:
    enableLog: true
    command: npm run deploy
```

### Log Storage Configuration

Logs are stored in one of two locations:

1. Custom log directory specified in VSCode settings:
   - Configure logFolder in VSCode settings to specify a custom log directory

> Example:

```json
"envTrack.logFolder": "./my-custom-logs"
```

#### Default location:

If no custom location is specified, logs are stored in .vscode/logs

### Log Access

Command logs can be accessed through:

- The Running Commands tree view in the sidebar
- Each command execution creates a separate log file
- Log files are named based on the command execution timestamp

(To be expanded)

## Best Practices
(To be expanded)
