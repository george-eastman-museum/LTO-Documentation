# LTO Documentation

## Terms and Definitions
### Device Numbering
- To keep this documentation generic the '#' sign is used in place of numbers in most places.
- For example, `/dev/sg5` or `/dev/st0` will instead be referred to as `/dev/sg#` and `/dev/st#` as the number may vary depending on your configuration.
### **/dev/st0** vs **/dev/nst0**
- Both of these can be used to refer to the first tape device connected to your computer.
- However, there is a difference in the tape device functionality that denoted by each.
- `/dev/st#` is a rewinding device.
- `/dev/nst#` is a non-rewinding device.
- If you want the tape to rewind after an action is performed, it should be referenced as `/dev/st#`
- If you want to leave off at the end of your last action on the tape (for example when writing files to prevent overwriting data), you would use `/dev/nst#`
- As long as you remember to manually rewind or seek to the correct location when necessary, `/dev/nst#` should always work.

## Configuring Everything
### Prerequisites
#### Linux
Depending on your specific needs, you will need to install some or all of the following:
- `mt-st` - for tape control
- `mtx` - for auto-loaders and tape libraries
- `ltfs`
- `bru`

## Loading Tapes Using the Command Line

## Unloading Tapes

## Mounting the LTFS File System

## Using BRU

## Troubleshooting
### DR_OPEN IM_REP_EN
- This message most likely means that you forgot to load the tape using the mtx command
