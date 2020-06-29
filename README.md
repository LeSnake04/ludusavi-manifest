# Ludusavi Manifest
**This project is still a prototype! It is subject to change, and not all data is available yet.**

The Ludusavi Manifest format is a YAML structure for defining the location of
game save data and other files that are of interest to back up. Although this
project was started for use by [Ludusavi](https://github.com/mtkennerly/ludusavi),
the goal is for the manifest format to be generic enough for any game backup tool
to implement, while leaving room for new fields and functionality over time.

This repository contains the [primary manifest](data/manifest.yaml), which is
compiled from data on [PCGamingWiki](https://www.pcgamingwiki.com/wiki/Home),
along with accessing the Steam API for game installation directory names.
If you find any data that is missing or incorrect, please contribute to the wiki,
and such changes will be incorporated into the primary manifest periodically.
There is also a list of [games without any info on what to back up](data/missing.md).

Game developers may include a secondary manifest (named `.ludusavi.yaml`) with
their games, so that backup tools can automatically detect and use it to discover
what files need to be backed up for save data and configuration.

## Format
For the schema, refer to [schema.yaml](data/schema.yaml). Note that the primary
manifest is validated with [schema.strict.yaml](data/schema.strict.yaml), which
additionally specifies enums for some fields. However, tools should implement
[schema.yaml](data/schema.yaml), so that new values in the manifest do not break
older tools.

Here is an example:

```yaml
An Example Game:
  files:
    <base>/saves:
      tags:
        - save
    <base>/settings.json:
      when:
        - os: windows
        - os: linux
      tags:
        - config
    <base>/other:
      when:
        - os: mac
          store: steam
  installDir:
    AnExampleGame: {}
  registry:
    <regHkcu>/Software/An Example Game:
      tags:
        - save
        - config
  steam:
    id: 123
```

This means:

* `<base>/saves` will be backed up on any system.
* `<base>/settings.json` will be backed up if you're using Windows or Linux.
* `<base>/other` will be backed up if you're using Mac and Steam.
* On Windows, the registry path `<regHkcu>/Software/An Example Game` will be
  backed up.

Paths can include these placeholders:

| placeholder         | meaning                                                                   |
|---------------------|---------------------------------------------------------------------------|
| `<root>`            | a directory where games are installed (configured in backup tool)         |
| `<game>`            | an `installDir` (if defined) or the game's canonical name in the manifest |
| `<base>`            | shorthand for `<root>/**/<game>`                                          |
| `<home>`            | current user's home directory in the OS (`~`)                             |
| `<storeUserId>`     | current user's ID in the game store                                       |
| `<osUserName>`      | current user's name in the OS                                             |
| `<winAppData>`      | `%APPDATA%` on Windows                                                    |
| `<winLocalAppData>` | `%LOCALAPPDATA%` on Windows                                               |
| `<winPublic>`       | `%PUBLIC%` on Windows                                                     |
| `<winProgramData>`  | `%PROGRAMDATA%` on Windows                                                |
| `<winDir>`          | `%WINDIR%` on Windows                                                     |
| `<xdgData>`         | `$XDG_DATA_HOME` on Linux                                                 |
| `<xdgConfig>`       | `$XDG_CONFIG_HOME` on Linux                                               |
| `<regHkcu>`         | `HKEY_CURRENT_USER` in the Windows registry                               |
| `<regHklm>`         | `HKEY_LOCAL_MACHINE` in the Windows registry                              |

## Implementation
Tools must implement the following in addition to respecting the schema:

* For paths, first substitute the placeholders, then evaluate as a glob.
  Because of treating paths as globs, a path may match multiple files or
  directories.
* When a path identifies a folder, the backup includes all of its files
  and subdirectories recursively.
* Relative paths must be resolved relative to the location of the manifest file.
  This is important for secondary manifests to work correctly without
  hard-coding their location.
* If a tool supports secondary manifests, they must be automatically detected
  when they are named `.ludusavi.yaml` and exist anywhere within a configured
  root.

Tools may also:

* Use store-specific logic to narrow down the `**` in `<base>`. For example,
  with Steam, it would be `<root>/steamapps/common/<game>`.

The latest version of the primary manifest can be downloaded from
https://raw.githubusercontent.com/mtkennerly/ludusavi-manifest/master/data/manifest.yaml .
To check for updates:

* Store the value of the `ETag` header for the last downloaded version.
* Send a GET request to the URL with the `If-None-Match` header set to the
  last known `ETag` value.
* If the response code is 304, then no update is needed.
* If the response code is 200, then store the new `ETag` value.

## Development
Please refer to [CONTRIBUTING.md](CONTRIBUTING.md).
