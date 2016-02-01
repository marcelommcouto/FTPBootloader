TFTP secondary bootloader
=========================

This boot loader runs whenever the MCU is reset or powered on.

It first attemps to download a file from a TFTP server. Please
see the file config.h for definitions of the filename and network
addresses.

If the server is down, or the file is not available, the bootloader
checks the existing application code to see if it is valid. If the
application code is valid, the bootloader executes the application.

The bootloader will go into an infinite loop if it cannot download
the file and the existing application is not valid.

The bootloader will go into a restricted loop (3 attempts by default)
if the download succeeded but the application code is invalid, or errors
ocurred part way through the the download, or flash write errors 
ocurred during the boot load process. The idea is that the bootloader
won't wear out the flash memory if the boot load process is just not 
going to work. 
 
Diagnostics
-----------
If the bootloader cannot proceed for any reason, it will provide
some feedback to the user by blinking an LED (PORT0:22 by default).
This only happens when the bootloader detects that the user application
code is invalid.

 1 blink - the download was successful, but the user code was 
 	invalid.
 	
 2 blinks - the request to the TFTP server timed out, either the
 	server is down, or the network is not properly configured
 	
 3 blinks - the TFTP server responded indicating the that the file
 	is not available
 	
 4 blinks - a TFTP error ocurred during the download process
 
 5 blinks - flash write errors ocurred during the download process


[** A lot of the below has been copied almost verbatim from the 
    NXP RDB1768cmsis_usb_bootloader example readme file **]

Using the bootloader
--------------------
Build the bootloader within the IDE and then load it into flash over
the LPCLink debug connection (by selecting the Debug option within 
IDE).


Creating applications to upload via the bootloader
--------------------------------------------------
The TFTP bootloader writes the user application code into flash that is
intended to run from address 0x10000 (64KB) in the memory map. 

Thus applications that you upload via the bootloader need to be built
slightly differently to normal, as they will be executed from a 
non-zero address in the memory map. This is done by modifying the 
scripts used by the linker to control code and data placement.

You will also need to create a plain binary version of your 
application (and add a checksum to it).

Modifying your linker scripts
-----------------------------
The Code Red tools suite by default uses a "managed linker script" 
mechanism to create a script for each Build configuration that is 
suitable for the MCU selected for the project, and the C libraries 
being used.

It will create (and at times modify) three linker script files for
each build configuration of your project:

<projname>_<buildconfig>_lib.ld
<projname>_<buildconfig>_mem.ld
<projname>_<buildconfig>.ld

This set of hierarchical files are used to define the C libraries
being used, the memory map of the system and the way your code and
data is placed in memory. These files will be located in the build
configuration subdirectories of your project (Debug and Release).

When creating applications to be uploaded with the bootloader, you 
need to bypass the managed linker script mechanism and create your
own bespoke linker scripts.

One very important point though is that you are advised not to simply
modify the managed linker scripts in place, but instead to copy them
to another location and modify them there. This will prevent any 
chance of the tools overwritting them at some point in the future.

The following steps detail the simplest way of creating linker 
scripts suitable for creating bootload'able image for the project
"myproj".

1) Create a new subdirectory within your project called, say, 
   "linkscripts".

2) Build the application for debug. This will create the three
   managed link script (.ld) files in the Debug subdirectory of
   your project.

3) Copy the three managed linker script (.ld) files into the 
   linkscripts directory.

4) Modify the filenames of the linker script files in your 
   linkscripts directory to remove the word "Debug". Thus
   "myproj_Debug.ld" would become "myproj.ld", and similarly
   for the other two .ld files.

5) Open the file "myproj.ld". Near the top of this file you will
   see two INCLUDE statements. Modify these to remove the word 
   "Debug" and to insert path information. Thus they will become:
   
   INCLUDE "..linkscripts/myproj_lib.ld "
   INCLUDE "..linkscripts/myproj_mem.ld "
   
   Save the updated "myproj.ld"

6) Now open the file "myproj_mem.ld". You need to change two lines
   in this file, such that the base address used for the image is 
   64KB (0x10000) rather than 0x0, and the the length of the flash
   is 448KB (0x70000) rather than 512KB (0x80000). Thus you will need
   to change the MFlash512 and __top_MFlash512 lines as follows:

   MFlash512 (rx) : ORIGIN = 0x10000, LENGTH = 0x70000

   __top_MFlash512 = 0x10000 + 0x70000;
   
   Save the updated "myproj_mem.ld".
   
Having created your own linker script files, you now need to turn off
the managed linker script mechanism. To do this:

1) Open the Project properties. 
2) In the left-hand list of the Properties window, open "C/C++ Build"
   and select "Settings" and then the "Tool Settings" tab.
3) Now choose "MCU Linker - Target" and untick the Manage linker 
   script box.
4) Now enter the name of the your linker script into the Linker script
   field. This will need to include relative path information, so that
   the linker can find the script (relative to the current build 
   configuration directory (i.e. Debug or Release). For the project
   "myproj", with your linker scripts stored in the "linkscripts"
   subdirectory, the required entry will be:
   
      "../linkscripts/myproj.ld" 
    
   Note that the quote marks are required.

5) Now switch to editing the properties for the Release build
   configuration, and repeat the previous steps to change this to
   use you own linker script instead of the managed link script.

Creating a binary image
-----------------------
By default, the Code Red tools suite creates an ELF image (Executable 
Linkable Format) which is suitable for downloading via the debugger.
However to use the bootloader, you need to convert this into a plain
binary file suitable for the processor to execute directly from 
memory.

To do this automatically each time you carry out a build, you can 
modify you projects post-build steps. To do this:

1) Open the Project properties.
 
2) Select "C/C++ Build" -> "Settings" and switch to the "Build Steps"
   tab.

3) In the "Post-build steps" box, you need to add the following two
   commands:
 
   arm-none-eabi-objcopy -O binary ${BuildArtifactFileName} ${BuildArtifactFileBaseName}.bin
   checksum ${BuildArtifactFileBaseName}.bin;
   
   Note that commands you add to the list need to be separate by 
   semi-colons (";") and that the hash ("#") is comment character
   that will cause the rest of the line to be ignored.

4) Now switch to the Release build configuration and add to the 
   above commands to the post-build steps.
       
[ Some old versions of the Code Red tools may not have the checksum
[ program installed. If you encounter problems then copy the
[ executable contained in the "checksum" subdirectory of the 
[ RDB1768_usb_bootloader project to the "bin" subdirectory at the
[ top level of your Code Red tools installation directory.
  
Now, when you build your project, a .bin file should be created in the
Debug or Release directory which is suitable for downloading via the
tftp bootloader.

If you want to debug your application that is loaded by the tftp 
bootloader and currently running, you can do so by following the
instructions here:
http://support.code-red-tech.com/CodeRedWiki/DebugRunningSystem




