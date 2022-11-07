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

## Installation - Prerequisites
### Linux
Depending on your specific needs, you will need to install some or all of the following:
- `mt-st` - for tape control
- `mtx` - for auto-loaders and tape libraries
- `ltfs` - Install the reference implementation from [here](https://github.com/LinearTapeFileSystem/ltfs "ltfs reference implementation")
- `bru` - Download and install Argest BRU Core from the [BRU website](https://www.tolisgroup.com/index.html "BRU website")

## Verifying Functionality
- Connect and turn on your LTO drive
- To check that the drive is correctly recognized, run the command:
``` cat /proc/scsi/scsi ```
- If you are using an auto-loader, you should see two tape-related devices (in this case associated with scsi6, but with different Lun numbers)

## Loading Tapes From a Tape Library Using the Command Line
- Start by checking
```sudo mtx -f /dev/sg# status```
- Next, load the tape from the desired slot into the desired drive.
- MTX load commands use the following structure:
```sudo mtx -f /dev/sg# load slot# device#```
- If loading a tape into device 0 (the first device recognized), `device#` can be either `0` or ommitted from the command.
- In this example command, we are loading a tape from slot 6 into device 0
```sudo mtx -f /dev/sg5 load 6```
- Once the tape loads, you can verify that the tape has loaded correctly using the following command:
```sudo mt -f /dev/nst0 status```
- If everything worked, you should see `BOT ONLINE` at the bottom of the returned text.

## Unloading Tapes

## Mounting the LTFS File System

## Using BRU

## Troubleshooting
### DR_OPEN IM_REP_EN
- This message most likely means that you forgot to load the tape using the mtx command
