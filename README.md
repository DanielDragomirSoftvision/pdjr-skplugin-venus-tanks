# pdjr-skplugin-venus-tanks

Inject Signal K tank data onto the host system D-Bus.

__pdjr-skplugin-venus-tanks__ is a plugin for Signal K servers running
on Venus OS.

The plugin was designed as a work-around for Venus' broken handling of
CAN-connected multi-channel tank monitoring devices (like the Maretron
FPM100 and the Garnet SeeLevel), but, it will, of course, operate with
Signal K tank data from any source.

__pdjr-skplugin-venus-tanks__ creates a D-Bus service for user-selected
Signal K tanks and echoes tank updates to the associated service, making
the tank data available for *inter-alia* rendering in the Venus GUI.

The plugin borrows GUI enhancements from Kevin Windrem's
[tank repeater](https://github.com/kwindrem/SeeLevel-N2K-Victron-VenusOS)
project to achieve the display shown below.
Of course, you can also roll your own GUI for tank data rendering.

![CCGX tank display](venus.png)

## Background

Support for tank monitoring in Venus OS is fundamentally broken.
The OS code implementing 'socketcan' services assumes one tank sensor
device per physical tank and generates a single D-Bus tank service for
each sensor device based on this false understanding.

Consequently, a tank service in Venus that represesents a multi-channel
tank sensor device is chaotically updated with data from all the tank
sensors that are connected to the multi-channel device resulting in a
GUI rendering of the data that is unintelligible.

Somewhere in this broken-ness Venus discards tank sensor instance
numbers making disaggregation of the garbled composite data at best
problematic and, for non-trivial tank installations, infeasible.

Even so, there have been a number of attempts at implementing fixes
and work-arounds based on the timing of fluid type and tank level
data updates but this does not allow disambiguation in installations
which have more than one tank of a particular fluid type.

If you have a multi-channel tank sensor on CAN and you have only one
tank of each fluid type, then look at Kevin Windrem's project (see
above) for a solution that doesn't involve Signal K).

## System requirements

__pdjr-skplugin-venus-tanks__ will most likely only be useful in
Signal K servers running under Venus OS.

If you require similar functionality but do not run Signal K under
Venus or you prefer to maintain D-Bus tank data with a native Venus
process, then consider using this
[alternative Python application](https://github.com/preeve9534/venus-signalk-tank-service).

## Installation

Download and install __pdjr-skplugin-venus-tanks__ using the _Appstore_
link in your Signal K Node server console.
The plugin can also be obtained from the 
[project homepage](https://github.com/preeve9534/pdjr-skplugin-venus-tanks)
and installed using
[these instructions](https://github.com/SignalK/signalk-server-node/blob/master/SERVERPLUGINS.md).

Once installed, you should must configure the plugin and enable its
operation in Signal K.

## Configuration

1. Login to your Signal K dashboard and navigate to
   _Server->Plugin config_->_Venus tanks_.

2. OPTION: "Use GUI enhancements?"

   This option is checked by default and tells the plugin that it should
   use this projects versions of ```OverviewMobile.qml``` and
   ```TileTank.qml``` rather than the system default versions.
   
   To achieve this the plugin will backup the existing versions of these
   files and install its enhanced versions in their place.
   The GUI changes implemented by the two ```.qml``` files prevent the
   native display of CAN derived tank data and adjust the 'mobile' display
   page to better accommodate the display of multiple tank entries.
   
   Unchecking this option will prevent the use of the plugin's enhanced
   GUI, if necessary reverting any system changes that may have been made
   previously by restoring the backed up ```.qml``` files.
   
3. OPTION: "Tanks"

   This array option is empty be default, telling __pdjr-skplugin-venus-tanks__
   to create a D-Bus service for every tank reported in Signal K.

   If you want the plugin to support just specific tanks or you need to
   adjust the tank capacity value reported by Signal K for one or more
   tanks then you must configure each tank explicitly by adding entries
   to the *Tanks* list, one for each tank you want the plugin to process.
   
   Each tank entry consists of *path* and *factor* settings.

   *path* specifies the Signal K tank path of the tank being configured
   and must be of the form 'tanks.*fluid-type*__.__*tank-instance*'.
   
   *factor* must be a decimal value greater than or equal to 0.0 (it
   defaults to 1.0).
   This feature is useful when individual tank sensor monitors are
   poorly configured or report in non-standard ways: some perhaps
   returning volume in litres, others in cubic-metres.
   Venus OS expects tank capacity to be reported in litres.
   
4. When you have made any changes you require, 
specified all the tanks you wish the plugin to process,
   update the configuration.
   
4. Reboot Signal K.

You can revert to processing all tanks by restoring the tank list to empty.

## Reviewing operation in Venus OS

Once the plugin is installed, configured and working you can log into
your Venus host and check things out.

On my five-tank system, for example:
```
$> dbus-spy
Services
com.victronenergy.battery.ttyO2                                          BMV-700
com.victronenergy.fronius
com.victronenergy.logger
com.victronenergy.modbustcp
com.victronenergy.qwacs
com.victronenergy.settings
com.victronenergy.system
com.victronenergy.tank.signalk_tank_0_3                   SignalK tank interface
com.victronenergy.tank.signalk_tank_0_4                   SignalK tank interface
com.victronenergy.tank.signalk_tank_1_1                   SignalK tank interface
com.victronenergy.tank.signalk_tank_1_2                   SignalK tank interface
com.victronenergy.tank.signalk_tank_5_0                   SignalK tank interface
com.victronenergy.vebus.ttyO1                          Quattro 24/8000/200-2x100
com.victronenergy.vecan.can0
```
If you choose one of the tank services you will be able to see tank
data updates as they occur. For example.
```
com.victronenergy.tank.signalk_tank_0_3
Capacity                                                                  1.7981
Connected                                                                      1
DeviceInstance                                                                 3
FirmwareVersion                                                                0
FluidType                                                                      0
HardwareVersion                                                                0
Level                                                                     89.104
Mgmt/Connection                                                   SignalK tank:3
Mgmt/ProcessName     /data/venus-signalk-tank-service-main/signalktankservice.py
Mgmt/ProcessVersion                                         1.0 on Python 2.7.14
ProductId                                                                      0
ProductName                                               SignalK tank interface
Remaining                                                                1.60218
```

You can see that my port-side fuel tank is reporting capacity in
cubic-metres: I should really adjust this to litres by setting a
multiplication *factor* of 1000.0 for this tank.

### Updating the Venus GUI for multiple tank display

If you have more than one or two tanks on your system, then you may
wish to make some changes to the Venus GUI in order to better 
display this data.

On my system I use the GUI tweaks implemented by @kwindrem in his
[tank repeater](https://github.com/kwindrem/SeeLevel-N2K-Victron-VenusOS)
project.
You can get these changes by clicking the above link and following
the installation instructions bearing in mind that:
   
1. When you run the repeater project setup script, respond to the
   first prompt with 'a' (Activate) and subsequent prompts with 'y'.
   This will activate @kwindrem's repeater (we don't need this) and
   install his GUI changes (we do need these).
   
2. You should then run the repeater project setup script again,
   responding to the first prompt with 'd' (Disable) and subsequent
   prompts with 'y'.
   This will disable @kwindrem's repeater, but leave his GUI changes
   in place.

## Acknowledgements

Thanks to @kwindrem for making this a whole lot easier than it might have
been by designing his repeater software in a way which allows its components
to be leveraged by others.

Thanks to @mvader at Victron for his support and encouragement and for
inviting me to think about a Signal K plugin variant of
[venus-signalk-tank-handler](https://github.com/preeve9534/venus-signalk-tank-service/).

## Author

Paul Reeve \<<preeve@pdjr.eu>\>
