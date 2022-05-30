> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [alice.climent-pommeret.red](https://alice.climent-pommeret.red/posts/how-and-why-to-unhook-the-import-address-table/)

> One day, I was trying to bypass an EDR and I noticed something interesting. The EDR I was trying to b......

One day, I was trying to bypass an EDR and I noticed something interesting.

The EDR I was trying to bypass wasn’t hooking the DLL in their code with `jmp` instruction like other EDRs in user-land.

In this case, it was hooking directly the `Import Address Table`. This technique makes the usual move like live-patching, or erasing the loaded DLL with one freshly loaded from disk useless.

I had to unhook the `Import Address Table` of my process.

In this post, I will explain what is an Import Address Table and how to unhook it.

However, some concepts such as `Binding` and `Forwarder` will not be covered. If you want to learn more about those, I highly recommand [this article](https://docs.microsoft.com/en-us/archive/msdn-magazine/2002/march/inside-windows-an-in-depth-look-into-the-win32-portable-executable-file-format-part-2).

The `Import Directory Table` is an array of `IMAGE_IMPORT_DESCRIPTOR`.

![](https://alice.climent-pommeret.red/hooking/import_desc.png)

Structure of IMAGE_IMPORT_DESCRIPTOR

Each `IMAGE_IMPORT_DESCRIPTOR` structure are related to a DLL where functions were imported by the binary.

**Functions of DLL linked dynamically at run time will not be in the `Import Directory Table`.**

> **Run-time dynamic linking** : Usually, when the application calls the `LoadLibrary` or `LoadLibraryEx` to load a DLL and uses `GetProcAddress` to retrieve the functions. You can find more about Run-time dynamic linking [in the Microsoft documentation](https://docs.microsoft.com/en-us/windows/win32/dlls/run-time-dynamic-linking).
> 
> Load-time dynamic linking can also be made using the `Process Environment Block (PEB)` and the `Export Address Table (EAT)`. You can find more about this technique [here](https://alice.climent-pommeret.red/posts/direct-syscalls-hells-halos-syswhispers2/#the-common-ground)

In the case of unhooking, 3 fields is important to us in the `IMAGE_IMPORT_DESCRIPTOR` structures:

*   **Name** : This field contains the DLL name;
*   **OriginalFirstThunk** : Points to the `Import Lookup Table` (also called Import Name Table)
*   **FirstThunk** : Points to `Import Address Table` or `IAT`

Earlier, I said that the `Import Directory Table` is an array of `IMAGE_IMPORT_DESCRIPTOR` and we just saw that each `IMAGE_IMPORT_DESCRIPTOR` have an `Import Address Table` and an `Import Lookup Table`.

This means that there is as many `Import Address Table` that there is `IMAGE_IMPORT_DESCRIPTOR`.

So **THE** `Import Address Table` usually means **ALL** the `Import Address Tables` of a binary !

[](#the-import-address-table-and-the-import-lookup-table)The Import Address Table and the Import Lookup Table
-------------------------------------------------------------------------------------------------------------

The `Import Address Table` and the `Import Lookup Table` are 2 arrays of `_IMAGE_THUNK_DATA_` structures. **Each `_IMAGE_THUNK_DATA_` is related to an imported function**.

![](https://alice.climent-pommeret.red/hooking/image_thunk.png)

Structure of _IMAGE_THUNK_DATA_

We can see that in this structure there is a “UNION”. This means that we can only have one of the 4 elements being set at the same time.

In our case and in this article only `Function` and `AddressOfData` will be used. **In this exercise all the functions of the binary are imported by name**.

When the binary is on the disk, the `_IMAGE_THUNK_DATA_` structures are set with the `AddressOfData` field pointing to `_IMAGE_IMPORT_BY_NAME` structures.

![](https://alice.climent-pommeret.red/hooking/import_by_name.png)

Structure of _IMAGE_IMPORT_BY_NAME

In this structure :

*   **Hint** : It’s the ordinal of the function without the base. The base can be found on the `Export Directory` of the related DLL but it’s not useful for us here.
*   **Name** : The name of the imported function.

If we make an illustration of the relationships between structures it looks like this.

![](https://alice.climent-pommeret.red/hooking/00_Step.png)

Overview of the structures in the importing process

However, on the disk the content of the `Import Address Table` and the `Import Lookup Table` are exactly the same. Both contains an array of `_IMAGE_THUNK_DATA_` pointing to the same `_IMAGE_IMPORT_BY_NAME` structures.

![](https://alice.climent-pommeret.red/hooking/001_Step.png)

IAT and ILT pointing to the same data

When the binary is executed and loaded in memory, the loader parses all the functions names in each `Import Lookup Table`, find the corresponding address in the target DLL and write the address in the right field of the `Import Address Table`.

At this point the `AddressOfData` field of `_IMAGE_THUNK_DATA_` structures in the `Import Address Table` are replaced by the `Function` field.

Once the `Import Address Table` are filled with the proper data, if we parse the `Import Address Table` and the `Import Lookup Table` of an `IMAGE_IMPORT_DESCRIPTOR`, we can see that the first element of the `Import Address Table` is the address of the function pointed by the first `_IMAGE_THUNK_DATA_` element of the `Import Lookup Table`.

If we take the second element of the `Import Address Table` it’s the address of the function pointed by the second `_IMAGE_THUNK_DATA_` element of the `Import Lookup Table` same goes for the third, fourth etc.

![](https://alice.climent-pommeret.red/hooking/02_Step.png)

IAT and ILT loaded

[](#the-hook-and-the-unhooking)The hook and the unhooking
---------------------------------------------------------

From here it’s pretty straightforward. The EDR/AV or malware will replace addresses in the `Import Address Table` to point to its own code. This is the hook.

> If you want to know more about `hooks`, you can find more information [here](https://alice.climent-pommeret.red/posts/direct-syscalls-hells-halos-syswhispers2/#what-is-a-hook-)

Lucky for us, detecting this kind of hook is pretty simple. We need to parse the `Export Address Table` of the DLLs used by our binary to retrieve the `Relative Virtual Address` of our imported functions.

![](https://alice.climent-pommeret.red/hooking/EAT_Parse.png)

Retrieve the Relative Virtual Address of functions

Then, we just have to compute the real addresses using the base address of the target DLLs and the `Relative Virtual Address` of the functions.

> If you want to know more on how to parse the `Export Address Table` and retrieves functions addresses, you can find more information [here](https://alice.climent-pommeret.red/posts/direct-syscalls-hells-halos-syswhispers2/#retrieving-windows-api-functions-addresses-parsing-the-export-address-table-eat)

Finally, we compare the addresses in the `Import Address Table` and the “real” addresses freshly computed.

If some addresses in the `Import Address Table` don’t match our computed addresses this means that those addresses are hooks !

![](https://alice.climent-pommeret.red/hooking/Compare.png)

Comparing address in the IAT and address computed from the EAT

If that’s the case, you just have to replace the hook by the real address of the function and voilà !

Here a little proof-of-concept allowing to unhook the `Import Address table` of a running process:

```
#include <winternl.h>
#include <windows.h>
#include <stdio.h>
#include <stdbool.h>
#include <dbghelp.h>
#pragma comment (lib, "dbghelp.lib")

void UnhookIAT() {

	ULONG size;
	DWORD i, j, x;
	DWORD oldProtect = 0;
	BOOL found = false;
	int sizetab;
	LPVOID TrueRVA;
	
	unsigned char xKernel32[] = { 'k','e','r','n','e','l','3','2','.','d','l','l', 0x0 };
	unsigned char xVirtualProtect[] = { 'V','i','r','t','u','a','l','P','r','o','t','e','c','t', 0x0 };
	typedef BOOL (WINAPI * VirtualProtect_t)(LPVOID, SIZE_T, DWORD, PDWORD);
	VirtualProtect_t VirtualProtect_p = (VirtualProtect_t) GetProcAddress(GetModuleHandle((LPCSTR) xKernel32), (LPCSTR) xVirtualProtect);

	// get Base address of the PE
	HANDLE baseAddress = GetModuleHandle(NULL);		
	
	// get Import Table of PE
	PIMAGE_IMPORT_DESCRIPTOR importTbl = (PIMAGE_IMPORT_DESCRIPTOR) ImageDirectoryEntryToDataEx(
												baseAddress,
												TRUE,
												IMAGE_DIRECTORY_ENTRY_IMPORT,
												&size,
												NULL);

	
	int nbelement = (size/20)-1;
	for (i = 0; i < nbelement ; i++){
		
		//Get name of the DLL in the Import Table
		char * importName = (char *)((PBYTE) baseAddress + importTbl[i].Name);
		printf("DLL name in IAT : %s\n",importName);
		
		//Get Import Lookup Table (OriginalFirstThunk) and Import Address Table (FirstThunk)
		PIMAGE_THUNK_DATA thunk = (PIMAGE_THUNK_DATA) ((PBYTE) baseAddress + importTbl[i].FirstThunk);
		PIMAGE_THUNK_DATA originalFirstThunk = (PIMAGE_THUNK_DATA) ((PBYTE) baseAddress + importTbl[i].OriginalFirstThunk);
		PIMAGE_IMPORT_BY_NAME function = NULL; 
		char* functionName;
		
		//Parse DLL loaded in memory to retrieve various info
		const LPVOID BaseDLLAddr = (LPVOID)GetModuleHandle((LPCSTR)importName);
		PIMAGE_DOS_HEADER pImgDOSHead = (PIMAGE_DOS_HEADER) BaseDLLAddr;
		PIMAGE_NT_HEADERS pImgNTHead = (PIMAGE_NT_HEADERS)((DWORD_PTR) BaseDLLAddr + pImgDOSHead->e_lfanew);
		PIMAGE_EXPORT_DIRECTORY pImgExpDir =(PIMAGE_EXPORT_DIRECTORY)((LPBYTE)BaseDLLAddr+pImgNTHead->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_EXPORT].VirtualAddress);
		PDWORD Address=(PDWORD)((LPBYTE)BaseDLLAddr+pImgExpDir->AddressOfFunctions);
		PDWORD Name=(PDWORD)((LPBYTE)BaseDLLAddr+pImgExpDir->AddressOfNames);
		PWORD Ordinal=(PWORD)((LPBYTE)BaseDLLAddr+pImgExpDir->AddressOfNameOrdinals);

		//loop through all function in the lookup table for the current dll
		while (originalFirstThunk->u1.AddressOfData != NULL){
			
			function = (PIMAGE_IMPORT_BY_NAME)((DWORD_PTR)baseAddress + originalFirstThunk->u1.AddressOfData);
			functionName = function->Name;
			sizetab = 7;
			//Avoid those function or I'll crash
			char* exclude[]= {"EnterCriticalSection","LeaveCriticalSection","DeleteCriticalSection","InitializeSListHead","HeapAlloc","HeapReAlloc","HeapSize"};
			for (x = 0; x < sizetab ; x++){
				if(_stricmp(functionName, exclude[x]) == 0){
					found = true;
				}
			}
			
			if(!found)
			{
				//Get RVA from DLL loaded in memory
				for(j=0;j<pImgExpDir->NumberOfFunctions;j++){
					if(!strcmp(functionName,(char*)BaseDLLAddr+Name[j])){
						TrueRVA = (PVOID)((LPBYTE)Address[Ordinal[j]]);
						break;
					}
				}
		
				//Compute real address
				uintptr_t moduleBase = (uintptr_t)BaseDLLAddr;
				uintptr_t RVA = (uintptr_t)TrueRVA;
				uintptr_t* TrueAddress = (uintptr_t*)(moduleBase + RVA);
				PROC * currentFuncAddr = (PROC *) &thunk->u1.Function;

				if(*currentFuncAddr != (PROC)(TrueAddress)) {
					oldProtect = 0;
					VirtualProtect_p((LPVOID) currentFuncAddr, 8, PAGE_READWRITE, &oldProtect); 
					printf("Bad News ! Function %s is hooked ! Address is %x and it's suppose to be %x \nUnhook like the captain !\n",functionName, *currentFuncAddr, TrueAddress);
					*currentFuncAddr = (PROC)(TrueAddress);
					VirtualProtect_p((LPVOID) currentFuncAddr, 8, oldProtect, &oldProtect);
				}else{
					printf("Good news ! Function %s is not hooked :D\n",functionName);
				}
			}
			++originalFirstThunk;
			++thunk;
			found = false;
		}
	}
}


int main(void) {
   
	UnhookIAT();
	
	return 0;
}
```

You can also find this code on [my GitHub](https://github.com/xalicex/Unhook-Import-Address-Table).

In an attacker point of view you want to remove hooks because you need to get rid of EDR and AV detection.

However in a defensive point of view, you want to detect hooks placed by malware !

If you want to detect if running processes on your system have hooks in their `Import Address Table`, I highly recommand you to use the _excellent_ tool of [@hasherezade](https://twitter.com/hasherezade) [PE-Sieve](https://github.com/hasherezade/pe-sieve).

You can even find a [documentation page](https://github.com/hasherezade/pe-sieve/wiki/4.7.-Scan-for-IAT-Hooks-%28iat%29) just on the IAT hook detection.

Obviously this tool does much more than just detecting hooks in the IAT. If you are interested in detecting malware in memory, inline hooks, Process Hollowing, Process Doppelgänging, Reflective DLL Injection, etc. You should **definitely** take a look at it !

*   [An In-Depth Look into the Win32 Portable Executable File Format, Part 2 by Matt Pietrek](https://docs.microsoft.com/en-us/archive/msdn-magazine/2002/march/inside-windows-an-in-depth-look-into-the-win32-portable-executable-file-format-part-2)
*   [import table vs import address response in stackoverflow (and yes it’s not a joke you’ll see :D)](tablehttps://reverseengineering.stackexchange.com/questions/16870/import-table-vs-import-address-table/29224#29224)
*   [RED TEAM Operator: Malware Development Intermediate Course by Sektor7](https://institute.sektor7.net/rto-maldev-intermediate)