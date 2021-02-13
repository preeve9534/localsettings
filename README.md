# Local setting

[![Build Status](https://travis-ci.org/victronenergy/localsettings.svg?branch=master)](https://travis-ci.org/victronenergy/localsettings)

[Venus OS](https://github.com/victronenergy/venus/wiki) implements
a D-Bus service ```com.victronenergy.settings``` (the 'settings service')
for management of any application's persistent (that is non-volatile)
data or settings.

Giving a central service control over persistent settings means that
applications are relieved of the burden of managing their non-volatile
data.
It also means consistency: there is one place to see all the settings;
one process maintaining a log of settings changes (under
/log/localsettings/); and one mechanism for resetting all settings to
factory-default.

If you are writng a program that creates a new D-Bus service then
you must use the settings service to acquire a unique DeviceInstance
number for your service.

If you are writing a program of any sort that needs to maintain its
own persistent data or requires access to the persistent data of other
programs then the settings service must be used as a proxy for all
such access.

At the time of writing, a Python library provides an interface to the
settings service.

### D-Bus API

```com.victronenergy.settings``` exposes the following API.

### AddSetting

There are two usage patterns.

### AddSetting(path, tuple)

Where path should be set to



As a side-effect this will create the D-Bus path
/Settings/Devices/UniqueDeviceNumber/ClassAndVrmInstance and set its
value appropriately.

Add a new persistent setting and/or acquire a unique DeviceInstance
number.

Parameters:
- Groupname
- Setting name (can contain subpaths, for example display/brightness.
  /display/brightness will work as well and has the same effect)
- Default value
- Type ('i' - integer, 'f' - float, 's' - string)
- Min value
- Max value

Return code:
* 0 = OK
* Negative, see AddSettingError in the source for details

Notes:
* Set both min and max to 0 to work without a min and max value
* Executing AddSetting for a path that already exists will not cause the existing
  value to be changed. In other words, it is safe to call AddSetting without first
  checking if that setting, aka path, is already there.
* Vrm Device Instances: Localsettings can assign unique numbers per device class to a device. The single parameter for them is a tuple: `class:instance`, `battery:1` for example. When adding the special setting, /Settings/Devices/UniqueDeviceNumber/ClassAndVrmInstance, it will be set to an unique one. So if the default is set to battery:1 and that one already existed, it will get the next free unique number, and get set to `battery:2` for example.



#### AddSettings
This dbus method call allows to add multiple settings at once which
saves some roundtrips if there are many settings like the gui has.

Unlike the AddSetting, it doesn't make a distinction between groups
and setting and only accepts a single path. The type is based on the
(mandatory) default value and doesn't need to be passed. min, max and silent
are optional.

Required parameters:
- "path" the (relative) path for the setting. /Settings/Display/Brightness when called \
  on / or /Display/Brightness when called on /Settings etc.
- "default" the default value of the setting. The type of the default values determines \
  the setting type.

Optional parameters:
- "min"
- "max"
- "silent" don't log changes

For each entry, at least error and path are returned (unless it
wasn't passed). The actual value is returned when no error occured.

Commandline examples:

```
dbus com.victronenergy.settings / AddSettings \
'%[{"path": "/Settings/Test", "default": 5}, {"path": "/Settings/Float", "default": 5.0}]'

[{'error': 0, 'path': '/Settings/Test', 'value': 1},
 {'error': 0, 'path': '/Settings/Float', 'value': 5.0}]

or on /Settings:

dbus com.victronenergy.settings /Settings AddSettings \
'%[{"path": "Test", "default": 5}, {"path": "Float", "default": 5.0}]'

[{'error': 0, 'path': 'Test', 'value': 1},
 {'error': 0, 'path': 'Float', 'value': 5.0}]

or for testing:

dbus com.victronenergy.settings /Settings/Devices AddSettings '%[{"path": "a/ClassAndVrmInstance", "default": "battery:1"}, {"path": "b/ClassAndVrmInstance", "default": "battery:1"}]'
[{'error': 0, 'path': 'a/ClassAndVrmInstance', 'value': 'battery:1'},
 {'error': 0, 'path': 'b/ClassAndVrmInstance', 'value': 'battery:2'}
```

#### RemoveSettings
Removes all settings for a given array with paths

returns an array with 0 for success and -1 for failure.

#### GetValue
Returns the value. Call this function on the path of which you want to read the
value. No parameters.

#### GetText
Same as GetValue, but then returns str(value).

#### SetValue
Call this function on the path of with you want to write a new value.

Return code:
*  0 = OK
* -1 = Error

#### GetMin
See source code

#### GetMax
See source code

#### SetDefault
See source code

## Usage examples and libraries
### Command line
Typical implementation in your code in case you want some settings would be:

1. Always do an AddSetting in the start of your code. This will make sure the setting
exists, and will not overwrite an existing value. Example with commandline tool:

    dbus -y com.victronenergy.settings /Settings AddSetting GUI Brightness 50 i 0 100

    In which 50 is the default value, i the type, 0 the minimum value and 100 the maximum value.
2. Then read it:

    dbus -y com.victronenergy.settings /Settings/GUI/Brightness GetValue

3. Or write it:

    dbus -y com.victronenergy.settings /Settings/GUI/Brightness SetValue 50

4. dbus com.victronenergy.settings /Settings AddSettings \
'%[{"path": "Int", "default": 5}, {"path": "Float", "default": 5.0}, {"path": "String", "default": "string"}]'

5. dbus com.victronenergy.settings /Settings RemoveSettings '%["Int", "Float", "String"]'

Obviously you won't be calling dbus -y everytime, but implement some straight dbus
interface in your code. Below are some examples for different languages.

### Python

To do this from Python, see import settingsdevice.py from velib_python. Below code gives a good example:

Somewhere in your init code, make the settings:

    from settingsdevice import SettingsDevice  # available in the velib_python repository
    settings = SettingsDevice(
        bus=dbus.SystemBus() if (platform.machine() == 'armv7l') else dbus.SessionBus(),
        supportedSettings={
            'loggingenabled': ['/Settings/Logscript/Enabled', 1, 0, 1],
            'proxyaddress': ['/Settings/Logscript/Http/Proxy', '', 0, 0],
            'proxyport': ['/Settings/Logscript/Http/ProxyPort', '', 0, 0],
            'backlogenabled': ['/Settings/Logscript/LogFlash/Enabled', 1, 0, 1],
            'backlogpath': ['/Settings/Logscript/LogFlash/Path', '', 0, 0],  # When empty, default path will be used.
            'interval': ['/Settings/Logscript/LogInterval', 900, 0, 0],
            'url': ['/Settings/Logscript/Url', '', 0, 0]  # When empty, the default url will be used.
            },
        eventCallback=handle_changed_setting)

Have a callback some where, in above code it is handle_changed_setting. That is how
you'ĺl be informed that someone, for example the GUI, has changed a setting. Above
function has this definition:

    def handle_changed_setting(setting, oldvalue, newvalue):
        print 'setting changed, setting: %s, old: %s, new: %s' % (setting, oldvalue, newvalue)


To read or write a setting yourself, do:

    settings['url'] = ''


Or read from it:

    print(settings['url'])

### QT / C++
todo.

### C
todo.

## Running on a linux PC
It is also possible to run localsettings on a linux PC, which may be convenient for testing other
CCGX services.

The localsettings script requires python dbus, gobject and xml support (for python 2.7). On a
debian system install the packages python-dbus, python-gobject and python-lxml.
