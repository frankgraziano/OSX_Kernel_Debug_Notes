#[How-To] Local & Remote Kernel Debugging on OS X 10.11+
Now that GDB is formally deprecated/unsupported for Kernel Debugging in OS X a set of instructions for using LLDB is necessary.

##boot-args 101
###How to check your boot args:
	- $nvram -p

###How to set your boot-args for Thunderbolt Networked version (both “sides” need Thunderbolt Adapters
- Remote computer (Slave/Client/Crashing Machine)
	- $sudo nvram boot-args -v debug=0xd44 kdp_match_name=en4 _panicd_ip=192.168.11.2

###Explanation of Values
	nvram 	 		= Easy way to access the boot-args, otherwise edit the apple boot .plist file
	boot-args 		= Duh
	-v 	            	= Not necessary, turns off the "pretty" apple screen and lets you see startup stuff
	debug=0xd44		= Bitmask values that are better explained by the Internet. Trust me, just use these
	kdp_match_name=en4	= Hack to specify which NIC to use upon crashing, en4 is our thunderbolt adapter, YMMV.
	_panicd_ip		= The IP of our receiver/remote debugger/listener
	_panicd_port		= Don't set this unless you have to. The default is '1069' leave it alone

###How to set your boot-args for Master/Slave Debugging
(VM waits for remote debugger to connect before booting up)
- Remote computer (Slave/Client/Crashing Machine)
	- $sudo nvram boot-args="debug=0x141 kext-dev-mode=1 kcsuffix=development pmuflags=1 -v"

- In order to boot into the development kernel remotely we need to copy the dev kernel to the system folder

$ for 10.11.x you need to disable SIP, boot into Recovery mode (Cmd+R when booting)
	- $csrutil disable
	- $reboot
	- $sudo cp /Library/Developer/KDKs/KDK_10.11.1_15B42.kdk/System/Library/Kernels/kernel.development /System/Library/Kernels/kernel.development 
###Explanation of Values
- nvram 	 	= Easy way to access the boot-args, otherwise edit the apple boot .plist file
- boot-args 		= Duh
- debug=0x141		= Bitmask values various settings. (DB_HALT|DB_ARP|DB_LOG_PI_SCRN)
- kext-dev-mode=1	= Allows you to load unsigned kernel extensions (OPTIONAL)
- kcsuffix=development	= Allows you to specify a debug kernel if you want (=debug) (OPTIONAL)
- pmuflags=1		= Disables the kernel watchdog timer
- -v 	            	= Not necessary, turns off the "pretty" apple screen and lets you see startup stuff

## Setting up your Target Machine:
	- Download the Kernel Debug Kit from Apple Developer website under the OS X download category.
	- Make sure the Kernel Debug Kit you download matches the version of OS X installed on your target machine.
	- “El Capitan”
10.11.0 - http://adcdownload.apple.com/Developer_Tools/Kernel_Debug_Kit_10.11_build_15A284/Kernel_Debug_Kit_10.11_build_15A284.dmg
10.11.1 - http://adcdownload.apple.com/OS_X/Kernel_Debug_Kit_10.11.1_Build_15B42/Kernel_Debug_Kit_10.11.1_Build_15B42.dmg

	- Mount the Kernel Debug Kit disk image on the development machine.
	- For more information on the Kernel Debug Kit, see the Read Me file included in the disk image.

## Setting up your Debug Session:
###From DEBUG Machine (Host/Server/Not Crashing Machine)
- Set the Static IP for the Thunderbolt Network Adapter:
	Static IP: 192.168.11.2
	Static NM: 255.255.255.0
	Static GW: 192.168.11.1

- Test the IP Connectivity
	$ ping 192.168.11.3

- Start the debugger
	$ lldb

- Tell LLDB that we want to work with a remote OSX instance
	$(lldb) platform select remote-macosx


###From TARGET Machine (Slave/Client/Machine that you want to crash)
- Test the IP Connectivity
	$ ping 192.168.11.2

- Trigger the Kernel Panic/Crash on your Target Machine
	$ ./crash_and_burn

- You should see some output on the screen saying "Waiting for Remote Debugger"


###From DEBUG Machine (Host/Server/Not Crashing Machine)
- Initiate the remote debugging session
	$(lldb) kdp-remote 192.168.11.3

- Congratulations now you can debug remotely!!!


## Building the 'debugserver' binary
Note: This is only necessary if you want to connect to the target/test machine with a remote LLDB session.

- Debugserver is a listener that you invoke thusly:
	$ ./debugserver *:4444 -a "ProcessNameYouWantToServeUp"

	*	= means listen for a remote LLDB session from any source IP (you should specify your IP for security)
	4444 	= is the port that you will listen on
	-a 	= puts 'debugserver' into "attach" mode

To utilize this setup you would launch LLDB like this:
	- $lldb
	- $(lldb) platform select remote-osx (or remote-ios or whatever)
	- $(lldb) kdp-remote 192.168.11.3:4444 (IP and Listener port you specified above)

##Install XCODE:
- Download via the App Store

##Install Brew:
$ ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"

##Install Swig: 
$ wget http://prdownloads.sourceforge.net/swig/swig-3.0.5.tar.gz; tar xzvf swig-3.0.5.tar.gz;./configure/make/make install

##Install LLDB:
###Use the Source Luke:

From Terminal:
$ git clone http://llvm.org/llvm/lldb.git
$ cd lldb;open lldb.codeproj (XCode should launch automagically)

From XCode:
- Click "Install" to install additional XCode components
- Click on the yellow triangle in the upper right of the window
- Click "Perform Changes", this will fix the build settings for your version of OS X.
- Click "Disable" for snapshots
- Click "darwin-debug" in the upper menu bar
- Click "debugserver" (selects the build profile that we want)
- Click "My Mac (64-Bit)"
- Click "lldb" in the Left pane to bring up Build Settings
- Click "10.10" for the Deployment Target
- Click "Build Phases" options in the Build Settings workflow
- Delete the last "Run Script" action to delete the codesign script, hack but it was only way to get it to build
- Click the Play button (Or Apple+B) to intiate the Build
- Click "Enable" to enable Developer mode
- Click "OK" after you enter your credentials
- You should hopefully see a notification of "Build Succeeded"
- Click "Tools" on the left pane of LLDB Project
- Click debugserver => debugserver.xcodeproj => Products
- Right-Click on “debugserver”
- Click "Show in Finder"
- Copy this file somewhere so you can use it to launch your remote debugger

You can also Locate the binary directly from Terminal:
- $ cd /Users/<yourname>/Library/Developer/Xcode/DerivedData/lldb-<something_random>/Build/Products/Debug
- Copy this file somewhere so you can use it to launch your remote debugger

##Using ‘debugserver’ to enable remote debugging for your App
http://iphonedevwiki.net/index.php/Debugserver




##Useful Links:

GDB to LLDB Command Reference:
http://lldb.llvm.org/lldb-gdb.html

GDB to LLDB Command Reference:	https://developer.apple.com/library/mac/documentation/IDEs/Conceptual/gdb_to_lldb_transition_guide/document/lldb-command-examples.html

Apple Kernel Debug Guidance:	https://developer.apple.com/library/mac/documentation/Darwin/Conceptual/KernelProgramming/build/build.html

Apple Kernel Dump Guidance:
https://developer.apple.com/library/mac/technotes/tn2004/tn2118.html

Remote Debugging Documentation:
http://lldb.llvm.org/remote.html

Building LLDB on OS X:
http://lldb.llvm.org/build.html#BuildingLldbOnMacOSX

Two Machine Debugging Notes:
http://www.sovapps.com/remote-debugging-on-mac-os-x-with-xcode-6

Some LLDB Debugging Ideas:
https://gist.github.com/steakknife/07df81ffe382d5f257d7

Single Machine Debugging Steps:
http://ddeville.me/2015/08/kernel-debugging-with-lldb-and-vmware-fusion/

KGMacros:
http://ho.ax/posts/2012/02/debugging-the-mac-os-x-kernel-with-vmware-and-gdb/
