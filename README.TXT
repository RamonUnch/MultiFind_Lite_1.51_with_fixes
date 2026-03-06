This software is one of the best search utilities on Windows and I still use it.
Unfortunately it is no longer maintainedand I have no access to source code.
So I had to manually patch everything using disassembly.


# LIST OF CHANGES

2)
FIX: GetWindowPlacement() call where the dev forgot to set .length to 0x2C (44bytes)
ADDRESS: 00470838
:00470838 8B442408        mov eax, dword ptr [esp+08]
:0047083C C7002C000000    mov dword ptr [eax], 0000002C
:00470842 89442408        mov dword ptr [esp+08], eax
:00470846 FF2588134700    Jmp dword ptr [00471388] ; <-- USER32.GetWindowPlacement, Ord:0187h


REFS:
:0041845E FF1588134700    Call dword ptr [00471388] ; -> E8 D5830500 90 // 00 05 83 D5 (OK, no fix actually needed? still replace it for consistency)
:00442BBB FF1588134700    Call dword ptr [00471388] ; -> E8 78DC0200 90 // 00 02 DC 78
:00442CEF FF1588134700    Call dword ptr [00471388] ; -> E8 44DB0200 90 // 00 02 DB 44
:00442DCB 8B3D88134700    mov edi, dword ptr [00471388] ; -> BF3808470090 // mov edi, 00470838
:00442F31 FF1588134700    Call dword ptr [00471388] ; -> E8 02D90200 90 // 00 02 D9 02

replace by -> E8XXXXXXXX ie:
IE: :0040A8DD E8CE820300              call 00442BB0 DIFF = 382CE
00442BB0 - 0040A8DD - 5 = 00 03 82 CE ==> E8 CE 82 03 00


===========================================================================
3) CreatePen forgot to select OUT and to delete !!!!
-- avoid CreatePen alltogether --
So we NOP all the code below and set EAX to 0 with 33C0


------
:004452D6 51                      push ecx
:004452D7 6A01                    push 00000001
:004452D9 6A00                    push 00000000

* Reference To: GDI32.CreatePen, Ord:0049h
                                  |
:004452DB FF1570104700            Call dword ptr [00471070]
:004452E1 50                      push eax
:004452E2 53                      push ebx

* Reference To: GDI32.SelectObject, Ord:025Eh
                                  |
:004452E3 FF1544104700            Call dword ptr [00471044]
-----


-------
:00445319 50                      push eax
:0044531A 6A01                    push 00000001
:0044531C 6A00                    push 00000000

* Reference To: GDI32.CreatePen, Ord:0049h
                                  |
:0044531E FF1570104700            Call dword ptr [00471070]
:00445324 50                      push eax
:00445325 53                      push ebx

* Reference To: GDI32.SelectObject, Ord:025Eh
                                  |
:00445326 FF1544104700            Call dword ptr [00471044]
----


This mean that we get the ugly stock pen 1pixel balck line around the small icons
This could be improved later (maybe thicker pen with double drawing)

===========================================================================
4)

Set some memory to 0 on allocations for safety.
--------
:0043706C 57                      push edi
:0043706D 6A00                    push 00000000 ; <- replace by push 0x0040 GMEM_ZEROINIT = 0x0040
:0043706F 8944241C                mov dword ptr [esp+1C], eax

* Reference To: KERNEL32.GlobalAlloc, Ord:0285h
                                  |
:00437073 FF15FC114700            Call dword ptr [004711FC]




REPLACE FillRect(hdc, &rect, GetSysColorBrush(5)); -> FillRect(hdc, &rect, 5 + 1);
----
>>>>>>:004476D1 6A05                    push 00000005
>>>>>>
>>>>>>* Reference To: USER32.GetSysColorBrush, Ord:016Dh
>>>>>>                                  |
>>>>>>:004476D3 FF154C134700            Call dword ptr [0047134C]
>>>>>>:004476D9 50                      push eax
>>>>>>:004476DA 8D442414                lea eax, dword ptr [esp+14]
>>>>>>:004476DE 50                      push eax
:004476DF 53                      push ebx
:004476E0 FF1554134700            Call dword ptr [00471354] // USER32.FillRect, Ord:00EFh


replace by :
          6A06                    push 06  ; HBRUSH
          50                      push eax ; [rect]
          53                      push ebx ; hDC
:004476E0 FF1554134700            Call dword ptr [00471354] ; USER32.FillRect, Ord:00EFh


===========================================================================
5) some more mem stuff:

:0044F002 8B3DD0114700            mov edi, dword ptr [004711D0] ; <- HERNEL32.HeapAlloc in edi
....
:0044F05D 6A00                    push 00000000 ; <- replace by push 0x08 (6A08) for dwFlags
:0044F05F FF3574494800            push dword ptr [00484974]
:0044F065 FFD7                    call edi ; Calling HeapAlloc


:0044F910 FF15D0114700            Call dword ptr [004711D0] ; <- E8 3B0F0200 90 // call 00470850 (+00 02 0F 3B)

 - special helper to force ZERO_MEMORY flag (8h)
:00470850 834C240808      or dword ptr [esp+08], 00000008
:00470855 FF25D0114700    Jmp dword ptr [004711D0] ; KERNEL32.HeapAlloc, Ord:029Dh
:0047085B 90                      nop


===========================================================================
6)

USE:
6A05                    push 00000005 // WINDOW BACKGROUND
FF154C134700            Call dword ptr [0047134C] ; USER32.GetSysColorBrush, Ord:016Dh
9090906A05FF154C134700

instead of :
:0041396E 68FFFFFF00              push 00FFFFFF
:00413973 FF1534104700            Call dword ptr [00471034] ; GDI32.CreateSolidBrush, Ord:0052h


AND:

:00444F16 68FFFFFF00              push 00FFFFFF
:00444F1B 8BF0                    mov esi, eax
:00444F1D FF1534104700            Call dword ptr [00471034] ; GDI32.CreateSolidBrush, Ord:0052h
  ......
  ......
:004450E4 57                      push edi  <- BRUSH // -> NOP ALL
:004450E5 FF1530104700            Call dword ptr [00471030]; GDI32.DeleteObject, Ord:00D0h -> NOP ALL



===========================================================================
7)

REMOVE LockWindowUpdate() calls replaced them with 90s and 33C0
:00412568 50                      push eax
:00412569 FF15D4134700            Call dword ptr [004713D4]; USER32.LockWindowUpdate, Ord:01E7h

:00412E49 53                      push ebx
:00412E4A FF15D4134700            Call dword ptr [004713D4]; USER32.LockWindowUpdate, Ord:01E7h




===========================================================================
9)

; AVOID GDI ERROR by not selecting the font on the DC when painting the small [...] in the small buttons
; NOP ALL of that !!
:0044748A 6A00                    push 00000000 ; lParam
:0044748C 6A00                    push 00000000 ; wParam
:0044748E 6A31                    push 00000031 ; msg = WM_GETFONT
:00447490 50                      push eax ; hwnd

:00447491 FF155C144700            Call dword ptr [0047145C] ; USER32.SendMessageW, Ord:0263h
:00447497 50                      push eax ; HFONT we got from WM_GETFONT message
:00447498 53                      push ebx ; hDC
:00447499 FF1544104700            Call dword ptr [00471044] ; GDI32.SelectObject, Ord:025Eh



===========================================================================
10) Get rid of more GetSysColorBrush()

--------
:00444F19 6A05                    push 00000005;  <-- NOP
:00444F1B 8BF0                    mov esi, eax

* Reference To: USER32.GetSysColorBrush, Ord:016Dh
                                  |
:00444F1D FF154C134700            Call dword ptr [0047134C]; <-- mov eax, 0x06


--------
:004451B6 6A05                    push 00000005

* Reference To: USER32.GetSysColorBrush, Ord:016Dh
                                  |
:004451B8 FF154C134700            Call dword ptr [0047134C]
:004451BE 50                      push eax
:004451BF 8D4C240C                lea ecx, dword ptr [esp+0C]
:004451C3 51                      push ecx
:004451C4 EB0E                    jmp 004451D4

* Referenced by a (U)nconditional or (C)onditional Jump at Address:
|:004451B4(C)
|
:004451C6 6A0F                    push 0000000F

* Reference To: USER32.GetSysColorBrush, Ord:016Dh
                                  |
:004451C8 FF154C134700            Call dword ptr [0047134C]
:004451CE 50                      push eax
:004451CF 8D54240C                lea edx, dword ptr [esp+0C]
:004451D3 52                      push edx

* Referenced by a (U)nconditional or (C)onditional Jump at Address:
|:004451C4(U)
|
:004451D4 57                      push edi

* Reference To: USER32.FillRect, Ord:00EFh
                                  |
:004451D5 FF1554134700            Call dword ptr [00471354]

--------


:00445279 6A0F                    push 0000000F ; <-- push 0x10

* Reference To: USER32.GetSysColorBrush, Ord:016Dh
                                  |
:0044527B FF154C134700            Call dword ptr [0047134C] ; NOP
:00445281 50                      push eax ; NOP
:00445287 FF1554134700            Call dword ptr [00471354] ; FillRect()

---------
:004456AB 6A0F                    push 0000000F ;  <-- push 0x10
:004456AD 89442414                mov dword ptr [esp+14], eax
* Reference To: USER32.GetSysColorBrush, Ord:016Dh
                                  |
:004456B1 FF154C134700            Call dword ptr [0047134C] ; <- NOP
:004456B7 50                      push eax ; NOP
:004456B8 8D44241C                lea eax, dword ptr [esp+1C]
:004456BC 50                      push eax
:004456BD 57                      push edi

* Reference To: USER32.FillRect, Ord:00EFh
                                  |
:004456BE FF1554134700            Call dword ptr [00471354]

----------
:0044701A 6A0F                    push 0000000F

* Reference To: USER32.GetSysColorBrush, Ord:016Dh
                                  |
:0044701C FF154C134700            Call dword ptr [0047134C]
:00447022 50                      push eax
:00447023 8D44243C                lea eax, dword ptr [esp+3C]
:00447027 50                      push eax
:00447028 53                      push ebx

* Reference To: USER32.FillRect, Ord:00EFh
                                  |
:00447029 FF1554134700            Call dword ptr [00471354]


===========================================================================
11)

a) replace HeapCreate() with GetProcessHeap()
;-------------------------
 00458E5E                           SUB_L00458E5E:    ; this sub is called with 1 as parameter
 00458E5E  8BFF                     	mov	edi,edi
 00458E60  55                       	push	ebp
 00458E61  8BEC                     	mov	ebp,esp
>>> 00458E63  33C0                     	xor	eax,eax
>>> 00458E65  394508                   	cmp	[ebp+08h],eax
>>> 00458E68  6A00                     	push	00000000h ; dwMaxSize
>>> 00458E6A  0F94C0                   	setz 	al ; will be set to 1 if [ebp+08h] == 0 (not the case)
>>> 00458E6D  6800100000               	push	00001000h ; dwInitialSize
>>> 00458E72  50                       	push	eax       ; flOptions
 00458E73  FF1564114700             	call	[KERNEL32.dll!HeapCreate] <- replace
 00458E79  A374494800               	mov	[L00484974],eax
 00458E7E  85C0                     	test	eax,eax
 00458E80  7502                     	jnz	L00458E84
 00458E82  5D                       	pop	ebp
 00458E83  C3                       	retn
;---------------

b) Use Ctrl+T for new tab, Ctrl+W  and Ctrl+F4 to close current tab.
   This is more consistent with other Tabbed softwares.

===========================================================================
12)

Dynamically import SHMultiFileProperties by Ordinal (Win2K+)

file:0002C23E
:0042CE3E  FF157C124700        call    [SHELL32.dll!SHELL32.716]; <- E8 1D3A0400 90 // 43A1D

Also at FILE OFFSET 7DF40 replace CC02 by 9B00 (ILFree)

:00470860 (file:0006FC60)
 00470860                   SUB_L00470860:
 00470860  56               	push   esi
 00470861  53               	push   ebx
 00470862  8B5C240C         	mov    ebx,[esp+0Ch]
 00470866  8B742410         	mov    esi,[esp+10h]
 0047086A  6804024800       	push   L00480204 ; "SHELL32.dll"
 0047086F  FF1524114700     	call   [KERNEL32.dll!GetModuleHandleA]
 00470875  68CC020000       	push   000002CCh ; ORDINAL716 = SHMultiFileProperties
 0047087A  50               	push   eax
 0047087B  FF15CC104700     	call   [KERNEL32.dll!GetProcAddress]
 00470881  85C0             	test   eax,eax
 00470883  740C             	jz    L00470891
 00470885  89742410         	mov    [esp+10h],esi
 00470889  895C240C         	mov    [esp+0Ch],ebx
 0047088D  5B               	pop    ebx
 0047088E  5E               	pop    esi
 0047088F  FFE0             	jmp    eax
 00470891                   L00470891:
 00470891  33C0             	xor    eax,eax
 00470893  40               	inc    eax
 00470894  5B               	pop    ebx
 00470895  5E               	pop    esi
 00470896  C20800           	retn   0008h


: next is :004708A0


===========================================================================
13)

Get rid of LoadLibraryW() and use LoadLibraryA instead
:00437D6B  681C5B4700               	push	SWC00475B1C_UxTheme_dll <- make string ANSI instead.
:00437D70  FF15C8104700             	call	[KERNEL32.dll!LoadLibraryW] ; replace by FF1538124700

--------
Get rid of InitializeCriticalSectionAndSpinCount(&cs, count) and use InitializeCriticalSection instead.
TODO later, add SetSpinCount() in the stub
 004651C2  FF750C                   	push	[ebp+0Ch] ; spin count.
 004651C5  FF7508                   	push	[ebp+08h] ; critical section
 004651C8  FF1504114700             	call	[KERNEL32.dll!InitializeCriticalSectionAndSpinCount]
replace by :  FF7508   FF1524124700  33C0  40  (Because InitializeCriticalSection return VOID and not BOOL)


-----
GetRid of SHGetFolderPathW:
a) re-use InitializeCriticalSectionAndSpinCount entry to GetEnvironmentVariable
b) truncate the unicode string located at 00475A88 to add L"APPDATA" at >> 00475AF0 << some space left afte

 0042E252  50                       	push	eax  ; pszPath[MAX_PATH]
*0042E253  33ED                     	xor	ebp,ebp
 0042E255  55                       	push	ebp  ; dwFlags = 0 (SHGFP_TYPE_CURRENT)
 0042E256  55                       	push	ebp  ; hToken = NULL
 0042E257  6A1A                     	push	0000001Ah ; CSIDL_APPDATA 0x001A
 0042E259  55                       	push	ebp ; hwnd
*0042E25A  8BF1                     	mov	esi,ecx
 0042E25C  FF1574124700             	call	[SHELL32.dll!SHGetFolderPathW] ;  (+4263F) E8 3F260400 90

replace by
*0042E252  33ED                     	xor 	ebp,ebp
*0042E254  8BF1                     	mov 	esi,ecx
 0042E256  6804010000               	push	MAX_PATH=260
 0042E25B  50                       	push	eax
 0042E25C  E8 3F260400              	call	mygetUserdataPath
 0042E261  90                       	nop

c) remove reference to SHGetFolderPathW() <- ShellExecuteW() instead...

d) Use helper below at address :00470890
;------------------------------------------------------------------------
; HRESULT WINAPI mygetUserdataPath(wchar_t *dst, DWORD dst_len)
; PORTABLE MODE: create a folder next to the exe with the same name without .exe...
; Otherwise %APPDATA% will be used...
;------------------------------------------------------------------------
:004708A0                           SUB_L004708A0:
 004708A0  56                       	push	esi
 004708A1  53                       	push	ebx
 004708A2  8B5C240C                 	mov	ebx,[esp+0Ch]
 004708A6  8B742410                 	mov	esi,[esp+10h]
 004708AA  56                       	push	esi
 004708AB  53                       	push	ebx
 004708AC  6A00                     	push	00000000h
 004708AE  FF152C124700             	call	[KERNEL32.dll!GetModuleFileNameW]
 004708B4  83F806                   	cmp	eax,00000006h
 004708B7  7717                     	ja 	L004708D0
 004708B9                           L004708B9:
 004708B9  56                       	push	esi
 004708BA  53                       	push	ebx
 004708BB  68F05A4700               	push	SWC00475AF0_APPDATA
 004708C0  FF1504114700             	call	[KERNEL32.dll!GetEnvironmentVariableW]
 004708C6  85C0                     	test	eax,eax
 004708C8  0F94C0                   	setz 	al
 004708CB  0FB6C0                   	movzx	eax,al
 004708CE  EB12                     	jmp	L004708E2
 004708D0                           L004708D0:
 004708D0  66836443F800             	and	word ptr [ebx+eax*2-08h],0000h
 004708D6  53                       	push	ebx
 004708D7  FF1594104700             	call	[KERNEL32.dll!GetFileAttributesW]
 004708DD  40                       	inc	eax
 004708DE  74D9                     	jz 	L004708B9
 004708E0  33C0                     	xor	eax,eax
 004708E2                           L004708E2:
 004708E2  5B                       	pop	ebx
 004708E3  5E                       	pop	esi
 004708E4  C20800                   	retn	0008h
;------------------------------------------------------------------------


==========================================================================
14)
;  Name: .rdata (Data Section) ; --------------------------- (60 extra bytes)
;  Virtual Address:    00471000h  Virtual Size:    0000F7C2h <-- change to 0000F800h
;  Pointer To RawData: 0006FE00h  Size Of RawData: 0000F800h


; get rid of InterlockedExchange with lock xchg [], eax
:0044DEF8  6A00                      		push	00000000h <-  33C0      xor eax, eax
 0044DEFA  53                        		push	ebx       <-  F0 87 03  lock xchg [ebx], eax
 0044DEFB  FF15F4114700              		call	[KERNEL32.dll!InterlockedExchange] ; NOP
; replace by: 33C0 F08703 90909090
xor eax,eax;
lock xchg


; get rid of InterlockedCompareExchange with lock cmpxchg [ecx], esi
:0044DEB1  57                       	push	edi       ; compare
 0044DEB2  56                       	push	esi       ; exchange
 0044DEB3  FF75FC                   	push	[ebp-04h] ; dest
 0044DEB6  893E                     	mov	[esi],edi
 0044DEB8  FF15F8114700             	call	[KERNEL32.dll!InterlockedCompareExchange]
; replace by:
0:  89 F8                   mov    eax,edi                    ; compare in eax
2:  8B 4D FC                mov    ecx,DWORD PTR [ebp-0x4]    ; dst in ecx
5:  F0 0F B1 39             lock cmpxchg DWORD PTR [ecx], edi ; puts edi in [ecx] if eax!=[ecx]
6:  9090 9090               nops


==========================================================================
15)
a) Remove GetModuleHandleW() stupid loop more CC space for future...
:00458E8E                           SUB_L00458E8E: ; LOPP until GetModuleHandleW() succeeds
 00458E8E  8BFF                      		mov	edi,edi
 00458E90  55                        		push	ebp
 00458E91  8BEC                      		mov	ebp,esp
 00458E93  57                        		push	edi
 00458E94  BFE8030000                		mov	edi,000003E8h
 00458E99                           L00458E99:
 00458E99  57                        		push	edi
 00458E9A  FF15F0114700              		call	[KERNEL32.dll!Sleep]
 00458EA0  FF7508                    		push	[ebp+08h]
 00458EA3  FF15D4104700              		call	[KERNEL32.dll!GetModuleHandleW]
 00458EA9  81C7E8030000              		add	edi,000003E8h
 00458EAF  81FF60EA0000              		cmp	edi,0000EA60h
 00458EB5  7704                      		ja 	L00458EBB
 00458EB7  85C0                      		test	eax,eax
 00458EB9  74DE                      		jz 	L00458E99
 00458EBB                           L00458EBB:
 00458EBB  5F                        		pop	edi
 00458EBC  5D                        		pop	ebp
 00458EBD  C3                        		retn

replace by a simple return -1; => 8BFF 33C0 48 C3 and CC CC CC CC ...
----------
b) Prefer GetModuleHandleW for KERNEL32.DLL frees KERNEL32 ANSI string at 004735D4 (unification)

:00460665  68D4354700                		push	SSZ004735D4_KERNEL32 ; replace by 68 B02D4700 L"KERNEL32.DLL" (unicode)
 0046066A  FF1524114700              		call	[KERNEL32.dll!GetModuleHandleA] ; <- FF15D4104700

---------
c) get rid of the final GetSysColorBrush()

:00413971  6A05                      		push	00000005h
 00413973  FF154C134700              		call	[USER32.dll!GetSysColorBrush]
 00413979  898670030000              		mov	[esi+00000370h],eax

replace BY b8 06000000          mov    eax,0x6


===========================================================================
16)

a)
; NOP OUT ALL Font bold stuff (GDI handle leak)
:00416DBD  53                        		push	ebx ; 0
 00416DBE  53                        		push	ebx ; 0
 00416DBF  8B1D20144700              		mov	ebx,[USER32.dll!GetDlgItem]
 00416DC5  6A31                      		push	00000031h
 00416DC7  683D040000                		push	0000043Dh
 00416DCC  57                        		push	edi
 00416DCD  FFD3                      		call	ebx ; GetDlgItem(edi, 0000043D)
 00416DCF  8B355C144700              		mov	esi,[USER32.dll!SendMessageW]
 00416DD5  50                        		push	eax
 00416DD6  FFD6                      		call	esi ; SendMessageW(GetDlgItem(edi, 0000043D), 31, 0, 0)
 00416DD8  8D8C24AC000000            		lea	ecx,[esp+000000ACh]
 00416DDF  51                        		push	ecx
 00416DE0  6A5C                      		push	0000005Ch
 00416DE2  50                        		push	eax
 00416DE3  FF1540104700              		call	[GDI32.dll!GetObjectW]
 00416DE9  8D9424AC000000            		lea	edx,[esp+000000ACh]
 00416DF0  52                        		push	edx
 00416DF1  C78424C0000000BC0200+     		mov	dword ptr [esp+000000C0h],000002BCh ; make bold...
 00416DFC  FF153C104700              		call	[GDI32.dll!CreateFontIndirectW] ; /!\ never freed TT /!\
 00416E02  6A01                      		push	00000001h
 00416E04  50                        		push	eax
 00416E05  6A30                      		push	00000030h
 00416E07  683D040000                		push	0000043Dh
 00416E0C  57                        		push	edi
 00416E0D  FFD3                      		call	ebx ; GetDlgItem(edi, 0000043Dh);
 00416E0F  50                        		push	eax
 00416E10  FFD6                      		call	esi ; SendMessageW(GetDlgItem(edi), 30, eax=font );

------------
b) More GDI handle leaks fixed because of bad bitmap handling.
; below code is broken because LR_SHARED does not work on non standard bitmap sizes
; and the bitmap resource 151 is not one of the usual icon size.
 0042B30D  6840800000                		push	00008040h ; LR_DEFAULTSIZE | LR_SHARED
 0042B312  55                        		push	ebp ; cyDesired = 0
 0042B313  55                        		push	ebp ; cxDesired = 0
 0042B314  55                        		push	ebp ; uType = 0 (IMAGE_BITMAP)
 0042B315  6897000000                		push	00000097h ; bitmap resource 151
 0042B31A  50                        		push	eax ; hinst
 0042B31B  FF15FC124700              		call	[USER32.dll!LoadImageW] ; bmp
 0042B321  8944240C                  		mov	[esp+0Ch],eax
 0042B325  8B442418                  		mov	eax,[esp+18h]
 0042B329  2B442410                  		sub	eax,[esp+10h]
 0042B32D  33FF                      		xor	edi,edi
 0042B32F  85C0                      		test	eax,eax
 0042B331  7E29                      		jle	L0042B35C
 0042B333                           L0042B333:
 0042B333  8B4C240C                  		mov	ecx,[esp+0Ch]
 0042B337  6A04                      		push	00000004h; fuFlags = 4
 0042B339  6A32                      		push	00000032h; cx 32
 0042B33B  6A01                      		push	00000001h; cy = 1
 0042B33D  55                        		push	ebp ; x = 0
 0042B33E  57                        		push	edi ; y = 0
 0042B33F  55                        		push	ebp ; wData = NULL
 0042B340  51                        		push	ecx ; lData = hBMP
 0042B341  55                        		push	ebp ; lpOutputFunc = NULL
 0042B342  55                        		push	ebp ; hBrush = NULL
 0042B343  56                        		push	esi ; hDC
 0042B344  FF15F8124700              		call	[USER32.dll!DrawStateW]

so we nop out most of it and replace it with: 6A02 FF1584104700 C744241C32000000 8D542410 50 52 56 FF1554134700
	push   0x3
	call   DWORD PTR [GDI32.GetStockObject]
	mov    DWORD PTR [esp+0x1c],0x32
	lea    edx,[esp+0x10] ; &rc address
	push   eax ; GetStockObject(GRAY_BRUSH = 2)
	push   edx
	push   esi
	call   DWORD PTR [GDI32.FillRect] ; FillRect(esi=hDc, edx=&{0,0,client.left, 50}, GetStockObject(GRAY_BRUSH));
and a lot of NOPS...

Also delete bitmap resource 151 with resource editor
with that we have a simple gray brush as a background rather than a gradient made from a bitmap.
----

===========================================================================
17) remove creation on intermediate bmp for tab display.

; 0044567F  55                        		push	ebp ; ori_hDC
; 00445680  FF157C104700              		call	[GDI32.dll!CreateCompatibleDC]
; 00445686  8B4C2420                  		mov	ecx,[esp+20h] ; crc.right
; 0044568A  2B4C2418                  		sub	ecx,[esp+18h] ; ecx = crc.right - crc.left
; 0044568E  8BF8                      		mov	edi,eax ; compat_hDC
; 00445690  8B442424                  		mov	eax,[esp+24h] ; crc.bottom
; 00445694  2B44241C                  		sub	eax,[esp+1Ch] ; eax = crc.bottom - crc.top
; 00445698  50                        		push	eax ; height = crc.bottom - crc.top
; 00445699  51                        		push	ecx ; width = crc.right - crc.left
; 0044569A  55                        		push	ebp ; ori_hDC
; 0044569B  FF1578104700              		call	[GDI32.dll!CreateCompatibleBitmap]
; 004456A1  8BD8                      		mov	ebx,eax
; 004456A3  53                        		push	ebx ; hBMP
; 004456A4  57                        		push	edi ; compat_hDC
; 004456A5  FF1544104700              		call	[GDI32.dll!SelectObject] (
; 004456AB  6A10                      		push	00000010h ; brush 0F + 1 (for FillRect)
; 004456AD  89442414                  		mov	[esp+14h],eax ; old_hBMP
; 004456B1  90                        		nop
; 004456B2  90                        		nop
; 004456B3  90                        		nop
; 004456B4  90                        		nop
; 004456B5  90                        		nop
; 004456B6  90                        		nop
; 004456B7  90                        		nop
; 004456B8  8D44241C                  		lea	eax,[esp+1Ch]
; 004456BC  50                        		push	eax ; &rect
; 004456BD  57                        		push	edi ; compat_hDC
; 004456BE  FF1554134700              		call	[USER32.dll!FillRect] ; (compat_hDC, &rc, 0x10)
; 004456C4  8B4C2478                  		mov	ecx,[esp+78h]
; 004456C8  51                        		push	ecx       ;
; 004456C9  57                        		push	edi       ; compat_DC
; 004456CA  6A0F                      		push	0000000Fh ; 1
; 004456CC  8BCE                      		mov	ecx,esi
; 004456CE  E82DDCFFFF                		call	SUB_L00443300
; 004456D3  8B44241C                  		mov	eax,[esp+1Ch]
; 004456D7  8B542424                  		mov	edx,[esp+24h]
; 004456DB  8B4C2418                  		mov	ecx,[esp+18h]
; 004456DF  682000CC00                		push	00CC0020h ; Raster operation = SRCCOPY
; 004456E4  6A00                      		push	00000000h ; nYSrc
; 004456E6  6A00                      		push	00000000h ; nXSrc
; 004456E8  57                        		push	edi ; src DC = compat_hDC
; 004456E9  2BD0                      		sub	edx,eax
; 004456EB  52                        		push	edx ; nHeight
; 004456EC  8B542434                  		mov	edx,[esp+34h]
; 004456F0  2BD1                      		sub	edx,ecx
; 004456F2  52                        		push	edx ; nWidth
; 004456F3  50                        		push	eax ; nYdest
; 004456F4  51                        		push	ecx ; nXdest
; 004456F5  55                        		push	ebp ; dest DC = ori_hDC
; 004456F6  FF1574104700              		call	[GDI32.dll!BitBlt]
; 004456FC  8B442410                  		mov	eax,[esp+10h] ; old_hBMP
; 00445700  50                        		push	eax ; old_hBMP
; 00445701  57                        		push	edi ; compat_hDC
; 00445702  FF1544104700              		call	[GDI32.dll!SelectObject] ; (compat_hdc, old_BMP)
; 00445708  53                        		push	ebx ; hBMP
; 00445709  FF1530104700              		call	[GDI32.dll!DeleteObject] ; (hBMP)
; 0044570F  57                        		push	edi ; compat_hDC
; 00445710  FF1560104700              		call	[GDI32.dll!DeleteDC] ; (compat_hDC);

replace by; 90 8D4C2418 6A10 51 55 FF1554134700 8B4C2478 51 55 6A0 F8BCE E863DCFFFF EB77 CCCCCC....

===========================================================================
18) Fix some leaks.

; stop using underlined link global_font (nice but kinda useless).
; this can be used later
> 00437B9D                           L00437B9D:
> 00437B9D  A1704E4800                		mov	eax,[L00484E70]
> 00437BA2  6A00                      		push	00000000h
> 00437BA4  50                        		push	eax
> 00437BA5  6A30                      		push	00000030h
> 00437BA7  57                        		push	edi
  00437BA8  C7460801000000            		mov	dword ptr [esi+08h],00000001h
> 00437BAF  FF155C144700              		call	[USER32.dll!SendMessageW]
> 00437BB5  6A00                      		push	00000000h
> 00437BB7  6A00                      		push	00000000h
> 00437BB9  57                        		push	edi
> 00437BBA  FF15E0124700              		call	[USER32.dll!InvalidateRect]
only do
C7460801000000            		mov	dword ptr [esi+08h],00000001h
the rest is filled with CC


remove deletion of the global font (called when closing help dialog)
// NOP All that...
> 00437A9C  8B15704E4800              		mov	edx,[L00484E70]
> 00437AA2  52                        		push	edx
:00437AA3  C705644C480000000000      		mov	dword ptr [L00484C64],00000000h ; Zero out global cursor
> 00437AAD  FF1530104700              		call	[GDI32.dll!DeleteObject] ; delete global hfont
> 00437AB3  C705704E480000000000      		mov	dword ptr [L00484E70],00000000h ; Zero out global font

; do not create this font in the first place:
>>>> 004376C7  52                        		push	edx ; nop out
 004376C8  C644241D01                		mov	byte ptr [esp+1Dh],01h ; Underscore
>>>> 004376CD  FF153C104700              		call	[GDI32.dll!CreateFontIndirectW] ; replace by 33C0 + NOPS
 004376D3  8B3508144700              		mov	esi,[USER32.dll!LoadCursorW]
 004376D9  68897F0000                		push	00007F89h
 004376DE  6A00                      		push	00000000h
>>>>004376E0  A3704E4800                		mov	[L00484E70],eax ; store hFont; NOP OUT

; he forgot to release the DC just to get its dpi
 00413A12  6A5A                      		push	0000005Ah  ; LOGPIXELSY = 90
 00413A14  50                        		push	eax ; hwnd
 00413A15  FF15C8134700              		call	[USER32.dll!GetWindowDC] ; (hwnd)
 00413A1B  50                        		push	eax ; hDC
 00413A1C  FF1538104700              		call	[GDI32.dll!GetDeviceCaps]; (hDC, LOGPIXELSY);

replace by a call to file offset: 00044AA0 - 00012E15 => E8 861C0300 // 00031C86

helper function GetDeviceCapsForHwnd(hwnd, id) { dc = GetDC(hwnd); cap = GetDeviceCaps(dc, id); ReleseDC(hwnd, dc); return cap; }
file: 00044AA0:
57 56 53 8B742410 56 FF15D8124700 93 FF742414 53 FF1538104700 53 97 56 FF1550134700 97 5B 5E 5F C20800

; Now we can cache the Courier New font creation at the [00484E70] address. => Ax 70 4E 48 00
00413A3C  FF153C104700              call	[GDI32.dll!CreateFontIndirectW] ; <- replace by E8 8F1C0300 90
file 12E3C
0003108F => E8 8F100300 90
helper CreateFontIndirectOnceW() created at file:00044AD0
A1704E4800 85C0 750F FF742404 FF153C104700 A3704E4800 C20400
        mov     eax, DWORD PTR [00484E70] ; _cached_font
        test    eax, eax
    *   jne     END
        push    DWORD PTR [esp+4]
        call    [GDI32.dll!CreateFontIndirectW]
        mov     DWORD PTR [00484E70], eax ; _cached_font
    END:
        ret     4


===========================================================================
19) More resource usage optimisations.

a) Make a LoadIconImageOnce() at Address 004708E8
replace calls:
:00444BC9  FF15FC124700              call	[USER32.dll!LoadImageW] ; (2BD1A) => E8 1A BD 02 00 90
:00444C1C  FF15FC124700              call	[USER32.dll!LoadImageW] ; (2BCC7) => E8 C7 BC 02 00 90
:0044646B  FF15FC124700              call	[USER32.dll!LoadImageW] ; (2A478) => E8 78 A4 02 00 90
:00447571  FF15FC124700              call	[USER32.dll!LoadImageW] ; (29372) => E8 72 93 02 00 90


LoadIconImageOnce() at Address 004708E8
:004708E8                           SUB_L004708E8: (LoadIconImageOnce)
 004708E8  53                        		push	ebx
 004708E9  8B54240C                  		mov	edx,[esp+0Ch] ; Icon resource starts at 100.
 004708ED  8D5A9C                    		lea	ebx,[edx-64h]
 004708F0  8B049D5C644800            		mov	eax,[L0048645C+ebx*4]
 004708F7  85C0                      		test	eax,eax
 004708F9  7522                      		jnz	L0047091D
 004708FB  FF74241C                  		push	[esp+1Ch]
 004708FF  FF74241C                  		push	[esp+1Ch]
 00470903  FF74241C                  		push	[esp+1Ch]
 00470907  FF74241C                  		push	[esp+1Ch]
 0047090B  52                        		push	edx
 0047090C  FF74241C                  		push	[esp+1Ch]
 00470910  FF15FC124700              		call	[USER32.dll!LoadImageW]
 00470916  89049D5C644800            		mov	[L0048645C+ebx*4],eax ; save result in the cache.
 0047091D                           L0047091D:
 0047091D  5B                        		pop	ebx
 0047091E  C21800                    		retn	0018h ; DONE!

uses [0048645C - 004864FC] => Up to 40 Icons (some margin....)
=> Increase .data section virtual size up to 004864FC...


b) Also replace the final occurence of LoadImage by LoadIconW (for the option/help Icons)
; 0042B36B  6840800000                		push	00008040h ; LR_DEFAULTSIZE | LR_SHARED
; 0042B370  55                        		push	ebp ; 0
; 0042B371  55                        		push	ebp ; 0
; 0042B372  6A01                      		push	00000001h ; uType = IMAGE_ICON
 0042B374  51                        		push	ecx ;
 0042B375  50                        		push	eax ; hinst
; 0042B376  FF15FC124700              		call	[USER32.dll!LoadImageW] ; <- replace by LoadIconW (FF150C144700)


c) Replace DrawStateW by DrawIcon (for the option/help Icons)
; 0042B37C  6A03                      		push	00000003h ; DST_ICON
; 0042B37E  55                        		push	ebp       ; cy = 0
; 0042B37F  55                        		push	ebp       ; cx = 0
; 0042B380  6A0A                      		push	0000000Ah ; y
; 0042B382  6A0A                      		push	0000000Ah ; x
; 0042B384  55                        		push	ebp ; wData = 0
; 0042B385  50                        		push	eax ; lData = hIcon
; 0042B386  55                        		push	ebp ; lpOutputFunc = NULL
; 0042B387  55                        		push	ebp ; hBrush = NULL
; 0042B388  56                        		push	esi ; hDC
; 0042B389  FF15F8124700              		call	[USER32.dll!DrawStateW]; Replace by 50 6A0A 6A0A  56 CALL DrawIcon (FF1518134700)

===========================================================================
20) Reduce color count for some icons and the cursor (reduce exe size)
--> RELEASE 1.511 <--


TODO:
Dead code to investigate?
1) SUB_L00439400 ??
2) SUB_L00439250 ??

Fix ToolTip memort leak by re-using the window handlse (a table could be created).
Fix incomplete closing of a tab (some objects are not destroyed properly).
Fix the dev forgot to call LocalFree on the CommandLineToArgvW result (minor leak but ugly)


USEFULL:
[00484974] is a global Heap Handle
[00484C64] is a global CURSOR
[00484E70] is a global HFONT <- Used as a cache for Courier New font
there is still a lot of potential code to be reused. if needed
you can replace some SSE2 memcpy/memset functions by rep movsd and rep stosd


;; UXTHEME.DLL (loaded in SUB_L00437D20) ;;
[00484E78] is a handle to UxTheme.dll
[00484E7C] UxTheme.OpenThemeData()
[00484E80] UxTheme.CloseThemeData()
[00484E84] UxTheme.DrawThemeBackground()
[00484E88] UxTheme.GetThemeBackgroundContentRect()
[00484E8C] UxTheme.IsThemeActive()
[00484E90] UxTheme.DrawThemeParentBackground()
[00484E94] UxTheme.IsThemeBackgroundPartiallyTransparent()
; called in SUB_L00438260 and SUB_L00438110



-- ANSI STRINGS from import table:
  0047F3EA  COMCTL32.dll
  0047F452  SHLWAPI.dll ; <-- (to get rid of)
  0047F7C0  KERNEL32.dll
  0047FF40  USER32.DLL
  004800AC  GDI32.dll
  0048010C  ADVAPI32.dll
  00480204  SHELL32.dll
  00480296  ole32.dll

-- UNICODE Dlls:
B02D4700 L"KERNEL32.DLL" (unicode)


NT4 NOT AVAILABLE:
KERNEL32.GetLongPathNameW
KERNEL32.ReplaceFileW


// SHELL32.ORDINAL716 is exported by ordinal only in Win2K
;; SHELL32.ORDINAL716 (ord:02CCh) -> SHELL32.SHMultiFileProperties, Ord:02CCh (dynamicaly imported)
;;;; SHELL32.SHGetFolderPathW (ord:00C0h) -> ShellExecuteW() second occurence

SHLWAPI.DLL (NOT PRESENT)





FREE IMPORTS in the table (could be reused...):
KERNEL32.HeapCreate()                  < UNUSED
KERNEL32.LoadLibraryW()                < UNUSED
KERNEL32.InterlockedExchange()         < UNUSED
KERNEL32.InterlockedCompareExchange()  < UNUSED
USER32.LockWindowUpdate()              < UNUSED
GDI32.CreatePen()                      < UNUSED
GDI32.CreateSolidBrush()               < UNUSED
GDI32.GetSysColorBrush()               < UNUSED
GDI32.CreateCompatibleBitmap()         < UNUSED
GDI32.CreateCompatibleDC()             < UNUSED
GDI32.BitBlt()                         < UNUSED

SHELL32.ORDINAL716 (ord:02CCh) -> SHELL32.SHMultiFileProperties, Ord:02CCh < UNUSED but ordinal only.


LINKS:
https://www.geoffchappell.com/studies/windows/shell/shell32/history/ords400.htm
