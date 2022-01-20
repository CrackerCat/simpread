> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [hex-rays.com](https://hex-rays.com/blog/unpacking-mpressed-pe-dlls-with-the-bochs-plugin/)

> In IDA Pro 6.1 we extended the Bochs debugger plugin to support debugging of 64bit code snippets. Wit......

In IDA Pro 6.1 we extended the Bochs debugger plugin to support debugging of 64bit code snippets. With IDA Pro 6.2 it will be possible to debug PE+ executables as well. Since the execution will be emulated inside Bochs, a 64bit operating system is not required and one could be equally running a 32 or 64bit Linux, Mac OS or Windows operating system and still be able to debug 64bit PE files from IDA Pro.  
To illustrate this new feature, we are going to unpack and briefly analyze a PE+ trojan that is compressed with [MPRESS](http://www.matcode.com/mpress.htm) from [MATCODE Software](http://www.matcode.com/).We will illustrate how to unpack the DLL, recover the import table and cleanup the database to get it ready for analysis.  
[![](http://www.hexblog.com/wp-content/uploads/2011/07/bochs_options_thumb.gif)](http://www.hexblog.com/wp-content/uploads/2011/07/bochs_options.gif)  

Unpacking the DLL
-----------------

Our target is a 64bit trojan DLL identified as “Win32/Giku”. We start analyzing this DLL by loading it in _idaq64_ and checking the segment list by pressing Ctrl-S:  
[![](http://www.hexblog.com/wp-content/uploads/2011/07/mpress_thumb.gif)](http://www.hexblog.com/wp-content/uploads/2011/07/mpress.gif)  
Notice how the section names and attributes signal the presence of the MPRESS packer.  
To debug this DLL, make sure that “Bochs debugger plugin” is selected in PE and 64bit emulation mode.  
After starting the debugger, we observe the following code which calls the unpack():  
[![](http://www.hexblog.com/wp-content/uploads/2011/07/bochs_unpacked_thumb.gif)](http://www.hexblog.com/wp-content/uploads/2011/07/bochs_unpacked.gif)  
If we step a bit further, we reach the code that reconstructs the imports by calling LoadLibrary()/GetModuleHandle() in a loop followed by another nested loop that calls GetProcAddress() and stores the result in the IAT:  
[![](http://www.hexblog.com/wp-content/uploads/2011/07/bochs_restore_imports_thumb.gif)](http://www.hexblog.com/wp-content/uploads/2011/07/bochs_restore_imports.gif)  
Noting down the value of the **rdi** register once just before the **stosq** is executed will give us the IAT _start_ address and by noting **rdi** once more at the end of both the loops we get the IAT _end_ address.  
Shortly after the IAT has been restored, we notice the jump to the original entry point:  
[![](http://www.hexblog.com/wp-content/uploads/2011/07/bochs_jump_oep_thumb.gif)](http://www.hexblog.com/wp-content/uploads/2011/07/bochs_jump_oep.gif)  
And the entry point code:  
[![](http://www.hexblog.com/wp-content/uploads/2011/07/nosigs_thumb.gif)](http://www.hexblog.com/wp-content/uploads/2011/07/nosigs.gif)  
This is the real DllEntryPoint() of the unpacked program. Now with the OEP and the IAT start/end values at hand, we are ready to cleanup the database so it represents the unpacked program.

Reconstructing and cleaning the database
----------------------------------------

There are many ways to clean the database from stale / unused information after the program has been unpacked. This involves the following steps:

1.  Locating the IAT and creating an XTRN segment to represent the imports
2.  Deleting the packer’s entry point and adding the original entry point (after unpacking)
3.  Re-analyzing the code
4.  Applying FLIRT signatures
5.  Deleting the unused packer’s segments (optional)

Steps 1 to 3 can be automated using the _uunp_ plugin that ships with IDA. Select from the menu “Edit/Plugins/Universal unpacker manual reconstruct”:  
[![](http://www.hexblog.com/wp-content/uploads/2011/07/uunp_reconstruct_plg_thumb.gif)](http://www.hexblog.com/wp-content/uploads/2011/07/uunp_reconstruct_plg.gif)  
to invoke the _uunp_ plugin directly after reaching the OEP. Fill in all the previously gathered information:  
[![](http://www.hexblog.com/wp-content/uploads/2011/07/uunp_dlg_thumb.gif)](http://www.hexblog.com/wp-content/uploads/2011/07/uunp_dlg.gif)  
and press OK. A new imports segment will be created, the code segment will be reanalyzed and finally a memory snapshot of the unpacked program will be taken.  
We are now ready to apply the appropriate FLIRT signatures:  
[![](http://www.hexblog.com/wp-content/uploads/2011/07/apply_sig_thumb.gif)](http://www.hexblog.com/wp-content/uploads/2011/07/apply_sig.gif)  
After selecting the “vc64rtf” signature from the signatures window (Shift-F5) we notice how IDA identified library functions and colored them with light blue making reverse engineering even easier.

Analyzing the unpacked code
---------------------------

After the code is unpacked, a quick inspection, the _strings window_ reveals a set of encrypted strings:  
[![](http://www.hexblog.com/wp-content/uploads/2011/07/decrypt_str1_thumb.gif)](http://www.hexblog.com/wp-content/uploads/2011/07/decrypt_str1.gif)  
With the help of cross-references, the decryption function was identified. After giving it a proper prototype, we can [Appcall](http://www.hexblog.com/?p=112) it to decrypt the strings:  
[![](http://www.hexblog.com/wp-content/uploads/2011/07/decrypt_str2_thumb.gif)](http://www.hexblog.com/wp-content/uploads/2011/07/decrypt_str2.gif)  
We got a URL pointing to an encrypted text file. After digging a bit in the database, the function used to decrypt files was located:  
[![](http://www.hexblog.com/wp-content/uploads/2011/07/decfileasm_thumb.gif)](http://www.hexblog.com/wp-content/uploads/2011/07/decfileasm.gif)  
And this is the Appcall version of decrypt_file():  
[![](http://www.hexblog.com/wp-content/uploads/2011/07/decfile_thumb.gif)](http://www.hexblog.com/wp-content/uploads/2011/07/decfile.gif)  
We use it to decrypt _spm.txt_:  
[![](http://www.hexblog.com/wp-content/uploads/2011/07/spm_thumb.gif)](http://www.hexblog.com/wp-content/uploads/2011/07/spm.gif)  
x32.jpg is DLL packed with UPX and x64.jpg is a PE+ DLL packed with mpress.  
Hope you found this blog post useful. Comments and suggestion are welcome.