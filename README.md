# LTO Documentation - Command Line

## Terms and Definitions
### Device Numbering
- Connected devices such as LTO drives, CD drives, etc. are loaded via the Linux "SCSI generic" (sg) driver.
- Tape drives specifically will be loaded using the "SCSI tape" (st) driver as well.
- Connected devices of each type will receive a corresponding number (i.e. sg3, st0) starting at 0 and are accessible in the directory **/dev**.
- To keep this documentation generic the **#** sign is used in place of numbers in most places.
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
Depending on your specific needs, you must first install some or all of the following:
- `mt-st` - for tape control
- `mtx` - for auto-loaders and tape libraries
- `ltfs` - Linear Tape File System. Install the reference implementation from [here](https://github.com/LinearTapeFileSystem/ltfs "ltfs reference implementation")
- `bru` - Download and install Argest BRU Core from the [BRU website](https://www.tolisgroup.com/index.html "BRU website")

## Connecting Everything
- Connect and turn on your LTO drive
- To check that the drive is correctly recognized, run the command:
  ```
  cat /proc/scsi/scsi
  ```
- If you are using an auto-loader, you should see two tape-related devices (in this case associated with scsi6, but with different Lun numbers)

## Loading Tapes From a Tape Library
- Start by checking the status of the tape library. This will let you know which tapes are loaded into which slots.
  ```
  sudo mtx -f /dev/sg# status
  ```
- Next, load the tape from the desired slot into the desired drive.
- MTX load commands use the following structure:
  ```
  sudo mtx -f /dev/sg# load slot# device#
  ```
- If loading a tape into device 0 (the first device recognized), `device#` can be either `0` or ommitted from the command.
- In this example command, we are loading a tape from slot 6 into device 0
  ```
  sudo mtx -f /dev/sg5 load 6
  ```
- Once the tape loads, you can verify that the tape has loaded correctly using the following command:
  ```
  sudo mt -f /dev/nst0 status
  ```
- Near the top of the returned text you should see information about where in the tape you currently are.
- `File number=0, block number=0, partition=0` means that you are at the beginning of the tape.
- If everything worked you should also see `BOT ONLINE` at the bottom of the returned text.

## Unloading Tapes
- Before unloading, you should unmount any mounted ltfs file systems
- Run the following command:
  ```
  sudo umount /mnt/ltfs
  ```
- Now the file system is unmounted, you can rewind and unload the tape
  ```
  sudo mt -f /dev/nst0 rewoffl
  ```
- Once the tape finishes unloading, you can use mtx to return it to its original location:
  ```
  sudo mtx -f /dev/sg# unload
  ```

## LTFS
### Reading Tapes
- Create a mount point using the command:
  ```
  sudo mkdir /mnt/ltfs
  ```
- If you have not already loaded an LTO tape into your drive, make sure to do so before proceding.
- You can get the **sg#** of the tape drive from the ltfs command if you do not already have it from running the `cat` command:
  ```
  sudo ltfs -o device_list
  ```
- You should now be able to mount the ltfs file system using the following command (replace /dev/sg# with the corresponding location returned by the previous command):
  ```
  ltfs -o devname=/dev/sg# /mnt/ltfs
  ```
- To check that the tape was successfully mounted, run the `mount` command and you should see `/dev/sg# on /mnt/ltfs` near the bottom of the output.

## BRU
### Reading Tapes
- Check the status of your loaded tape using the MT command to make sure that you are at the beginning.
- Inspect the first block of the tape and get general info about it.
  ```
  sudo bru -g -b 2048k -f /dev/st# -v
  ```
  - **you may need to experiment with different values for `-b`. 2048k and 128k are common values for our tapes**
  - **use /dev/st# instead of /dev/nst# here so that you don't need to rewind if the value of `-b` was incorrect**
- Check the table of contents to get a list of all of the files on the tape.
  ```
  bru -t -b 2048k -f /dev/nst# -v
  ```
  - **This command will take a long time to run**
### Multi-Volume Tapes
- When BRU reaches the end of a volume it will prompt the user with the message `load volume # - press ENTER to continue on device 'dev/nst#'`
- You will then have the option to tell BRU how to proceed (**C**ontinue, **N**ew device, **Q**uit)
- If you have multiple drives connected, you can load the next volume into that drive and enter `N` to specify that BRU should now read from that device instead.
- If you only have a signle drive
  - open a new terminal tab or window.
  - Use mt and mtx to rewind and unload the tape.
  - Load the next volume into the drive.
  - Return to the original BRU prompt and press enter or type "c" and press enter.
  - BRU should continue processing on the new volume.

## Troubleshooting
- **MT Status `DR_OPEN IM_REP_EN`** - This message most likely means that you forgot to load the tape using the mtx command
