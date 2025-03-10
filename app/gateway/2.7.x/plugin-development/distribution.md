---
title: Plugin Development - (un)Installing your plugin
book: plugin_dev
chapter: 10
---

Custom plugins for Kong consist of Lua source files that need to be in the file
system of each of your Kong nodes. This guide will provide you with
step-by-step instructions that will make a Kong node aware of your custom
plugin(s).

These steps should be applied to each node in your Kong cluster, to ensure the
custom plugin(s) are available on each one of them.

## Packaging sources

You can either use a regular packing strategy (e.g. `tar`), or use the LuaRocks
package manager to do it for you. We recommend LuaRocks as it is installed
along with Kong when using one of the official distribution packages.

When using LuaRocks, you must create a `rockspec` file, which specifies the
package contents. For an example, see the [Kong plugin
template][plugin-template]. For more info about the format, see the LuaRocks
[documentation on rockspecs][rockspec].

Pack your rock using the following command (from the plugin repo):

    # install it locally (based on the `.rockspec` in the current directory)
    $ luarocks make

    # pack the installed rock
    $ luarocks pack <plugin-name> <version>

Assuming your plugin rockspec is called
`kong-plugin-my-plugin-0.1.0-1.rockspec`, the above would become;

    $ luarocks pack kong-plugin-my-plugin 0.1.0-1

The LuaRocks `pack` command has now created a `.rock` file (this is simply a
zip file containing everything needed to install the rock).

If you do not or cannot use LuaRocks, then use `tar` to pack the
`.lua` files of which your plugin consists into a `.tar.gz` archive. You can
also include the `.rockspec` file if you do have LuaRocks on the target
systems.

The contents of this archive should be close to the following:

    $ tree <plugin-name>
    <plugin-name>
    ├── INSTALL.txt
    ├── README.md
    ├── kong
    │   └── plugins
    │       └── <plugin-name>
    │           ├── handler.lua
    │           └── schema.lua
    └── <plugin-name>-<version>.rockspec


## Install the plugin

For a Kong node to be able to use the custom plugin, the custom plugin's Lua
sources must be installed on your host's file system. There are multiple ways
of doing so: via LuaRocks, or manually. Choose one of the following paths.

Reminder: regardless of which method you are using to install your plugin's
sources, you must still do so for each node in your Kong cluster.

### Via LuaRocks from the created 'rock'

The `.rock` file is a self contained package that can be installed locally
or from a remote server.

If the `luarocks` utility is installed in your system (this is likely the
case if you used one of the official installation packages), you can
install the 'rock' in your LuaRocks tree (a directory in which LuaRocks
installs Lua modules).

It can be installed by doing:

    $ luarocks install <rock-filename>

The filename can be a local name, or any of the supported methods, for example
`http://myrepository.lan/rocks/my-plugin-0.1.0-1.all.rock`.

### Via LuaRocks from the source archive

If the `luarocks` utility is installed in your system (this is likely the
case if you used one of the official installation packages), you can
install the Lua sources in your LuaRocks tree (a directory in which
LuaRocks installs Lua modules).

You can do so by changing the current directory to the extracted archive,
where the rockspec file is:

    $ cd <plugin-name>

And then run the following:

    $ luarocks make

This will install the Lua sources in `kong/plugins/<plugin-name>` in your
system's LuaRocks tree, where all the Kong sources are already present.

### Manually

A more conservative way of installing your plugin's sources is
to avoid "polluting" the LuaRocks tree, and instead, point Kong
to the directory containing them.

This is done by tweaking the `lua_package_path` property of your Kong
configuration. Under the hood, this property is an alias to the `LUA_PATH`
variable of the Lua VM, if you are familiar with it.

Those properties contain a semicolon-separated list of directories in
which to search for Lua sources. It should be set like so in your Kong
configuration file:

    lua_package_path = /<path-to-plugin-location>/?.lua;;

Where:

* `/<path-to-plugin-location>` is the path to the directory containing the
  extracted archive. It should be the location of the `kong` directory
  from the archive.
* `?` is a placeholder that will be replaced by
  `kong.plugins.<plugin-name>` when Kong will try to load your plugin. Do
  not change it.
* `;;` a placeholder for the "the default Lua path". Do not change it.

For example, if the plugin `something` is located on the file system and the
handler file is in the following directory:

    /usr/local/custom/kong/plugins/<something>/handler.lua

The location of the `kong` directory is `/usr/local/custom`, so the
proper path setup would be:

    lua_package_path = /usr/local/custom/?.lua;;

#### Multiple plugins

If you want to install two or more custom plugins this way, you can set
the variable to something like:

    lua_package_path = /path/to/plugin1/?.lua;/path/to/plugin2/?.lua;;

* `;` is the separator between directories.
* `;;` still means "the default Lua path".

You can also set this property via its environment variable
equivalent: `KONG_LUA_PACKAGE_PATH`.


## Load the plugin

1. Add the custom plugin's name to the `plugins` list in your
Kong configuration (on each Kong node):

        plugins = bundled,<plugin-name>

    Or, if you don't want to include the bundled plugins:

        plugins = <plugin-name>


    If you are using two or more custom plugins, insert commas in between, like so:

        plugins = bundled,plugin1,plugin2

    Or

        plugins = plugin1,plugin2

    You can also set this property via its environment variable equivalent:
    `KONG_PLUGINS`.

1. Update the `plugins` directive for each node in your Kong cluster.

1. Restart Kong to apply the plugin:

        kong restart

    Or, if you want to apply a plugin without stopping Kong, you can use this:

        kong prepare
        kong reload


## Verify loading the plugin

You should now be able to start Kong without any issue. Consult your custom
plugin's instructions on how to enable/configure your plugin
on a Service, Route, or Consumer entity.

1. To make sure your plugin is being loaded by Kong, you can start Kong with a
`debug` log level:

        log_level = debug

    or:

        KONG_LOG_LEVEL=debug

2. Then, you should see the following log for each plugin being loaded:

        [debug] Loading plugin <plugin-name>


## Remove a plugin

There are three steps to completely remove a plugin.

1. Remove the plugin from your Kong Service or Route configuration. Make sure
   that it is no longer applied globally nor for any Service, Route, or
   consumer. This has to be done only once for the entire Kong cluster, no
   restart/reload required.  This step in itself will make that the plugin is
   no longer in use. But it remains available and it is still possible to
   re-apply the plugin.

2. Remove the plugin from the `plugins` directive (on each Kong node).
   Make sure to have completed step 1 before doing so. After this step
   it will be impossible for anyone to re-apply the plugin to any Kong
   Service, Route, Consumer, or even globally. This step requires to
   restart/reload the Kong node to take effect.

3. To remove the plugin thoroughly, delete the plugin-related files from
   each of the Kong nodes. Make sure to have completed step 2, including
   restarting/reloading Kong, before deleting the files. If you used LuaRocks
   to install the plugin, you can do `luarocks remove <plugin-name>` to remove
   it.



## Distribute your plugin

The preferred way to do so is to use [LuaRocks](https://luarocks.org/), a
package manager for Lua modules. It calls such modules "rocks". **Your module
does not have to live inside the Kong repository**, but it can be if that's
how you'd like to maintain your Kong setup.

By defining your modules (and their eventual dependencies) in a [rockspec]
file, you can install those modules on your platform via LuaRocks. You can
also upload your module on LuaRocks and make it available to everyone!

Here is an [example rockspec][example-rockspec] using the `builtin`
build type to define modules in Lua notation and their corresponding file.

For more information about the format, see the LuaRocks
[documentation on rockspecs][rockspec].


## Troubleshooting

Kong can fail to start because of a misconfigured custom plugin for several
reasons:

`plugin is in use but not enabled`
: You configured a custom plugin from
  another node, and that the plugin configuration is in the database, but the
  current node you are trying to start does not have it in its `plugins`
  directive. To resolve, add the plugin's name to the node's `plugins`
  directive.

`plugin is enabled but not installed`
: The plugin's name is present in the `plugins` directive, but Kong can't load
the `handler.lua` source file from the file system. To resolve, make sure that
the [`lua_package_path`](/gateway/{{page.kong_version}}/reference/configuration/#development-miscellaneous-section)
directive is properly set to load this plugin's Lua sources.

`no configuration schema found for plugin`
: The plugin is installed and enabled in the `plugins` directive, but Kong is
unable to load the `schema.lua` source file from the file system. To resolve,
make sure that the `schema.lua` file is present alongside the plugin's
`handler.lua` file.

---

[rockspec]: https://github.com/keplerproject/luarocks/wiki/Creating-a-rock
[plugin-template]: https://github.com/Kong/kong-plugin
[example-rockspec]: https://github.com/Kong/kong-plugin/blob/master/kong-plugin-myplugin-0.1.0-1.rockspec
