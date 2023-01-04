# RedLineStealer Malware Analysis :

## Content :
	1 - Manual unpacking for the first stage.
	2 - Analysis of the shell code injected.
	3 - Extracting the second Stage.
	4 - List the actual functionalities of the malware.
	5 - Yara rule for detecting the unpacked sample.

## Basic info :
	md5     	FEA0D408C87697BE73C07B988419DC12
	sha1 		EDE45222A0BBD2BECBD21B20897DB5BCC048B991
	sha256  	DD14B18A44EF6AC49EDFE5952D5FD8D5C83FC887D405E97DA15E572ED092B221

The file is 32 bit executable with a not too high ".text" entropy "I admit that packers became more intelligent" 
and the imports and loaded libraries are very small so I believe that it will use run time resolving and loading for APIs and libraries.  
![error](Pics/file.png)

![error](Pics/die.png)

On trying to execute the file to inspect its behavior the file just exits silently so it may detected that it's running inside a VM So I quickly decided to proceed with the advanced analysis state

## Advanced Analysis :

At the start the program concatenates two strings together "C:\\Windows\\Microsoft.NET\\Frame" we don't know what it's doing for now but we will trace that while going on the investigation, but after that, there is a call to "CreateThread" with the address of the function below.

![error](Pics/start.png)

![error](Pics/thread.png)

what's happening here is that the main thread sleeps for three seconds if the other thread managed to finish its execution in these three seconds the main thread will continue the normal flow otherwise it will exit out of the main.
"I think this is an Anti-VM technique because VMs usually don't have much processing power".

If you passed the last check you will get to this block of assembly code.

![error](Pics/next.png)

Before going inside the unknown functions we can really make a good mind map of them just by focusing more on the assembly snippet in front of us, let me explain.
here we have one unknown function that takes two parameters and another concatenation for strings "C:\\Windows\\Microsoft.NET\\Framework\\v4.0.30319\\vbc.exe" (visual basic compiler) and resolves  "VirtualProtect" if we notice the first and second parameters of the unknown function are the same first and second parameters for the "VirtualProtect" API then as we know the parameters for "virtualProtect" we now know that this unknown function is taking a size of memory space in the first argument then doing something and return back a pointer to this memory in the second parameter.

With that known let us start analyzing it.

After looking you will notice that it's just a simple XOR Decryption with a hard coded key.

the pseudo code for this function will be something like this

```C
int __fastcall sub_FF1030(unsigned int memory_size, int memory_address)
{
  unsigned int counter;
  char counter_offset; 
  char XORed_data; 
  int result; 

  for ( counter = 0; counter < memory_size; ++counter )
  {
    counter_offset = *(_BYTE *)(counter + memory_address);
    XORed_data = counter_offset ^ XOR_Key[counter & 3];
    result = sub_FF1006((int)"uA72hxBa");// Doesn't have any effect because the return is never used
    *(_BYTE *)(counter + memory_address) += XORed_data - counter_offset;
  }
  return result; // never used
}

```
this function is called twice then the two allocated memory now have contains this,

the first one contains an executable code 

![error](Pics/code.png)

And the second one contains an executable file

![error](Pics/exe.png)

At the end, it pushes the address of the executable code into the stack to be the return address from the main so it will start executing it.


## Decrypted Code analysis (Shell Code):

The shell code started with a call to this function which I called "PEB_enumeration" to understand what happened here we need to explain a bit about the internals of the PEB structure.

At the offset "0x0C" of the "PEB" structure there is a pointer to another structure called "Ldr" and at offset "0x14" from that structure there is a pointer to a doubly linked list "InMemoryOrderModuleList" that points to every loaded module in the process.

So the malware uses the PEB structure to enumerate the loaded modules and their base address to be able to resolve the needed APIs at run time.

```
xor edx, edx          ; Make sure edx is empty
mov edx, fs:[edx+30h] ; Get the address of PEB
mov edx, [edx+0Ch]    ; Get the address of PEB->Ldr
mov edx, [edx+14h]    ; Get the PEB->Ldr->InMemoryOrderModuleList
```
And Knowing the base address of the module they can look for the API inside it with its hash and that is what happened here.

![error](Pics/resolve.png)

And here are some resolved APIs in this stage

	kernel32_ResumeThreadStub
	kernel32_TerminateProcessStub
	kernel32_VirtualAllocStub
	kernel32_SetThreadContextStub
	kernel32_ReadProcessMemoryStub
	kernel32_GetThreadContextStub
	kernel32_CreateProcessWStub
    kernel32_VirtualProtectExStub
    kernel32_VirtualFreeStub
	kernel32_WriteProcessMemoryStub
	kernel32_VirtualAllocExStub
	kernel32_CloseHandle

	ntdll_ZwUnmapViewOfSection
    ntdll_memcpy
	ntdll_RtlZeroMemory

Then the file will create a "vbc.exe" process in a suspended state inject and inject an executable inside its memory and resume the thread to begin the second stage.

![error](Pics/injected.png)

## Second Stage :

After dumping the injected file out of memory we just need to unmap it manually then we now have the second stage file to proceed with our analysis which is a ".NET" file

![error](Pics/second.png)

The malware starts by checking the region of the device where it runs, if it is in one of the below countries it will just exit without doing anything.

![error](Pics/countries.png)

then the malware will start trying to reach out for its C2 and will keep trying to do that every 5 seconds until it gets a response, that is what the "Id1" of the class "connectionProvider" do.
and here is the C2 Address "89.22.231.25:45245"

![error](Pics/connect.png)

Then configure some connection settings like headers and proxy use then start the actual collection of the data from the device using the "Invoker" method.

![error](Pics/collect.png)

And here are the methods that are being invoked each one is responsible for collecting a sort of data.

![error](Pics/methods.png)

And here is a sample of one of the collecting methods.

![error](Pics/sample.png)

Here is a list of the data that has been collected.

	IPv4 Address
	Domain Name
	Windows Version
	Virtual Display Size
	Country
	User Name
	Processor Info
	Graphics Card Info
	RAM Info
	Browsers Data
	Installed Programs
	Running Processes
	Available Languages
	Telegram data
	Discord tokens
	Steam configuration
	VPN Credentials(OpenVPN, ProtonVPN)
	Antiviruses
	screenshots

## Yara rule :

```Yara
rule redline : infostealer
{
	meta:
		description = "This is a basic rule for detecting unpacked RedLineStealer"
		author = "Amr Ashraf"
		
	strings:
		$mz = {4D 5A}			//MZ header
		
		$string1 = "Discord"
		
		$string2 = "net.tcp://"
		
		$string3 = "get_VirtualScreenWidth"
		
		$string4 = "Syncretise.exe"
		
		$string5 = "ChromeGetRoamingName"
		
		$string6 = "IsNullOrWhiteSpace"
		
		$string7 ="cookies.sqlite"

		$string8 ="installedBrowsers"


	condition:
    	($mz at 0) and (6 of ($string*))
}
```
