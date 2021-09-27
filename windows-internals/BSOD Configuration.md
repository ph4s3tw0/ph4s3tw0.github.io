# Configuring the Windows 10 x64 Blue Screen of Death

Written by hypervis0r

## Introduction

Some years ago, back in 2010, Mark Russinovich [wrote an article](https://techcommunity.microsoft.com/t5/windows-blog-archive/a-bluescreen-by-any-other-color/ba-p/723974) on how to change the Windows Vista Blue Screen of Death. In the article Mark goes over disassembling the `KeBugCheck2` and `KiDisplayBlueScreen` functions in WinDbg, and subsequently editing the memory of the kernel to change the blue screen colors. He later implemented a crash color picker into `NotMyFault`, a tool for crashing Windows, that worked by changing the VGA color palette (as at the time, blue screens were shown in VGA text mode). When Windows 8 rolled around, Microsoft decided that the blue screen should be less frightening, and added a sad emoticon to the crash screen, along with removing most of the crash information in favor of simplifying everything. This new blue screen was implemented using UEFI, specifically using the Graphics Output Protocol (GOP). Due to the new blue screen not using VESA/VGA, the `NotMyFault` color picker no longer worked, and nobody since has tried to replicate it's functionality on the latest versions of Windows, until now that is.

## What even is a "Blue Screen of Death"?

I started my quest to configure the Blue Screen by disassembling `ntoskrnl.exe `, the executive kernel that oversees everything in Windows. Blue Screens appear when the Windows kernel is in an unrecoverable state. This can happen when a "Bug Check" occurs in the system, which happens whenever a bug occurs in the kernel level. Bug Checks are a preventative measure, so that there is no loss of data and the system can halt gracefully. Drivers can call the function `KeBugCheck` to trigger a bug check, and crash the system. Due to `KeBugCheck` being the function that triggers the blue screen, I decided to start reverse engineering there. 

## Disassembling ntoskrnl.exe

After going through some iterations of `KeBugCheck`, I finally find a noteworthy function, `KiDisplayBlueScreen`. In this function, the kernel appears to send debug information over serial, and prepare to display the blue screen. While `KiDisplayBlueScreen` is not directly responsible for displaying the blue screen, it does call a rather interesting function that does, `BgpFwDisplayBugCheckScreen`. In this function, we find out that the screen is cleared and the background is set using `BgpClearScreen`, which takes in a color as a parameter. The color passed to `BgpClearScreen` is found by accessing some structs that I reversed.

![clear_screen](assets\ClearScreen.png)

Well how does `BgpCriticalState` get initialized? Following the xrefs led me to `BgpBcInitializeCriticalMode`, a function that gets called early in the Windows boot phase. This function sets all of the error messages shown in the blue screen, font information, and the key variable we are looking for, the background color.

![color_var](assets\color_var.png)

Noting the hex color format, it appears to be in ARGB format (Alpha, Red, Green, Blue). This is further supported by disassembling `BgpClearScreen`. `BgpClearScreen` works by essentially drawing a rectangle the size of your boot screen resolution and filling the rectangle with the specified color.

 ![fill_rect](assets\fill_rect.png)

As you can see, the most significant byte is set to the alpha, while the RGB colors get set by popping off the most significant byte (removing the A from ARGB). So, if we wanted to draw a red rectangle to the screen, we would use the color value `0xFFFF0000`. 

With our target in mind, I began work on developing a Proof of Concept.

## Writing a kernel level driver for Windows

So to change the color stored at the `BgpCriticalState`, I would need to modify the process memory of `ntoskrnl.exe`.  Luckily, in kernel space, every object shares the same address space, and can read and write each other's memory. That meant I just had to find the address of `BgpCriticalState`, dereference the offset to `BugCheckInfo`, and dereference the offset to the color address. To find the address of `BgpCriticalState`, I couldn't just grab the address from the process memory. Odds would be that address would only last for one hotfix, and I needed something more consistent. 

After doing some reversing with WinDbg, I found the offset between `BgpFwDisplayBugCheckScreen` and `BgpCriticalState`. While this offset could still be inconsistent, it would last longer than other methods. So now, all I needed to do was find `BgpFwDisplayBugCheckScreen` in memory. This proved difficult, as this function is not exported by `ntoskrnl.exe`. To get around this, I utilized a trick known as "signature scanning", where I take the first 100 bytes of the `BgpFwDisplayBugCheckScreen`, and go through each byte of the executive's memory, scanning for that pattern.

Once I found all the offsets I needed, it was just a matter of implementing it in C code.

## Example

So far, this example only works with UEFI (as far as I know). This driver has been tested with Windows 10 x64 20H2.

![RSOD](assets\RSOD.png)

If this doesn't work in a future version or in x86, yeah makes sense. This is very PoC!

Go check it out on the [phasetw0 Github](https://github.com/ph4s3tw0/BSODConfigure) now!

When testing, make sure to to enable testing using `bcdedit /set testsigning on`.

## Remarks

Just a side note for future me (and you reading this), there is a neat function that gets called early in the Windows boot stage called `BgpGxProcessQrCodeBitmap `. This function loads in a bitmap for the QR code into a BgpRect struct, which gets displayed when `BgpFwDisplayBugCheckScreen` gets called.  One might be able to modify it to display any image, which would be pretty neat.

## References

https://techcommunity.microsoft.com/t5/windows-blog-archive/a-bluescreen-by-any-other-color/ba-p/723974

https://techcommunity.microsoft.com/t5/windows-blog-archive/8220-blue-screens-8221-in-designer-colors-with-one-click/ba-p/723977
https://en.wikipedia.org/wiki/Blue_screen_of_death

https://colinxu.wordpress.com/2012/09/03/change-windows-8-bsod-color-and-emoticon/
