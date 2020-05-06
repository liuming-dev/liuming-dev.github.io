# `helm plugin` source code learning

> ## Quick introduce of `helm plugin`

A overview of usage by `helm plugin -h`:

```bash
~/Workspace4go/src/github.com/helm/helm/cmd/helm(master*) » ./helm plugin -h

Manage client-side Helm plugins.

Usage:
  helm plugin [command]

Available Commands:
  install     install one or more Helm plugins
  list        list installed Helm plugins
  uninstall   uninstall one or more Helm plugins
  update      update one or more Helm plugins

Flags:
  -h, --help   help for plugin

......
```

> ## `helm plugin list`

```bash
~/Workspace4go/src/github.com/helm/helm/cmd/helm(master*) » ./helm plugin list -h
list installed Helm plugins

Usage:
  helm plugin list [flags]

Aliases:
  list, ls

Flags:
  -h, --help   help for list

......
-----------------------------------------------------------------------------------------------------------------------------------------------
~/Workspace4go/src/github.com/helm/helm/cmd/helm(master*) » ./helm plugin list --debug
plugin_list.go:35: [debug] pluginDirs: /Users/apple/Library/helm/plugins
NAME            VERSION DESCRIPTION                                                                
diff            3.1.1   Preview helm upgrade changes as a diff                                     
monitor         0.4.0   Query at a given interval a Prometheus, ElasticSearch or Sentry instance...
push            0.8.1   Push chart package to ChartMuseum                                          
schema-gen      0.0.2   generate json schema for values yaml
```

> **Source code**

```go
// plugin_list.go
func newPluginListCmd(out io.Writer) *cobra.Command {
    cmd := &cobra.Command{
        Use:     "list",
        Aliases: []string{"ls"},
        Short:   "list installed Helm plugins",
        RunE: func(cmd *cobra.Command, args []string) error {
            debug("pluginDirs: %s", settings.PluginsDirectory)
            // 1. list the plugins in the local plugin directory
            plugins, err := plugin.FindPlugins(settings.PluginsDirectory)
            if err != nil {
                return err
            }

            // 2. output plugin information in table format
            table := uitable.New()
            table.AddRow("NAME", "VERSION", "DESCRIPTION")
            for _, p := range plugins {
                table.AddRow(p.Metadata.Name, p.Metadata.Version, p.Metadata.Description)
            }
            fmt.Fprintln(out, table)
            return nil
        },
    }
    return cmd
}
```

*1 list the plugins in the local plugin directory*

```go
// plugin.go
func FindPlugins(plugdirs string) ([]*Plugin, error) {
    found := []*Plugin{}
    // Let's get all UNIXy and allow path separators
    for _, p := range filepath.SplitList(plugdirs) {
        // load plugins in local plugin storage directory
        matches, err := LoadAll(p)
        if err != nil {
            return matches, err
        }
        found = append(found, matches...)
    }
    return found, nil
}


// plugin.go
func LoadAll(basedir string) ([]*Plugin, error) {
    plugins := []*Plugin{}
    // We want basedir/*/plugin.yaml
    scanpath := filepath.Join(basedir, "*", pluginFileName)
    // get every plugin's plugin.yaml path
    matches, err := filepath.Glob(scanpath)
    if err != nil {
        return plugins, err
    }

    if matches == nil {
        return plugins, nil
    }

    for _, yaml := range matches {
        // retrieve every plugin directory
        dir := filepath.Dir(yaml)
        // load the plugin's plugin.yaml to get Plugin object
        p, err := LoadDir(dir)
        if err != nil {
            return plugins, err
        }
        plugins = append(plugins, p)
    }
    return plugins, nil
}


// plugin.go
func LoadDir(dirname string) (*Plugin, error) {
    // read the plugin.yaml file
    data, err := ioutil.ReadFile(filepath.Join(dirname, pluginFileName))
    if err != nil {
        return nil, err
    }

    plug := &Plugin{Dir: dirname}
    // convert the plugin.yaml to Plugin object
    if err := yaml.Unmarshal(data, &plug.Metadata); err != nil {
        return nil, err
    }
    return plug, nil
}
```

> ## `helm plugin install`

```bash
~/Workspace4go/src/github.com/helm/helm/cmd/helm(master*) » ./helm plugin install -h

This command allows you to install a plugin from a url to a VCS repo or a local path.

Usage:
  helm plugin install [options] <path|url>... [flags]

Aliases:
  install, add

Flags:
  -h, --help             help for install
      --version string   specify a version constraint. If this is not specified, the latest version is installed

......
-----------------------------------------------------------------------------------------------------------------------------------------------
~/Workspace4go/src/github.com/helm/helm/cmd/helm(master*) » ./helm plugin install https://github.com/karuppiah7890/helm-schema-gen --debug
[debug] vcs_installer.go:159: cloning https://github.com/karuppiah7890/helm-schema-gen to /Users/apple/Library/Caches/helm/plugins/https-github.com-karuppiah7890-helm-schema-gen
[debug] vcs_installer.go:91: copying /Users/apple/Library/Caches/helm/plugins/https-github.com-karuppiah7890-helm-schema-gen to /Users/apple/Library/helm/plugins/helm-schema-gen
plugin_install.go:73: [debug] loading plugin from /Users/apple/Library/helm/plugins/helm-schema-gen
plugin.go:60: [debug] running install hook: /bin/sh -c cd $HELM_PLUGIN_DIR; scripts/install_version.sh
karuppiah7890/helm-schema-gen info checking GitHub for tag '0.0.2'
karuppiah7890/helm-schema-gen info found version: 0.0.2 for 0.0.2/Darwin/x86_64
karuppiah7890/helm-schema-gen info installed ./bin/helm-schema-gen
Installed plugin: schema-gen
```

> **Source code**

```go
// plugin_install.go
func (o *pluginInstallOptions) run(out io.Writer) error {
    installer.Debug = settings.Debug

    // 1. construct Installer by the given args from command
    i, err := installer.NewForSource(o.source, o.version)
    if err != nil {
        return err
    }
    // 2. execute the install logic
    if err := installer.Install(i); err != nil {
        return err
    }

    debug("loading plugin from %s", i.Path())
    // 3. load the installed plugin information
    p, err := plugin.LoadDir(i.Path())
    if err != nil {
        return err
    }

    // 4. run plugin install hook if install hook is set
    if err := runHook(p, plugin.Install); err != nil {
        return err
    }

    fmt.Fprintf(out, "Installed plugin: %s\n", p.Metadata.Name)
    return nil
}
```

*1 construct Installer by the given args from command*

```go
// installer.go
// instance a concrete Installer by the given plugin address
func NewForSource(source, version string) (Installer, error) {
    // Check if source is a local directory
    if isLocalReference(source) {
        return NewLocalInstaller(source)
    } else if isRemoteHTTPArchive(source) {
        return NewHTTPInstaller(source)
    }
    // if source is the git repository, instance a VCSInstaller
    return NewVCSInstaller(source, version)
}
```

*2 execute the install logic*

```go
// installer.go
func Install(i Installer) error {
    // create the plugin base directory
    if err := os.MkdirAll(filepath.Dir(i.Path()), 0755); err != nil {
        return err
    }
    // check whether plugin directory exists to be sure that plugin is installed or not
    if _, pathErr := os.Stat(i.Path()); !os.IsNotExist(pathErr) {
        return errors.New("plugin already exists")
    }
    // plugin has not been installed, begin to install
    return i.Install()
}


// vcs_installer.go
func (i *VCSInstaller) Install() error {
    // synchronize git repository to local plugin cache directory
    if err := i.sync(i.Repo); err != nil {
        return err
    }

    // check and find the given plugin version
    ref, err := i.solveVersion(i.Repo)
    if err != nil {
        return err
    }
    if ref != "" {
        if err := i.setVersion(i.Repo, ref); err != nil {
            return err
        }
    }

    // check whether the cached plugin directory is a valid plugin
    if !isPlugin(i.Repo.LocalPath()) {
        return ErrMissingMetadata
    }

    debug("copying %s to %s", i.Repo.LocalPath(), i.Path())
    // copy local plugin cache directory file recursively to local plugin directory
    return fs.CopyDir(i.Repo.LocalPath(), i.Path())
}
```

*3 load the installed plugin information by the installed directory*

```go
// plugin.go
func LoadDir(dirname string) (*Plugin, error) {
    // read all data of the installed plugin's plugin.yaml
    data, err := ioutil.ReadFile(filepath.Join(dirname, pluginFileName))
    if err != nil {
        return nil, err
    }

    plug := &Plugin{Dir: dirname}
    // deserialize to plugin metadata
    if err := yaml.Unmarshal(data, &plug.Metadata); err != nil {
        return nil, err
    }
    return plug, nil
}
```

*4 run plugin install hook if install hook is set*

```go
// plugin.go
func runHook(p *plugin.Plugin, event string) error {
    hook := p.Metadata.Hooks[event]
    if hook == "" {
        return nil
    }

    prog := exec.Command("sh", "-c", hook)
    // TODO make this work on windows
    // I think its ... ¯\_(ツ)_/¯
    // prog := exec.Command("cmd", "/C", p.Metadata.Hooks.Install())

    debug("running %s hook: %s", event, prog)

    plugin.SetupPluginEnv(settings, p.Metadata.Name, p.Dir)
    prog.Stdout, prog.Stderr = os.Stdout, os.Stderr
    if err := prog.Run(); err != nil {
        if eerr, ok := err.(*exec.ExitError); ok {
            os.Stderr.Write(eerr.Stderr)
            return errors.Errorf("plugin %s hook for %q exited with error", event, p.Metadata.Name)
        }
        return err
    }
    return nil
}
```

> ## `helm plugin update`

```bash
~/Workspace4go/src/github.com/helm/helm/cmd/helm(master*) » ./helm plugin update -h
update one or more Helm plugins

Usage:
  helm plugin update <plugin>... [flags]

Aliases:
  update, up

Flags:
  -h, --help   help for update

......
-----------------------------------------------------------------------------------------------------------------------------------------
~/Workspace4go/src/github.com/helm/helm/cmd/helm
~/Workspace4go/src/github.com/helm/helm/cmd/helm(master*) » ./helm plugin update schema-gen --debug
plugin_update.go:72: [debug] loading installed plugins from /Users/apple/Library/helm/plugins
[debug] vcs_installer.go:97: updating https://github.com/karuppiah7890/helm-schema-gen
plugin_update.go:114: [debug] loading plugin from /Users/apple/Library/helm/plugins/helm-schema-gen
plugin.go:60: [debug] running update hook: /bin/sh -c cd $HELM_PLUGIN_DIR; scripts/install_version.sh
karuppiah7890/helm-schema-gen info checking GitHub for tag '0.0.2'
karuppiah7890/helm-schema-gen info found version: 0.0.2 for 0.0.2/Darwin/x86_64
karuppiah7890/helm-schema-gen info installed ./bin/helm-schema-gen
Updated plugin: schema-gen
```

> **Source code**

```go
// plugin_update.go
func (o *pluginUpdateOptions) run(out io.Writer) error {
    installer.Debug = settings.Debug
    debug("loading installed plugins from %s", settings.PluginsDirectory)
    // 1. load plugins from local plugin directory
    plugins, err := plugin.FindPlugins(settings.PluginsDirectory)
    if err != nil {
        return err
    }
    var errorPlugins []string

    for _, name := range o.names {
        // 2. iterate plugin names passed in command, and check whether it's the goal to update
        if found := findPlugin(plugins, name); found != nil {
            // 3. update the found plugin
            if err := updatePlugin(found); err != nil {
                errorPlugins = append(errorPlugins, fmt.Sprintf("Failed to update plugin %s, got error (%v)", name, err))
            } else {
                fmt.Fprintf(out, "Updated plugin: %s\n", name)
            }
        } else {
            errorPlugins = append(errorPlugins, fmt.Sprintf("Plugin: %s not found", name))
        }
    }
    if len(errorPlugins) > 0 {
        return errors.Errorf(strings.Join(errorPlugins, "\n"))
    }
    return nil
}
```

*1 load plugins from local plugin directory*

source code reading see above.

*2 iterate plugin names passed in command, and check whether it's the goal to update*

```go
// plugin_uninstall.go
// find plugin according to plugin name
func findPlugin(plugins []*plugin.Plugin, name string) *plugin.Plugin {
    for _, p := range plugins {
        if p.Metadata.Name == name {
            return p
        }
    }
    return nil
}
```

*3 update the found plugin*

```go
// plugin_update.go
func updatePlugin(p *plugin.Plugin) error {
    // get the target plugin local directory
    exactLocation, err := filepath.EvalSymlinks(p.Dir)
    if err != nil {
        return err
    }
    // get the absolute local directory of the target plugin
    absExactLocation, err := filepath.Abs(exactLocation)
    if err != nil {
        return err
    }

    // 3.1 instance a VCSInstaller object by the given local plugin directory which is actually a git repository
    i, err := installer.FindSource(absExactLocation)
    if err != nil {
        return err
    }
    // 3.2 update the local plugin directory file via git
    if err := installer.Update(i); err != nil {
        return err
    }

    debug("loading plugin from %s", i.Path())
    // load the target plugin directory to get Plugin object
    updatedPlugin, err := plugin.LoadDir(i.Path())
    if err != nil {
        return err
    }

    // run update hook if set
    return runHook(updatedPlugin, plugin.Update)
}
```

*3.1 instance a VCSInstaller object by the given local plugin directory which is actually a git repository*

```go
// installer.go
func FindSource(location string) (Installer, error) {
    // check whether the given plugin directory is a git repository, if yes, return VCSInstaller object
    installer, err := existingVCSRepo(location)
    if err != nil && err.Error() == "Cannot detect VCS" {
        return installer, errors.New("cannot get information about plugin source")
    }
    return installer, err
}


// vsc_installer.go
func existingVCSRepo(location string) (Installer, error) {
    // instance Repo object by the given directory, if not, err will be returned
    repo, err := vcs.NewRepo("", location)
    if err != nil {
        return nil, err
    }
    // construct a new VCSInstaller object
    i := &VCSInstaller{
        Repo: repo,
        base: newBase(repo.Remote()),
    }
    return i, nil
}
```

*3.2 update the local plugin directory file via git*

```go
// installer.go
func Update(i Installer) error {
    // check whether local plugin exists or not according to checking whether plugin directory exists or not
    if _, pathErr := os.Stat(i.Path()); os.IsNotExist(pathErr) {
        return errors.New("plugin does not exist")
    }
    // update local plugin dirctory file via git
    return i.Update()
}


// vsc_installer.go
func (i *VCSInstaller) Update() error {
    debug("updating %s", i.Repo.Remote())
    // check whether local git repository files are modified or not
    if i.Repo.IsDirty() {
        return errors.New("plugin repo was modified")
    }
    // update local git repository files
    if err := i.Repo.Update(); err != nil {
        return err
    }
    // check whether the local directory is a valid plugin directory
    if !isPlugin(i.Repo.LocalPath()) {
        return ErrMissingMetadata
    }
    return nil
}
```

> ## `helm plugin uninstall`

```bash
~/Workspace4go/src/github.com/helm/helm/cmd/helm(master*) » ./helm plugin uninstall -h
uninstall one or more Helm plugins

Usage:
  helm plugin uninstall <plugin>... [flags]

Aliases:
  uninstall, rm, remove

Flags:
  -h, --help   help for uninstall

......
-----------------------------------------------------------------------------------------------------------------------------------------------
~/Workspace4go/src/github.com/helm/helm/cmd/helm(master*) » ./helm plugin uninstall schema-gen --debug
plugin_uninstall.go:70: [debug] loading installed plugins from /Users/apple/Library/helm/plugins
Uninstalled plugin: schema-gen
```

> **Source code**

```go
// plugin_uninstall.go
func (o *pluginUninstallOptions) run(out io.Writer) error {
    debug("loading installed plugins from %s", settings.PluginsDirectory)
    // 1. load plugins from local plugin directory
    plugins, err := plugin.FindPlugins(settings.PluginsDirectory)
    if err != nil {
        return err
    }
    var errorPlugins []string
    for _, name := range o.names {
        // 2. iterate plugin names passed in command, and check whether it's the goal to update
        if found := findPlugin(plugins, name); found != nil {
            // 3. uninstall the found plugin
            if err := uninstallPlugin(found); err != nil {
                errorPlugins = append(errorPlugins, fmt.Sprintf("Failed to uninstall plugin %s, got error (%v)", name, err))
            } else {
                fmt.Fprintf(out, "Uninstalled plugin: %s\n", name)
            }
        } else {
            errorPlugins = append(errorPlugins, fmt.Sprintf("Plugin: %s not found", name))
        }
    }
    if len(errorPlugins) > 0 {
        return errors.Errorf(strings.Join(errorPlugins, "\n"))
    }
    return nil
}
```

*1 load plugins from local plugin directory*

source code reading see above.

*2 iterate plugin names passed in command, and check whether it's the goal to update*

source code reading see above.

*3 uninstall the found plugin*

```go
// plugin_uninstall.go
func uninstallPlugin(p *plugin.Plugin) error {
    // delete the plugin directory include the files
    if err := os.RemoveAll(p.Dir); err != nil {
        return err
    }
    // run plugin delete type hook if set
    return runHook(p, plugin.Delete)
}
```
