# Snapraid AIO Script
The definitive all-in-one [SnapRAID](https://github.com/amadvance/snapraid) script on Linux. I hope you'll agree :).

There are many SnapRAID scripts out there, but none had the features I wanted. So I made my own, inspired by existing solutions.

It is meant to be run periodically (daily), do the heavy lifting and send an email you will actually read.

Supports single and dual parity configurations. It is highly customizable and has been tested with Debian 11/12 and [OpenMediaVault 6/7](https://github.com/openmediavault/openmediavault).

Contributions are welcome!

# Table of Contents

- [Highlights](#highlights)
  * [How it works](#how-it-works)
    + [Additional Features](#additional-features)
  * [Customization](#customization)
    + [Available options](#available-options)
  * [A nice email report](#a-nice-email-report)
- [Requirements](#requirements)
- [Installation](#installation)
  * [First Run](#first-run)
  * [OMV and SnapRAID plugin](#omv-and-snapraid-plugin)
  * [Installing `hd-idle`](#installing-hd-idle-for-automatic-disk-spindown)
- [Upgrade](#upgrade)
- [Known Issues](#known-issues)
- [Star History](#star-history)
- [Credits](#credits)

# Highlights

## How it works
- After some preliminary checks, the script will execute `snapraid diff` to figure out if parity info is out of date, which means checking for changes since the last execution. During this step, the script will ensure drives are fine by reading parity and content files. 
- One of the following will happen:     
    - If parity info is out of sync **and** the number of deleted or changed files exceed the threshold you have configured it **stops**. You may want to take a look to the output log.
    - If parity info is out of sync **and** the number of deleted or changed files exceed the threshold, you can still **force a sync** after a number of warnings. It's useful If you often get a false alarm but you're confident enough. This is called "Sync with threshold warnings"
    	- Instead of forcing a sync based on the number of deleted files, you may consider the `ADD_DEL_THRESHOLD` feature, by allowing a sync that would otherwise violate the delete threshold, if the ratio of added to deleted files is greater than the value set. 
    - If parity info is out of sync **but** the number of deleted or changed files did not exceed the threshold, it **executes a sync** to update the parity info.
- When the parity info is in sync, either because nothing has changed or after a successfully sync, it runs the `snapraid scrub` command to validate the integrity of the data, both the files and the parity info. If sync was cancelled or other issues were found, scrub will not be run. 
    - Note that each run of the scrub command will validate only a (configurable) portion of parity info to avoid having a long running job and affecting the performance of the server. 
    - Scrub frequency can also be customized in case you don't want to do it every time the script runs. 
    - It is still recommended to run scrub frequently. 
- Extra information can be added, like SnapRAID's disk health report or SnapRAID array status.  
- When the script is done sends an email with the results, both in case of error or success, and triggers any 3rd party notifications configured.

### Additional Features
- Docker container management
	- Manage containers before SnapRAID operations and restore them when finished. It avoids nasty errors aboud data being written during SnapRAID sync.
	- Support for local and remote Docker instances. Also manage multiple remote Docker instances at once. 
		- **Note:** Remote Docker instances require SSH passwordless access.
	- You can either choose to pause or stop your containers.
- Ignore Files for thresholds warnings
 	- If you have many files that change but you want to ignore them (e.g related to frequent backup rotation) you can do so to decrease counts for your thresholds.   
- Custom Hooks 
	- Define shell commands or scripts to run before and after SnapRAID operations.
- Multiple configuration files
  	- Use a different configuration file when running the script instead of the default config
- 3rd Party notification support
	- Can be used to track script execution time, status and promptly alert about errors.
	- Supports [Healthchecks.io](https://healthchecks.io), and 100+ services via Apprise (Telegram, Discord, Slack, etc) 
    - You can also get notified with the `Snapraid SMART log` and `Snapraid Status`
    - You can get the full email report if a warning is issued (breached threshold)
	- Notification Hook: if your favourite notification service is not supported by this script, you can use a custom notification command or another mail binary **[deprecated, use Apprise]**
- Important messages are also sent to the system log.
- Emails reports are still the best way to get detailed but readable information.

## Customization
Many options can be changed to your taste, their behavior is documented in the config file.
If you don't know what to do, I recommend using the default values and see how it performs.

### Available options
- Sync options
	- Sync always (Forced Sync).
	- Sync after a number of breached threshold warnings. 
	- Sync only if thresholds warnings are not breached (enabled by default).
	- Sync even if the delete threshold has been breached, but the ratio of added to deleted files is greater than the value set. 
	- User definable thresholds for deleted and updated files.
- Scrub options 
	- Enable or disable scrub job.
	- Delayed option, disabled by default. Run scrub only after a number of script executions, e.g. every 7 times. If you don't want to scrub your array every time, this one is for you.
	- Data to be scrubbed - by default 5% older than 10 days.
	- Scrub new data - scrub the data that was just added by the sync.
- Pre-hashing - enabled by default. Mitigates the lack of ECC memory, reading data twice to avoid silent read errors.
- Force zero size sync -  disabled by default. Forces the operation of syncing a file with zero size that before was not. Use with caution!
- Ignore Files for thresholds warnings - disabled by default
  	- It is called `IGNORE_PATTERN` in the config file.
  	- Ignore unwanted updated/changed/deleted files defined on their path(s), that would otherwise increase counts and breach your tresholds.
  	- This is an advanced feature as it requires the use of regular expressions.  Use with caution!
  	- More information can be found in the config file.
- Snapraid Status - disabled by default. Shows the status of the array.
	- This info can also be sent to Telegram or Discord
- SMART Log - enabled by default. A SnapRAID report for disks health status.
  	- This info can also be sent to Telegram or Discord
- Verbosity option - disabled by default. When enabled, includes the TOUCH and DIFF commands output. Please note email will be huge and mostly unreadable.
- SnapRAID Output (log) retention - disabled by default (log is overriden every run)
	- Detailed output retention for each run
	- You can choose the amount of days and the path, by default set to the user home 
- Healthchecks.io integration
   - The script will report to Healthchecks.io when is started and when is completed. If there's a failure it's included as well.
   -  This service will also show how much time the script takes to complete.
  	- If the script ends with a **_WARNING_** message, it will report **_DOWN_** to Healthchecks.io, if the message is **_COMPLETED_** it will report **_UP_**. 
- Notifications services via [Apprise](https://github.com/caronc/apprise)
	- Send notifications to Telegram, Discord, Slack... you name it! Apprise supports 100+ services! Configuration is simple, instructions [are here](https://github.com/caronc/apprise/wiki).
	- If you don't read your emails every day, this is a great one for you, since you can be quickly informed if things go wrong. 
  	- The script will report when it's started and when it's completed. If there's a failure, it's notified as well.
  	- You can choose to get the output attached if there's a warning (only supported by some services, check Apprise docs)
- Email report via [Apprise](https://github.com/caronc/apprise)
  	- If your distro doesn't have `mailx` or `sendmail`, you can use Apprise to deliver your email reports
   - You can choose to get the output attached if there's a warning
- Notification Hook **[deprecated, use Apprise]**
	- Made for external services or mail binaries with different commands than `mailx`.
	- Configure the path of the script or the mail binary to be invoked.
	- You can still use native services since it only replaces the standard email.
 	- You can choose to run the final hook before or after the spindown command, if configured.
- Update Check - enabled by default
  	- The script will check via GitHub if there's an update and alert the user via the configured notification systems
  	- If you don't like this, it can be disabled
- Docker Container management
	- A list of containers you want to be interrupted before running actions and restored when completed.
   	- Docker mode - choose to pause/unpause or to stop/restart your containers
   	- Docker remote - if docker is running on a remote machine
- Command line arguments
  	- Can be used to override the default behaviour.
  	- You can force a sync by adding  `--force-sync`
  	- You can specify another config file when running the script by adding  `--config /home/alternate_config.conf`
- Custom Hooks
	- Commands or scripts to be run before and after SnapRAID operations.
	- Option to display friendly name to in the email output
- Spindown - spindown disks after the script has completed operations. Uses a rewritten version of [hd-idle](https://github.com/adelolmo/hd-idle).

 
You can also change more advanced options such SnapRAID binary location, log file location and mail binary.

## A nice email report
This script produces emails that don't contain a list of changed files to improve clarity.

You can re-enable full output in the email by switching the option `VERBOSITY`. The full report is available in `/tmp/snapRAID.out` but will be replaced after each run, or deleted when the system is shut down. You can enable the retention policy to keep logs for some days and customize the folder location.

Here's an example email report. 


```markdown
## [COMPLETED] DIFF + SYNC + SCRUB Jobs (SnapRAID on omv-test.local)
SnapRAID Script Job started [Tue 20 Apr 11:43:37 CEST 2021]
Running SnapRAID version 11.5
SnapRAID AIO Script version X.YZ

----------

## Preprocessing
Healthchecks.io integration is enabled.
Configuration file found.
Checking if all parity and content files are present.
All parity files found.
All content files found.
Docker containers management is enabled.

### Stopping Containers [Tue 20 Apr 11:43:37 CEST 2021]
Stopping Container - Code-server
code-server
Stopping Container - Portainer
portainer

----------

## Processing
### SnapRAID TOUCH [Tue 20 Apr 11:43:37 CEST 2021]
Checking for zero sub-second files.
No zero sub-second timestamp files found.
TOUCH finished [Tue 20 Apr 11:43:38 CEST 2021]

### SnapRAID DIFF [Tue 20 Apr 11:43:38 CEST 2021]
DIFF finished [Tue 20 Apr 11:43:38 CEST 2021]
**SUMMARY of changes - Added [0] - Deleted [0] - Moved [0] - Copied [0] - Updated [1]**
There are no deleted files, that's fine.
There are updated files. The number of updated files (1) is below the threshold of (500).
SYNC is authorized. [Tue 20 Apr 11:43:38 CEST 2021]

### SnapRAID SYNC [Tue 20 Apr 11:43:38 CEST 2021]
Self test...  
Loading state from /srv/dev-disk-by-label-DISK1/snapraid.content...  
Scanning disk DATA1...  
Scanning disk DATA2...  
Using 0 MiB of memory for the file-system.  
Initializing...  
Hashing...  
SYNC - Everything OK  
Resizing...  
Saving state to /srv/dev-disk-by-label-DISK1/snapraid.content...  
Saving state to /srv/dev-disk-by-label-DISK2/snapraid.content...  
Saving state to /srv/dev-disk-by-label-DISK3/snapraid.content...  
Saving state to /srv/dev-disk-by-label-DISK4/snapraid.content...  
Verifying /srv/dev-disk-by-label-DISK1/snapraid.content...  
Verifying /srv/dev-disk-by-label-DISK2/snapraid.content...  
Verifying /srv/dev-disk-by-label-DISK3/snapraid.content...  
Verifying /srv/dev-disk-by-label-DISK4/snapraid.content...  
Verified /srv/dev-disk-by-label-DISK4/snapraid.content in 0 seconds  
Verified /srv/dev-disk-by-label-DISK3/snapraid.content in 0 seconds  
Verified /srv/dev-disk-by-label-DISK2/snapraid.content in 0 seconds  
Verified /srv/dev-disk-by-label-DISK1/snapraid.content in 0 seconds  
Syncing...  
Using 32 MiB of memory for 32 cached blocks.  
    DATA1 12% | *******  
    DATA2 82% | ************************************************  
   parity  0% |   
 2-parity  0% |   
     raid  1% | *  
     hash  1% |   
    sched 11% | ******  
     misc  0% |   
              |____________________________________________________________  
                            wait time (total, less is better)  
SYNC - Everything OK  
Saving state to /srv/dev-disk-by-label-DISK1/snapraid.content...  
Saving state to /srv/dev-disk-by-label-DISK2/snapraid.content...  
Saving state to /srv/dev-disk-by-label-DISK3/snapraid.content...  
Saving state to /srv/dev-disk-by-label-DISK4/snapraid.content...  
Verifying /srv/dev-disk-by-label-DISK1/snapraid.content...  
Verifying /srv/dev-disk-by-label-DISK2/snapraid.content...  
Verifying /srv/dev-disk-by-label-DISK3/snapraid.content...  
Verifying /srv/dev-disk-by-label-DISK4/snapraid.content...  
Verified /srv/dev-disk-by-label-DISK4/snapraid.content in 0 seconds  
Verified /srv/dev-disk-by-label-DISK3/snapraid.content in 0 seconds  
Verified /srv/dev-disk-by-label-DISK2/snapraid.content in 0 seconds  
Verified /srv/dev-disk-by-label-DISK1/snapraid.content in 0 seconds

SYNC finished [Tue 20 Apr 11:43:40 CEST 2021]

### SnapRAID SCRUB [Tue 20 Apr 11:43:40 CEST 2021]
Self test...  
Loading state from /srv/dev-disk-by-label-DISK1/snapraid.content...  
Using 0 MiB of memory for the file-system.  
Initializing...  
Scrubbing...  
Using 48 MiB of memory for 32 cached blocks.  
    DATA1  2% | *  
    DATA2 18% | **********  
   parity  0% |   
 2-parity  0% |   
     raid 21% | ************  
     hash  7% | ****  
    sched 51% | ******************************  
     misc  0% |   
              |____________________________________________________________  
                            wait time (total, less is better)  
SCRUB - Everything OK  
Saving state to /srv/dev-disk-by-label-DISK1/snapraid.content...  
Saving state to /srv/dev-disk-by-label-DISK2/snapraid.content...  
Saving state to /srv/dev-disk-by-label-DISK3/snapraid.content...  
Saving state to /srv/dev-disk-by-label-DISK4/snapraid.content...  
Verifying /srv/dev-disk-by-label-DISK1/snapraid.content...  
Verifying /srv/dev-disk-by-label-DISK2/snapraid.content...  
Verifying /srv/dev-disk-by-label-DISK3/snapraid.content...  
Verifying /srv/dev-disk-by-label-DISK4/snapraid.content...  
Verified /srv/dev-disk-by-label-DISK4/snapraid.content in 0 seconds  
Verified /srv/dev-disk-by-label-DISK3/snapraid.content in 0 seconds  
Verified /srv/dev-disk-by-label-DISK2/snapraid.content in 0 seconds  
Verified /srv/dev-disk-by-label-DISK1/snapraid.content in 0 seconds

SCRUB finished [Tue 20 Apr 11:43:41 CEST 2021]

----------

## Postprocessing
SnapRAID Smart
SnapRAID SMART report:  
   Temp  Power   Error   FP Size  
      C OnDays   Count        TB  Serial                Device    Disk  
      -      -       -  SSD  0.0  00000000000000000001  /dev/sdb  DATA1  
      -      -       -  SSD  0.0  01000000000000000001  /dev/sdc  DATA2  
      -      -       -    -  0.0  02000000000000000001  /dev/sdd  parity  
      -      -       -  SSD  0.0  03000000000000000001  /dev/sde  2-parity  
      -      -       -  n/a    -  -                     /dev/sr0  -  
      0      -       -    -  0.0  -                     /dev/sda  -  
The FP column is the estimated probability (in percentage) that the disk  
is going to fail in the next year.  
Probability that at least one disk is going to fail in the next year is 0%.

## Restarting Containers [Tue 20 Apr 11:43:41 CEST 2021]

Restarting Container - Code-server
code-server
Restarting Container - Portainer
portainer
All jobs ended. [Tue 20 Apr 11:43:41 CEST 2021]
Email address is set. Sending email report to yourmail@example.com [Tue 20 Apr 11:43:41 CEST 2021]
```

# Requirements

If you are running a Debian based distro (with `apt` package manager) the script will automatically install these dependencies for you.
- [`python3-markdown`](https://packages.debian.org/bullseye/python3-markdown) to format emails
- `curl` to use Healhchecks
- [`jq`](https://packages.debian.org/bullseye/jq) - used to send discord notifications, is a lightweight and flexible command-line JSON processor
- [`bc`](https://packages.debian.org/bullseye/bc) - used for for floating-point comparisons
- [`Apprise`](https://github.com/caronc/apprise) - used to send notifications to 100+ services
   - To install Apprise, the script will use [`pipx`](https://github.com/pypa/pipx). The whole process is managed by the script
   - When Apprise is installed the first time, the script will exit and you'll have to restart it manually. This is needed because of pipx instalation, othwerise Apprise would not be found.


Dependencies that require manual installation:
- `hd-idle` to spin down disks - [Link](https://github.com/adelolmo/hd-idle), installation instructions [below](#installing-hd-idle-for-automatic-disk-spindown)

# Installation and usage

1. Install the packages listed in the Requirements section if you're not running a distro with `apt` package manager
2. Download the latest version from [Releases](https://github.com/auanasgheps/snapraid-aio-script/releases) 
3. Extract the archive wherever you prefer 
   - e.g. `/usr/sbin/snapraid`
4. Give executable rights to the main script 
   - `chmod +x snapraid-aio-script.sh`
5. Open the config file and make changes to the config file as required. 
   - Every config is documented but defaults are pretty resonable, so don't make changes if you're not sure.
   - When you see  `""` or `''` in some options, do not remove these characters but just fill in your data.
   - If you want to spindown your disks, you need to install [hd-idle](https://github.com/adelolmo/hd-idle)
6. Schedule the script execution. 
   - I recommend running the script daily.
  

### Command Line arguments

The script supports two command line arguments to override the default behaviour:
  - Script config file path: provides an alternative script config file path. Add `--config <path>`. Example `snapraid-aio-script.sh --config /home/alternate_config.sh`
  - Override sync protection: execute a forced sync, useful after a warning. Add `--force-sync`. It can be used along with `--config`.

It is tested on OMV6 and OMV7, but will work on other distros. In such case you may have to change the mail binary or SnapRAID location.

### OMV7 USERS
OMV7's SnapRAID plugins introduced support for multiple arrays. This means each SnapRAID config file does not have a predictable name, unlike what occurred with OMV6 or standard SnapRAID installs. 
When running on OMV7, the AIO Script will search for a single SnapRAID configuration file in the new path `/etc/snapraid/`. If multiple arrays are found, it will inform you to adjust your configuration.

## First Run
If you start with empty disks, you cannot use (yet) this script, since it expects SnapRAID files which would not be found.

First run `snapraid sync`. Once completed, the array will be ready to be used with this script.

## OMV and SnapRAID plugin
This script perfectly replaces the OMV built-in script. 
In the OMV GUI, browse to _System > Scheduled Tasks_ and remove/disable the `omv-snapraid-diff` job. 
Also, you can igore all the settings you find at _Services > SnapRAID > Diff Script Settings_, since they only apply to the plugin's built-in script. 

## Installing `hd-idle` for Automatic Disk Spindown
If you would like to enable automatic disk spindown after the script job runs, then you will need to install `hd-idle`. The version included in default Debian and Ubuntu repositories is buggy and out of date - fortunately developer [adelolmo](https://github.com/adelolmo/hd-idle) has improved the project and released an updated version.

**NOTE:** This script is NOT compatible with the `hd-idle` version found in the Debian repositories. You *must* use the updated `hd-idle` binaries for spindown to work. If you receive and error such as `hd-idle cannot spindown scsi disk /dev//dev/sda:` then that is a sign that you are using the old/buggy version. Follow the instructions below to update.

1. Remove any previously existing versions of `hd-idle`, either by manually removing the binaries, or running `apt remove hd-idle` to remove the version from the default respositories.
2. For all recent Ubuntu and Debian releases, install the developers's repository using instructions [on the developer's website](https://adelolmo.github.io/). The command snippet below will select the correct repository based on your current release, and add it to your apt sources.

```
sudo apt-get install apt-transport-https
wget -O - http://adelolmo.github.io/andoni.delolmo@gmail.com.gpg.key | sudo apt-key add -
echo "deb http://adelolmo.github.io/$(lsb_release -cs) $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/adelolmo.github.io.list
```

3. Run `apt update`, and `apt install hd-idle` to install the updated version. You do not need to specify the respository, apt will automatically install the newset version from the new repository.
4. In your `script-config.conf` file, change `SPINDOWN=0` to `SPINDOWN=1` to enable spindown.
5. If you wish to use `hd-idle` as a service to manage your disks outside of the scope of the Snapraid AIO Script, refer to these [additional instructions](https://forum.openmediavault.org/index.php?thread/37438-how-to-spin-down-hard-drives-with-hd-idle/) on the OpenMediaVault forum.

# Upgrade 
If you are using a previous version of the script, do not use your config file. Please move your preferences to the new `script-config.conf` found in the archive. 

# Known Issues
- You tell me!

# Star History

[![Star History Chart](https://api.star-history.com/svg?repos=auanasgheps/snapraid-aio-script&type=Date)](https://www.star-history.com/#auanasgheps/snapraid-aio-script&Date)

# Credits
All rights belong to the respective creators. 
This script would not exist without:
- [Zack Reed](https://zackreed.me/snapraid-split-parity-sync-script/) for most of the original script
- [mtompkins](https://gist.github.com/mtompkins/91cf0b8be36064c237da3f39ff5cc49d) for most of the original script
- [sburke](https://zackreed.me/snapraid-split-parity-sync-script/#comment-300) for the Debian 10 fix
- metagliatore (a friend not on Github) for helping out on several BASH issues
- [ozboss](https://forum.openmediavault.org/wsc/index.php?user/27331-ozboss/)
- [tehniemer](https://github.com/tehniemer)
- [cmcginty](https://github.com/cmcginty)
- [nzlov](https://github.com/nzlov)
- [Caedis](https://github.com/Caedis)
- [Pushpender](https://github.com/ranapushpender)
- [Phidauex](https://github.com/phidauex)

