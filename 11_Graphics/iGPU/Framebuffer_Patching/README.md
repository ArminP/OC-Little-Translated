# Intel iGPU Framebuffer patching for connecting an external Monitor (Laptop)

**TABLE of CONTENTS**

- [About](#about)
	- [Problem description](#problem-description)
	- [Possible causes](#possible-causes)
	- [Solution](#solution)
	- [Supported Connection Types](#supported-connection-types)
	- [Workflow overview](#workflow-overview)
- [1. Preparations](#1-preparations)
	- [Create a bootable USB flash drive](#create-a-bootable-usb-flash-drive)
	- [Required Tools and Resources](#required-tools-and-resources)
- [2. Test your current configuration](#2-test-your-current-configuration)
- [3. Verify/adjust the `AAPL-ig-platform-id`](#3-verifyadjust-the-aapl-ig-platform-id)
- [4. Adding Connectors](#4-adding-connectors)
	- [Gather data for your framebuffer (Example 1)](#gather-data-for-your-framebuffer-example-1)
	- [Gather data for your framebuffer (Example 2)](#gather-data-for-your-framebuffer-example-2)
	- [Gathering data for your framebuffer (Example 3)](#gathering-data-for-your-framebuffer-example-3)
	- [Understanding the Parameters](#understanding-the-parameters)
	- [Translating the data into `DeviceProperties` (manually)](#translating-the-data-into-deviceproperties-manually)
- [5. Testing and modifying the framebuffer patch](#5-testing-and-modifying-the-framebuffer-patch)
	- [If case 1 occurs](#if-case-1-occurs)
	- [If case 2 occurs](#if-case-2-occurs)
	- [If case 3 occurs](#if-case-3-occurs)
	- [If case 4 occurs](#if-case-4-occurs)
- [6. Final Steps](#6-final-steps)
- [Credits and further resources](#credits-and-further-resources)

## About
This guide is for modifying framebuffers for Intel iGPUs and modifying `DeviceProperties` for connector types, so you can connect a secondary display to the `HDMI` or `DisplayPort` of your Laptop.

### Problem description
Your Laptop boots into macOS and the internal screen works, but:

1. if you connect a secondary monitor to your Laptop it won't turn on at all or 
2. the handshake between the system and both displays takes a long time and both screens turn off and on several times during the handshake until a stable connection is established.

> **Note**: If you don't get a picture at all you could try a [fake ig-platform-id](https://github.com/5T33Z0/OC-Little-Translated/blob/main/11_Graphics/iGPU/Framebuffer_Patching/Fake_IG-Platform-ID.md) to force the system into [VESA mode](https://wiki.osdev.org/VESA_Video_Modes). For desktop systsems, follow CaseyJ's [General Framebuffer Patching guide](https://www.tonymacx86.com/threads/guide-general-framebuffer-patching-guide-hdmi-black-screen-problem.269149/) instead.

### Possible causes
- Using an incorrect or sub-optimal `AAPL,ig-platform-id` for your iGPU
- Misconfigured framebuffer patch with incorrect BusID and/or flags for the used cable/connector

### Solution
- Modify framebuffer patch for your iGP

### Supported Connection Types
- **HDMI** &rarr; **HDMI**
- **DP** &rarr; **DP** (DisplayPort)
- **HDMI** &rarr; **DVI**
- **DP** &rarr; **DVI**

:warning: **Important**: You cannot use **VGA** or any other analog video signal with modern macOS for that matter. So if this was your plan, you can stop right here!

> **Note**: Although the examples used throughout this guide is for getting the Intel UHD 620 to work, the basic principle is applicable to any other iGPU model supported by macOS as well. Just make sure to use the framebuffer data required for your iGPU.

### Workflow overview
This it the tl;dr version of what we are going to do basically:

1. Copy working EFI to FAT32 formatted USB flash drive
2. Find a suitable Framebuffer
	- Find your iGPU
	- Add recommended `AAPL,ig-platform-id` to `DeviceProperties`
	- Add default Connector Data to config.plist
3. Test if ext. Monitor is working
4. Adjust/modify connector data to optimize handshake
5. Test again
6. Repeat steps 4 and 5 until you think it's optimal
7. Add framebuffer patch to to config.plist on system disk.

## 1. Preparations

### Create a bootable USB flash drive
- Mount your system's EFI partition 
- Copy your EFI folder to a FAT32 formatted USB flash drive
- Unmount the EFI partition again – from now on, We will work with the config stored on the USB flash drive!
- Reboot
- Change the boot order of your drives in the BIOS so that the USB flash drives takes the first slot. Otherwise you have to select the USB flash drive manually after each reboot

**Explanation**: Since we will probably have to do a lot of rebooting to test the framebuffer, we will not work with the config.plist stored on the internal disk because a) we don't want to mess up our working configuration and b) we don't have mount the EFI partition every time to access the config.plist since  it's stored on a flash drive. This really speeds up the workflow.

### Required Tools and Resources
- A FAT32 formatted USB Flash drive
- External Monitor and a cable to connect it to your system (obviously)
- [**Hackintool App**](https://github.com/headkaze/Hackintool/releases) for creating/modifying framebuffer  properties
- [**ProperTree**](https://github.com/corpnewt/ProperTree) to copy over Device Properties to your config
- [**Intel Ark**](https://ark.intel.com/content/www/us/en/ark.html) (for researching CPU specs such as used on-board graphics and device-id)
- [**OpenCore Install Guide**](https://dortania.github.io/OpenCore-Install-Guide/) (for referencing default/recommended framebuffers)
- [**Intel HD Graphics FAQs**](https://github.com/acidanthera/WhateverGreen/blob/master/Manual/FAQ.IntelHD.en.md#intel-uhd-graphics-610-655-coffee-lake-and-comet-lake-processors) for Framebuffers and Device-IDs and additional info.
- [**Big to Little Endian Converter**](https://www.save-editor.com/tools/wse_hex.html) to convert Framebuffers from Big Endian to Little Endian and vice versa
- Monitor cable(s) to populate the output(s) of your mainboard (usually HDMI an/or DisplayPort) 
- Lilu and Whatevergreen kexts enabled (mandatory)

## 2. Test your current configuration
Before we do any editing we will run a basic test. You can skip this if you already know that your external display isn't working. Observe what happens after attaching our external display to your Laptop:

Ext. Display turns On? | Ext. Display detected <br> in Hackintool?|  Action(s)
:---------------------:|:---------------------------------------:|------------------
No | No| <ul> <li> [Verify/adjust used `AAPL,ig-platform-id`](#3-verifyadjust-the-aapl-ig-platform-id) <li> [Add Connectors to framebuffer patch](#adding-connectors) <li> Change Bus ID for both **`con1`** and **`con2`**  
No | Yes (Index?)| Adjust flags for the connector detected at Index X (X= con1 or con2)

If the display is detected in Hackintool's "Patch" section, either **Index 1** or **Index 2** should turn red: 

![display-red](https://github.com/5T33Z0/OC-Little-Translated/assets/76865553/f671908c-ae2f-4241-a782-f35ccaa6c918)

If the display is detected and turns on:

Ext. Display detected <br> in Hackintool?| Handshake| Symptoms | Action
:---------------------------------------:|:--------:|----------|--------
Yes (Index?)| Slow| <ul> <li> Mouse Cursor lagging <li> System feels unresponsive <li> Display turns on late after completing boot sequence <li> Displays turn on and off more than 3 times during handshake|Adjust connector flags for the connector at detected Index. In my case it's connected via **con2**.
Yes (Index?)| Fast| Everything feels normal|Congrats. Nothing to do!

Take note of your test results. Depending on the results you might be able to skip sections.

## 3. Verify/adjust the `AAPL-ig-platform-id`

1. Find the CPU used in your Notebook. If you don't know, you can enter in Terminal: 

	```
	sysctl machdep.cpu.brand_string
	```
2. Search your model number on https://ark.intel.com/ to find its specs. In my case it's an `i5-8265U`.

3. Take note of the the CPU family it belongs to (for example "Whisky Lake") 

4. Identify the iGPU model of the CPU. If the actual model of the iGPU is not specified (i.e. if it says "Intel® UHD Graphics for xth Generation Intel® Processors"), use sites like netbookcheck.net or check in Windows Device Manager to find the exact model. In my case, it's an Intel UHD Graphics 620: <br> ![devmanigp](https://github.com/5T33Z0/OC-Little-Translated/assets/76865553/1844b3a0-01fa-4655-9cf8-ab771388512f)

4. Next, verify that you are using the recommended `AAPL,ig-platform-id` for your CPU and iGPU:
	- Find the recommended framebuffer for your mobile CPU family [in this list](https://github.com/5T33Z0/OC-Little-Translated/blob/main/11_Graphics/iGPU/iGPU_DeviceProperties.md#framebuffers-laptopnuc) (based on data from the OpenCore Install Guide)
	- Check if your iGPU requires a device-id to spoof a different iGPU model!
	- Compare the data with the `DeviceProperties` used in your config.plist

5. Make sure to cross-reference the default/recommended framebuffer for your iGPU against the ones listed in the [Intel HD Graphics FAQs](https://github.com/acidanthera/WhateverGreen/blob/master/Manual/FAQ.IntelHD.en.md#intel-hd-graphics-faqs). But keep in mind that the Intel HD FAQs uses [Big Endian instead of Little Endian](https://github.com/5T33Z0/OC-Little-Translated/blob/main/11_Graphics/iGPU/Framebuffer_Patching/Big-Endian_Little-Endian.md) which is required for the `config.plist`.

6. If necessary, adjust the framebuffer patch (mainly `AAPL,ig-platform-id`) in the config.plist stored on your USB flash drive according to the data you gathered in steps 4 or 5. If you are already using the correct/recommended `AAPL,ig-platform-id`, then you can skip to the next chapter "Adding Connectors".

7. If you have to change the `AAPL,ig-platform-id` but your framebuffer patch already contains connector patches (entries containing `framebuffer-con…`), disable the whole device property entry by placing `#` in front of the dictionary `#PciRoot(0x0)/Pci(0x2,0x0)` and create a new one: `PciRoot(0x0)/Pci(0x2,0x0)` to start with a clean slate. 

8. Add the recommended data you found. If your system requires additional properties so that the internal display works correctly (like backlight register fixes, etc.), copy them over from your previous framebuffer patch.

8. Save your config and reboot from the USB flash drive. If the system boots and the internal display works, we can now add data to connect to an external display. If it doesn't boot, reset the system, boot from the internal disk and start over.

**NOTES**:

- The OpenCore Install Guide only provides *basic* settings to enable your primary display. It does not include additional connectors (except for Ivy Bridge Laptops).
- In my case, the recommended framebuffer and device-id for the Intel UHD 620 differ: Dortania recommends AAPL,ig-platform-id `00009B3E` and device-id `9B3E0000` to spoof the iGPU as Intel UHD 630, while Intel HD FAQs recommends AAPL,ig-platform-id `0900A53E` and device-id `A53E0000` to spoof it as Intel Iris 655 which worked better for me in the end.

## 4. Adding Connectors
Now that we have verified that we are using the recommended framebuffer, we need to gather the connectors data associated with the selected framebuffer in the Intel HD FAQs. Since there are 2 different recommendations for my iGPU, I will look at both.

### Gather data for your framebuffer (Example 1)
The recommended Framebuffer for my Intel UHD suggested by Dortania is AAPL,ig-platform-id `00009B3E`. Now we need to find the connector data for this framebuffer in the Intel HD FAQs:

1. Convert the value for framebuffer to Big Endian. You can use Hackintool's calculator to do this. In my case it's `0x3E9B0000`.
2. Visit the [Intel HD Graphics FAQs](https://github.com/acidanthera/WhateverGreen/blob/master/Manual/FAQ.IntelHD.en.md)
3. Press <kbd>CMD</kbd>+<kbd>F</kbd> to search within the site
4. Enter your framebuffer (converted to Big Endian, without the leading 0x), here: `3E9B0000`
5. You should find your value in a table with other framebuffers
6. Scroll down past the table until you see "Spoiler: … connectors". Click on it to reveal the data
7. Hit forward in the search function to find the next match. This will be the framebuffer data for your AAPL,ig-platform-id!

**Here's the Data for ID 3E9B0000:**

```
ID: 3E9B0000, STOLEN: 57 MB, FBMEM: 0 bytes, VRAM: 1536 MB, Flags: 0x0000130B
TOTAL STOLEN: 58 MB, TOTAL CURSOR: 1 MB (1572864 bytes), MAX STOLEN: 172 MB, MAX OVERALL: 173 MB (181940224 bytes)
Model name: Intel HD Graphics CFL CRB
Camellia: CamelliaDisabled (0), Freq: 0 Hz, FreqMax: 0 Hz
Mobile: 1, PipeCount: 3, PortCount: 3, FBMemoryCount: 3

[0] busId: 0x00, pipe: 8, type: 0x00000002, flags: 0x00000098 - ConnectorLVDS
[1] busId: 0x05, pipe: 9, type: 0x00000400, flags: 0x00000187 - ConnectorDP
[2] busId: 0x04, pipe: 10, type: 0x00000400, flags: 0x00000187 - ConnectorDP

00000800 02000000 98000000
01050900 00040000 87010000
02040A00 00040000 87010000
```

The picture below lists the same data for the 3 connectors this framebuffer provides but with some additional color coding:

![FBADATA02](https://github.com/5T33Z0/OC-Little-Translated/assets/76865553/91a78db5-ec2a-4a1f-8eba-df6720787ed3)

<details>
<summary><strong>More Examples</strong> (click to reveal)</summary>

### Gather data for your framebuffer (Example 2)
Since the Intel HD FAQs recommends a different framebuffer in certain cases you should double-check this, too. So look-up the data for your CPU family in the document and scroll down to where it says ***"Recommended framebuffers"***. In my case the Intel HD FAQs suggests 0x3EA50009 instead.

**Here's the Data for ID: 3E9B0000**
 
```
ID: 3EA50009, STOLEN: 57 MB, FBMEM: 0 bytes, VRAM: 1536 MB, Flags: 0x00830B0A
TOTAL STOLEN: 58 MB, TOTAL CURSOR: 1 MB (1572864 bytes), MAX STOLEN: 172 MB, MAX OVERALL: 173 MB (181940224 bytes)
Model name: Intel HD Graphics CFL CRB
Camellia: CamelliaV3 (3), Freq: 0 Hz, FreqMax: 0 Hz
Mobile: 1, PipeCount: 3, PortCount: 3, FBMemoryCount: 3

[0] busId: 0x00, pipe: 8, type: 0x00000002, flags: 0x00000098 - ConnectorLVDS
[1] busId: 0x05, pipe: 9, type: 0x00000400, flags: 0x000001C7 - ConnectorDP
[2] busId: 0x04, pipe: 10, type: 0x00000400, flags: 0x000001C7 - ConnectorDP

00000800 02000000 98000000
01050900 00040000 C7010000
02040A00 00040000 C7010000
```
### Gathering data for your framebuffer (Example 3)
In the end, I settled with framebuffer `0x3EA50004` instead and in the end you will understand why.

```
ID: 3EA50004, STOLEN: 57 MB, FBMEM: 0 bytes, VRAM: 1536 MB, Flags: 0x00E30B0A
TOTAL STOLEN: 58 MB, TOTAL CURSOR: 1 MB (1572864 bytes), MAX STOLEN: 172 MB, MAX OVERALL: 173 MB (181940224 bytes)
Model name: Intel Iris Plus Graphics 655
Camellia: CamelliaV3 (3), Freq: 0 Hz, FreqMax: 0 Hz
Mobile: 1, PipeCount: 3, PortCount: 3, FBMemoryCount: 3

[0] busId: 0x00, pipe: 8, type: 0x00000002, flags: 0x00000498 - ConnectorLVDS
[1] busId: 0x05, pipe: 9, type: 0x00000400, flags: 0x000003C7 - ConnectorDP
[2] busId: 0x04, pipe: 10, type: 0x00000400, flags: 0x000003C7 - ConnectorDP

00000800 02000000 98040000
01050900 00040000 C7030000
02040A00 00040000 C7030000`
```
</details>

Next, we need to "translate" this data into `DeviceProperties` for our config.plist so we can inject it into macOS. But before we do we should look at what all of these paramaters mean…

### Understanding the Parameters
Let's have a look inside Hackintools "Connectors" tab to get to know the parameters we are working with:

|Parameter|Description|
|:-------:|-----------|
**Index**| An **Index** represents a *physical* graphics output on your Laptop. In macOS, up to 3 software connectors can be assigned (`con0` to `con2`) to 3 connectors (Indexes `1` to `3`). Index `-1` has no physical connector:</br>![Connectors](https://github.com/5T33Z0/OC-Little-Translated/assets/76865553/38ab2a7f-c342-4f1a-81a0-72decf1d0b4d) </br>Framebuffers which only contain `-1` Indexes (often referred to as "headless" or "empty") are used in setups where a discrete GPU is used for displaying graphics while the iGPU performs computational tasks only, such as Platform-ID `0x9BC80003`:</br>![headless](https://github.com/5T33Z0/OC-Little-Translated/assets/76865553/00b0b232-0de7-4a1b-a01f-8c6fabb90753)|
|**BusID**| ![](/Users/stunner/Desktop/BUSIDs.png) </br> The **Bus ID** is used to select the busses or pathways available for connecting displays. Every connector (`con`) must be assigned a *unique* `BusID` through which the signal travels from the iGPU to the physical port(s). Unique means each Bus ID can only be assigned to one connector at a time! And: only certain combinations of BusIDs and connector Types are allowed: <ul> <li> For **DisplayPort**: `0x02`, `0x04`, `0x05` or `0x06`<li>For **HDMI**: `0x01`, `0x02`, `0x04`, `0x06` (availabilty may vary) <li> For **DVI**: same as HDMI <li> For **VGA**: N/A|
**Pipe**| Responsible for taking the graphics data from the iGPU and converting it into a format that can be displayed on a monitor or screen. It performs tasks such as scaling, color correction, and synchronization to ensure that the visual output appears correctly on the connected display. This is fixed for each framebuffer
|**Type**| Type of the *physical* connector (DP, HDMI, DVi, LVDS, etc). Each connector type has a specific value associated with it: <ul><li> **DP**: `00040000` <li> **HDMI**: `00080000` <li> **DVI**: `0080000`
|**Flags**| A bitmask representing connector "Flags" for the selected connector. The recommended value for connect value for any connection is `C7030000`:</br>![Flags](https://github.com/5T33Z0/OC-Little-Translated/assets/76865553/94aa0944-a3dc-4fb8-b68e-ba4c502c7bac) <br> For a complete list of framebuffer and connector flags, [click here](https://github.com/5T33Z0/OC-Little-Translated/blob/main/11_Graphics/iGPU/Framebuffer_Patching/Framebuffer_Connector_Flags.md)

### Translating the data into `DeviceProperties` (manually)

**Recap**: here is the suitable framebuffer patch we found earlier:

```
ID: 3E9B0000, STOLEN: 57 MB, FBMEM: 0 bytes, VRAM: 1536 MB, Flags: 0x0000130B
TOTAL STOLEN: 58 MB, TOTAL CURSOR: 1 MB (1572864 bytes), MAX STOLEN: 172 MB, MAX OVERALL: 173 MB (181940224 bytes)
Model name: Intel HD Graphics CFL CRB
Camellia: CamelliaDisabled (0), Freq: 0 Hz, FreqMax: 0 Hz
Mobile: 1, PipeCount: 3, PortCount: 3, FBMemoryCount: 3

[0] busId: 0x00, pipe: 8, type: 0x00000002, flags: 0x00000098 - ConnectorLVDS
[1] busId: 0x05, pipe: 9, type: 0x00000400, flags: 0x00000187 - ConnectorDP
[2] busId: 0x04, pipe: 10, type: 0x00000400, flags: 0x00000187 - ConnectorDP

00000800 02000000 98000000
01050900 00040000 87010000
02040A00 00040000 87010000
```
No we need to to "translate" this into `DeviceProperties`

**Address**: `PciRoot(0x0)/Pci(0x2,0x0)`

You probably have these entries already – just with the values recommended for your iGPU:

Key | Type | Value| Notes
----|:----:|:----:|------
`AAPL,ig-platform-id`|Data| `00009B3E` |For Laptops with UHD 620
||||
`framebuffer-patch-enable`| Data | `01000000` | Enables Framebuffer patching via Whatevergreen
`framebuffer-stolenmem` | Data | `00003001` | Only needed if "About this Mac" section shows 7mb or less after patching 
`framebuffer-fbmem`| Data | `00009000` | Only needed if "About this Mac" section shows 7mb or less after patching 
||||
`device-id`|Data|`9B3E0000`| Device-ID is required for UHD 620

**And these we need for the Connectors:**

For now, we will leave every value as is, except connector type, since most modern systems use `HDMI` instead of `DisplayPort`.

Key                    | Type | Value| Notes
-----------------------|:----:|:----:|------
`AAPL,slot-name` | String | `Internal@0,2,0`| Internal location of the iGPU (optional). This lists the iGPU in the "Graphics" category of System Profiler
`device-id` | Data | `9B3E0000` | For spoofing Intel UHD 620 as Intel UHD 630. Only add if required for your iGPU model.
`device_type` | String | `VGA compatible controller` | Optional. If you are installing macOS on a legacy system which doesn't have drivers for your iGPU, you should disable it so the display can run in VESA mode. Otherwise it might just turn off.
`disable-external-gpu` |Data| `01000000` | Optional. Only required if your Laptop has an incompatible dGPU
`framebuffer-con1-busid` |Data | `05000000` | BusID used to transmit data to the physical  port # 1 on your machine
`framebuffer-con1-enable` | Data | `01000000` | Enables Patching Connector #2 via Whatevergreen
`framebuffer-con1-flags` | Data | `87010000` | Default flags for connector 3
`framebuffer-con1-index` | Data | `01000000` | Connector 3 has Index 2
`framebuffer-con1-pipe`| Data| `09000000`| Pipe 9
`framebuffer-con1-type` | Data | `00080000` | HDMI. If you have DisplayPort use `00040000` instead
`framebuffer-con2-busid` | Data | `04000000` | BusID used to transmit data to physical port #2 of your machine
`framebuffer-con2-enable` | Data | `01000000` | Enables Patching Connector #3 via Whatevergreen
`framebuffer-con2-flags` | Data| `87010000` | Default flags for connector 3
`framebuffer-con2-index` | Data | `02000000` | Connector 3 has Index 2
`framebuffer-con2-pipe` | Data | `0A000000` | Pipe 10, converted to hex = 0A
`framebuffer-con2-type` | Data | `00080000` | HDMI. If you have DisplayPort use `00040000` instead
`framebuffer-con3-busid` | Data | `00000000` | BusID used to transmit data to physical port #2 of your machine
`framebuffer-con3-enable` | Data | `01000000` | Optional. Enables Patching Connector #4 via Whatevergreen
`framebuffer-con3-index` | Data | `FFFFFFFF` | Optional. Disables the dummy con3
`framebuffer-con3-pipe` | Data | `00000000` | Optional. Selects Pipe 0 which takes con3 offline 
`framebuffer-patch-enable` | Data | `0100000` | Enables Framebuffer patching via Whatevergreen
`model` | String | Intel UHD Graphics 620 | Optional property

> **Note**: We don't add properties for `con0` since this is the internal display which should work fine with the correct framebuffer.

**This is how the DevicyProperties for your iGPU should look like now**:

![cfg-step1](https://github.com/5T33Z0/OC-Little-Translated/assets/76865553/760bd4d0-e2de-493b-8fb0-aee52caf15c0)

## 5. Testing and modifying the framebuffer patch
Now that we have added the default connectors for the selected framebuffer reboot from your USB flash drive and connect your external monitor. 

Observe the behavior of the system:

Case | Ext. Display <br>ON? |Detected in <br> Hackintool | Handshake | Action
:---:|:--------------------:|:--------------------------:|:----------:|---------
1 | **NO** | **NO** | – | Change the BusIDs for **con1** and **con2** and test again
2 | **NO** | **YES** | – | Take note of the Index and BusID and adjust the connector flags and test again
3 | **YES** | **YES** | **Slow** | Modify connector flags and test again
4 | **YES** | **YES** | **Fast** | You're done. Transfer the DeviceProperties to your main config.plist

### If case 1 occurs
Since the external display has not been detected yet, it's most likely that the BusIDs for both connectors is incorrectc. And since we don't know which connector is actually used we need to change the Bus ID for both. 

- Possible BusIDs are: `1`, `2`, `4`, `5`, or `6`
- The same BusID can only be used for one con at a time!
- Listed below you find all possible unique combinations of connectors and BusIDs.
- If you are lucky, you don't need to run all the tests until the monitor is detected and/or turns on!

**Original values**

Key                      | Type | Value
-------------------------|:----:|:----:
`framebuffer-con1-busid` | Data | `05000000`
`framebuffer-con2-busid` | Data | `04000000`

**Next test**

Change the BusIDs for each `con`, save the config.plist and reboot from USB flash drive.

Key                      | Type | Value      | Tested <br> BusIDs
-------------------------|:----:|:----------:|---------------
`framebuffer-con1-busid` | Data | `01000000` | 5 
`framebuffer-con2-busid` | Data | `02000000` | 4 

**Next test**

Change the BusIDs for each `con`, save the config.plist and reboot from USB flash drive.


Key                      | Type | Value      | Tested <br> BusIDs
-------------------------|:----:|:----------:|---------------
`framebuffer-con1-busid` | Data | `02000000` | 5, 1 
`framebuffer-con2-busid` | Data | `01000000` | 4, 2 

**Next test**

Change the BusIDs for each `con`, save the config.plist and reboot from USB flash drive.

Key                      | Type | Value      | Tested <br> BusIDs
-------------------------|:----:|:----------:|---------------
`framebuffer-con1-busid` | Data | `04000000` | 5, 1, 2
`framebuffer-con2-busid` | Data | `06000000` | 4, 2, 1

**Next test**

Change the BusIDs for each `con`, save the config.plist and reboot from USB flash drive.

Key                      | Type | Value      | Tested <br> BusIDs
-------------------------|:----:|:----------:|----------------------
`framebuffer-con1-busid` | Data | `06000000` | 5, 1, 2, 4 
`framebuffer-con2-busid` | Data | `05000000` | 4, 2, 1, 6

If the external display still won't be detected, there must be another issue. Check `AAPL,ig-platform-id` again.

### If case 2 occurs

The testing procedure is the same as for case one. But you only have to test the BusIDs for **con1** or **con2** (depending on which con is detected). In this example, the external display has been detected on **con1**

**Original values**

Key                      | Type | Value
-------------------------|:----:|:----:
`framebuffer-con1-busid` | Data | `05000000`
`framebuffer-con2-busid` | Data | `04000000`

**Next test**

Change the BusIDs for the `con` that your monitor is using, save the config.plist and reboot from USB flash drive.

Key                      | Type | Value      | Tested <br> BusIDs
-------------------------|:----:|:----------:|---------------
`framebuffer-con1-busid` | Data | `01000000` | 5 

**Next test**

Change the BusIDs for the `con` that your monitor is using, save the config.plist and reboot from USB flash drive.

Key                      | Type | Value      | Tested <br> BusIDs
-------------------------|:----:|:----------:|---------------
`framebuffer-con1-busid` | Data | `02000000` | 5, 1,

**Next test**

Change the BusIDs for the `con` that your monitor is using, save the config.plist and reboot from USB flash drive.

Key                      | Type | Value     | Tested <br> BusIDs
-------------------------|:----:|:---------:|---------------
`framebuffer-con1-busid` | Data | `04000000`| 5, 1, 2,
`framebuffer-con2-busid` | Data | `05000000`| Needs to be changed once as well to avoid using the same BusID for both connectors twice (because the default BusID is `4` for **con2**)

**Next test**

Change the BusIDs for the `con` that your monitor is using, save the config.plist and reboot from USB flash drive.


Key                      | Type | Value     | Tested <br> BusIDs
-------------------------|:----:|:---------:|---------------
`framebuffer-con1-busid` | Data | `06000000`| 5, 1, 2, 4

If the external display still won't be detected, there must be another issue. Check `AAPL,ig-platform-id` again.

### If case 3 occurs
Modify the connector flags for the connector(s):

Key                      | Type | Value     | Tested <br> BusIDs
-------------------------|:----:|:---------:|---------------
`framebuffer-con1-flags` | Data| `C7030000` | Works well when using HDMI to HDMI and HDMI to DVI
`framebuffer-con2-flags` | Data| `C7030000` | Works well when using HDMI to HDMI and HDMI to DVI

Safe your config.plist, reboot from USB flash drive and observe if the handshake improves now. When using Adapters like HDMI to DVI the handshake takes a bit longer than using HDMI to HDMI but it should improve nonetheless. 

### If case 4 occurs
Congratulations. Continue with step 6.

## 6. Final Steps

Once you've found a framebuffer patch you are happy with, do the following:

- Open the config.plist containing your framebuffer patch with ProperTree
- Copy the Dictionary `PciRoot(0x0)/Pci(0x2,0x0)` to the clipboard (<kbd>CMD</kbd>+<kbd>c</kbd>)
- Mount your system's EFI 
- Open your config.plist with ProperTree
- Go to `DeviceProperties/Add` 
- Delete Dictionary `PciRoot(0x0)/Pci(0x2,0x0)`
- Press <kbd>CMD</kbd>+<kbd>v</kbd> to paste the new framebuffer patch
- Save and reboot
- In BIOS, change the boot order, so that the internal disk takes the firs slot again.
- Save and exit
- Boot macOS from internal drive
- Done
 
## Credits and further resources
- [General Framebuffer Patching Guide](https://www.tonymacx86.com/threads/guide-general-framebuffer-patching-guide-hdmi-black-screen-problem.269149/) by CaseySJ
- AppleIntelFramebufferAzul.kext ([Part 1](https://pikeralpha.wordpress.com/2013/06/27/appleintelframebufferazul-kext/) and [Part 2](https://pikeralpha.wordpress.com/2013/08/02/appleintelframebufferazul-kext-part-ii/)) by Piker Alpha 
- [Patching connectors](https://dortania.github.io/OpenCore-Post-Install/gpu-patching/intel-patching/connector.html#patching-connector-types) and [Bus IDs](https://dortania.github.io/OpenCore-Post-Install/gpu-patching/intel-patching/busid.html#patching-bus-ids) by Dortania
- benbaker76 for [Hackintool](https://github.com/benbaker76/Hackintool/releases)
- Sepcial tanks to **deeveedee** form insanlymac forums for the help!