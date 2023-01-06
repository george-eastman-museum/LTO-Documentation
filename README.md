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
- However, there is a difference in the tape device functionality denoted by each.
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
  sudo mtx -f /dev/sg# load slot# drive#
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
- If everything worked you should also see `ONLINE` in the bottom of the returned text.

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
- If unloading a tape from a drive other than drive 0, you can specify stlot# and drive# similar to the `load` command:
  ```
  sudo mtx -f /dev/sg# unload 9 1
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
  sudo ltfs -o devname=/dev/sg# /mnt/ltfs
  ```
- To check that the tape was successfully mounted, run the `mount` command and you should see `/dev/sg# on /mnt/ltfs` near the bottom of the output.
### Writing Tapes
- Before you can mount or write to a new tape, you must first format it as an LTFS tape.
- Enter the following `mkltfs` command to format a new tape that has been loaded into your drive:
  ```
  sudo mkltfs --device=/dev/sg# --tape-serial=<enter the 6 alphanumeric values from the label>
  ```
- Once this is complete, you should now be able to mount the LTFS tape using the commands from the above **Reading Tapes** section.
- The LTFS reference implementation page recommends using a utility that optimizes writes for tape when copying files.
- The `ltfs_ordered_copy` utility should be included with the LTFS reference implementation and can be used for this purpose.
- The following command should perform a simple recursive copy that preserves as many file attributes (creation date, etc.) as possible:
  ```
  ltfs_ordered_copy -a -v input output
  ```
- (**TODO**: look into the mkltfs --rules command, which sets rules for choosing files to write to the index partition, and how the index partition is used by other LTFS software we use)
- If you are unsure whether an `ltfs_ordered_copy` command will function the way you expect, you can always test the command out writing to hard drive first (the command will warn you that the destination is not an LTFS file system, but should still copy the files).
### Reformatting/Wiping Tapes
- If you need to erase the LTFS formatting on a tape, you can run the following command (THIS WILL DELETE ALL THE FILES):
  ```
  sudo mkltfs -d /dev/sg# -w
  ```

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
- If you only have a single drive
  - open a new terminal tab or window.
  - Use mt and mtx to rewind and unload the tape.
  - Load the next volume into the drive.
  - Return to the original BRU prompt and press enter or type "c" and press enter.
  - BRU should continue processing on the new volume.
### Restoring Files From Tape to a Different Directory
- The `-x` command is used to extract/restore files from a BRU LTO tape.
- Start by checking what the path information on the LTO tape looks like using the `-t` command.
- In order to change the path when restoring the files you will need to create a translation file.
- A translation file is just a simple text file listing the original path information and the new path information that you want to replace that with, separated by a space.
  - Example1: /Volumes/Original_Folder /mnt/New_Network_Share/LTO_Output
  - Example2: /Volumes/Original_Folder '/mnt/New Network Share with Spaces/LTO Output'
- Save your translation file somewhere that will be easy to reference later on.
- If you are on Linux and writing to an SMB share, you may want to manually mount the share to an easier to access location, such as in **/mnt** to avoid issues with reading the output path correctly.
- If you have not already created an output folder that you want to write your files to, do so now and `cd` to that directory.
  ```
  mkdir /mnt/New_Network_Share/LTO_Output
  cd /mnt/New_Network_Share/LTO_Output
  ```
- Run the following command to combine outputting to a relative directory with substituting paths using your translation file.
  ```
  sudo bru -xvvv -b 128k -PA -T/translation_file_path/translation_file -f /dev/nst#
  ```
  - Including `-T/translation_file_path/tranlation_file` will substitute any specified paths from the LTO tape with the paths you provided in your translation file. 
  - If there are any paths on the LTO tape that you did not include in your translation file, running the command from the desired output folder with the `-PA` command   ensures that they will still be written to that folder with their full original paths appended to the end of the current folder.
  - **TODO** - address writing /Volume in a better way (translation file should probably be able to cover this)
### Restoring A Specific File or Directory
- You can modify the restore command to only restore a specific folder using the `-E` command.
- Start by getting the path to the file you want to restore from the bru server catalog.
- **Note that paths with spaces need to be enclosed in quotes for the command to work**
- `cd` to the folder that you want to output to.
- Run the following extract command:
  ```
  sudo bru -xvvv -b 128k -PA -E "/path/to/file" -f /dev/nst0
  ```
- The restore command also works with wildcards if you want to restore the contents of an entire directory:
  ```
  sudo bru -xvvv -b 128k -PA -E "/path/to/folder/*" -f /dev/nst0
  ```
- **TODO** - more work to figure out translation files and seeking.

## Troubleshooting
- **MT Status `DR_OPEN IM_REP_EN`** - This message most likely means that you forgot to load the tape using the mtx command
