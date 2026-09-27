## Usuario

Como puedo descompilar o extraer el codigo fuente de un archivo .exe desde Termux

## Asistente

Sí, pero depende mucho del tipo de `.exe`. **No existe una descompilación universal que recupere el código fuente original**: normalmente se obtiene código aproximado o ensamblador.


Desde Termux puedes identificar primero qué contiene el `.exe`:


```
Bashfile programa.exe
```


### 1. Si es un `.exe` Windows nativo (PE)


Instala herramientas:


```
Bashpkg install binutils
```


Información básica:


```
Bashobjdump -x programa.exe
```


Extraer cadenas de texto:


```
Bashstrings programa.exe
```


Desensamblar:


```
Bashobjdump -d programa.exe > programa.asm
```


Esto produce **ensamblador**, no el código fuente original.


---


### 2. Si es un programa .NET


Si `file` o el análisis indica que es un ensamblado **.NET/CLR**, existe una posibilidad mucho mayor de recuperar código parecido a C#.


En un PC suelen utilizarse herramientas como **ILSpy**, **dnSpy** o **dotPeek**.


Desde Termux puedes comprobar primero:


```
Bashstrings programa.exe | grep -Ei 'mscoree|\.NET|System\.Runtime|#~'
```


Si aparece información de CLR, probablemente sea .NET.


---


### 3. Si es un programa Windows 16-bit NE


Esto es especialmente importante para tu caso si estás trabajando con ejecutables de Windows 3.x.


Comprueba:


```
Bashfile programa.exe
```


Si aparece algo como:


```
NE executable
```


es un **New Executable (NE)** de 16 bits.


En ese caso herramientas modernas como `objdump` no necesariamente podrán descompilarlo correctamente. Puedes inspeccionar su estructura con:


```
Bashhexdump -C programa.exe | head
```


y localizar la firma:


```
Bashxxd -l 64 programa.exe
```


Un `.EXE` DOS comienza normalmente con:


```
MZ
```


y un Windows 16-bit NE contiene posteriormente:


```
NE
```


---


### 4. Si quieres recuperar el código fuente


Hay que distinguir:


```
.EXE
 │
 ├── recursos ───────────► imágenes, iconos, textos, etc.
 │
 ├── código nativo ──────► ensamblador
 │                         │
 │                         └──► pseudocódigo mediante ingeniería inversa
 │
 ├── .NET/CLR ───────────► IL ──► C#/VB aproximado
 │
 └── Java/otros runtimes ─► bytecode ─► código aproximado
```


Por ejemplo, si tienes un programa C original:


```
Cint suma(int a, int b) {
    return a + b;
}
```


después de compilarlo, el `.exe` **no conserva necesariamente esa representación**. Puede terminar conceptualmente como:


```
asmmov eax, [a]
add eax, [b]
ret
```


Un descompilador puede reconstruir algo parecido a:


```
Cint suma(int a, int b)
{
    return a + b;
}
```


pero **no puede garantizar que ese fuera el código fuente original**.


### Para tu caso concreto


Si me das el nombre/ruta del `.exe`, puedes ejecutar en Termux:


```
Bashfile /ruta/programa.exe
```


y:


```
Bashxxd -l 128 /ruta/programa.exe
```


Pégame **la salida de esos dos comandos** y puedo decirte exactamente si es **DOS MZ, Windows 16-bit NE, Win32 PE, .NET**, etc., y qué herramientas puedes utilizar directamente en Termux para extraer/desensamblar ese ejecutable.

## Usuario

~ $ file /storage/emulated/0/Download/armon3/ARMON.EXE
/storage/emulated/0/Download/armon3/ARMON.EXE: PE32 executable for MS Windows 4.00 (GUI), Intel i386, 8 sections
~ $ xxd -l 128 /storage/emulated/0/Download/armon3/ARMON.EXE
00000000: 4d5a 5000 0200 0000 0400 0f00 ffff 0000  MZP.............
00000010: b800 0000 0000 0000 4000 1a00 0000 0000  ........@.......
00000020: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000030: 0000 0000 0000 0000 0000 0000 0001 0000  ................
00000040: ba10 000e 1fb4 09cd 21b8 014c cd21 9090  ........!..L.!..
00000050: 5468 6973 2070 726f 6772 616d 206d 7573  This program mus
00000060: 7420 6265 2072 756e 2075 6e64 6572 2057  t be run under W
00000070: 696e 3332 0d0a 2437 0000 0000 0000 0000  in32..$7........

## Asistente

Perfecto. Con esa salida ya podemos determinar bastante bien qué es `ARMON.EXE`.


### Qué tienes


```
PE32 executable for MS Windows 4.00 (GUI), Intel i386, 8 sections
```


Eso significa:


- **PE32** → ejecutable Windows moderno de 32 bits, no NE de Windows 3.x.
- **Intel i386** → código x86 de 32 bits.
- **GUI** → aplicación gráfica, no consola.
- **8 sections** → el PE contiene ocho secciones.
- El `MZ` inicial es el encabezado DOS tradicional.
- En `0x40` aparece el pequeño stub DOS que muestra:


```
This program must be run under Win32
```


Por tanto, **ARMON.EXE es un ejecutable Win32**, y esto encaja con que lo estés ejecutando mediante Wine + Box64.


### Lo siguiente que haría en Termux


Primero instala las herramientas:


```
Bashpkg install binutils
```


Después:


```
Bashobjdump -x /storage/emulated/0/Download/armon3/ARMON.EXE
```


Esto nos permitirá ver cosas muy importantes:


```
Machine
Entry point
Image base
Sections
Imports
Exports
```


Especialmente quiero comprobar las **DLL que utiliza ARMON**.


Puedes obtener solamente los imports con:


```
Bashobjdump -p /storage/emulated/0/Download/armon3/ARMON.EXE | grep -A30 "DLL Name"
```


Y las cadenas:


```
Bashstrings -a /storage/emulated/0/Download/armon3/ARMON.EXE > ARMON_strings.txt
```


Luego:


```
Bashless ARMON_strings.txt
```


### Para intentar obtener pseudocódigo


Aquí hay una diferencia importante:


```
ARMON.EXE
   │
   └── PE32 / x86
          │
          ├── código máquina
          │
          ├── datos
          │
          ├── recursos
          │
          └── imports
```


No podemos simplemente convertirlo de nuevo a su `.c` original.


Pero podemos hacer:


```
código máquina
      ↓
desensamblado
      ↓
análisis de flujo
      ↓
pseudocódigo C aproximado
```


Una herramienta especialmente adecuada para esto es **Ghidra** o **Cutter/Radare2**. En Android/Termux, sin embargo, antes de intentar instalar algo pesado, podemos determinar **qué compilador y qué estructura tiene ARMON.EXE**.


Por ejemplo, ejecuta:


```
Bashobjdump -f /storage/emulated/0/Download/armon3/ARMON.EXE
```


y:


```
Bashobjdump -h /storage/emulated/0/Download/armon3/ARMON.EXE
```


y:


```
Bashobjdump -p /storage/emulated/0/Download/armon3/ARMON.EXE | head -100
```


**Pégame esas tres salidas.** Con ellas podemos identificar la estructura del ejecutable y decidir cuál es el método más apropiado para extraer **recursos, strings, imports y finalmente pseudocódigo** sin tocar tu `ARMON.EXE` original.

## Usuario

~ $ objdump -x /storage/emulated/0/Download/armon3/ARMON.EXE

/storage/emulated/0/Download/armon3/ARMON.EXE:  file format coff-i386
architecture: i386
start address: 0x00258520
Characteristics 0x818e
        executable
        line numbers stripped
        symbols stripped
        little endian
        32 bit words
        big endian

Time/Date               Fri Jun 19 18:22:17 1992
Magic                   010b    (PE32)
MajorLinkerVersion      2
MinorLinkerVersion      25
SizeOfCode              00257800
SizeOfInitializedData   00035600
SizeOfUninitializedData 00000000
AddressOfEntryPoint     00258520
BaseOfCode              00001000
BaseOfData              00259000
ImageBase               00400000
SectionAlignment        00001000
FileAlignment           00000200
MajorOSystemVersion     1
MinorOSystemVersion     0
MajorImageVersion       0
MinorImageVersion       0
MajorSubsystemVersion   4
MinorSubsystemVersion   0
Win32Version            00000000
SizeOfImage             002aa000
SizeOfHeaders           00000400
CheckSum                00000000
Subsystem               00000002        (Windows GUI)
DllCharacteristics      00000000
SizeOfStackReserve      00100000
SizeOfStackCommit       00004000
SizeOfHeapReserve       00100000
SizeOfHeapCommit        00001000
LoaderFlags             00000000
NumberOfRvaAndSizes     00000010

The Data Directory
Entry 0 00000000 00000000 Export Directory [.edata (or where ever we found it)]
Entry 1 00274000 00002406 Import Directory [parts of .idata]
Entry 2 00293000 00016400 Resource Directory [.rsrc]
Entry 3 00000000 00000000 Exception Directory [.pdata]
Entry 4 00000000 00000000 Security Directory
Entry 5 00279000 00019484 Base Relocation Directory [.reloc]
Entry 6 00000000 00000000 Debug Directory
Entry 7 00000000 00000000 Description Directory
Entry 8 00000000 00000000 Special Directory
Entry 9 00278000 00000018 Thread Storage Directory [.tls]
Entry a 00000000 00000000 Load Configuration Directory
Entry b 00000000 00000000 Bound Import Directory
Entry c 00000000 00000000 Import Address Table Directory
Entry d 00000000 00000000 Delay Import Directory
Entry e 00000000 00000000 CLR Runtime Header
Entry f 00000000 00000000 Reserved
TLS directory:
  StartAddressOfRawData: 0x677000
  EndAddressOfRawData: 0x677010
  AddressOfIndex: 0x65d4cc
  AddressOfCallBacks: 0x678010
  SizeOfZeroFill: 0
  Characteristics: 0
  Alignment: 0


The Import Tables:
  lookup 00000000 time 00000000 fwd 00000000 name 002747dc addr 0027412c

    DLL Name: kernel32.dll
    Hint/Ord  Name
           0  GetCurrentThreadId
           0  DeleteCriticalSection
           0  LeaveCriticalSection
           0  EnterCriticalSection
           0  InitializeCriticalSection
           0  VirtualFree
           0  VirtualAlloc
           0  LocalFree
           0  LocalAlloc
           0  InterlockedDecrement
           0  InterlockedIncrement
           0  VirtualQuery
           0  WideCharToMultiByte
           0  MultiByteToWideChar
           0  lstrlenA
           0  lstrcpynA
           0  lstrcpyA
           0  LoadLibraryExA
           0  GetThreadLocale
           0  GetStartupInfoA
           0  GetProcAddress
           0  GetModuleHandleA
           0  GetModuleFileNameA
           0  GetLocaleInfoA
           0  GetLastError
           0  GetCommandLineA
           0  FreeLibrary
           0  FindFirstFileA
           0  FindClose
           0  ExitProcess
           0  WriteFile
           0  UnhandledExceptionFilter
           0  SetFilePointer
           0  SetEndOfFile
           0  RtlUnwind
           0  ReadFile
           0  RaiseException
           0  GetStdHandle
           0  GetFileSize
           0  GetFileType
           0  CreateFileA
           0  CloseHandle

  lookup 00000000 time 00000000 fwd 00000000 name 00274ac8 addr 002741d8

    DLL Name: user32.dll
    Hint/Ord  Name
           0  GetKeyboardType
           0  LoadStringA
           0  MessageBoxA
           0  CharNextA

  lookup 00000000 time 00000000 fwd 00000000 name 00274b0e addr 002741ec

    DLL Name: advapi32.dll
    Hint/Ord  Name
           0  RegQueryValueExA
           0  RegOpenKeyExA
           0  RegCloseKey

  lookup 00000000 time 00000000 fwd 00000000 name 00274b4e addr 002741fc

    DLL Name: oleaut32.dll
    Hint/Ord  Name
           0  VariantChangeTypeEx
           0  VariantCopyInd
           0  VariantClear
           0  SysStringLen
           0  SysFreeString
           0  SysReAllocStringLen
           0  SysAllocStringLen

  lookup 00000000 time 00000000 fwd 00000000 name 00274bde addr 0027421c

    DLL Name: kernel32.dll
    Hint/Ord  Name
           0  TlsSetValue
           0  TlsGetValue
           0  LocalAlloc
           0  GetModuleHandleA
           0  GetModuleFileNameA

  lookup 00000000 time 00000000 fwd 00000000 name 00274c40 addr 00274234

    DLL Name: advapi32.dll
    Hint/Ord  Name
           0  RegQueryValueExA
           0  RegOpenKeyExA
           0  RegCloseKey

  lookup 00000000 time 00000000 fwd 00000000 name 00274c80 addr 00274244

    DLL Name: kernel32.dll
    Hint/Ord  Name
           0  lstrcpyA
           0  WriteFile
           0  WaitForSingleObject
           0  VirtualQuery
           0  VirtualAlloc
           0  Sleep
           0  SizeofResource
           0  SetThreadLocale
           0  SetFilePointer
           0  SetEvent
           0  SetErrorMode
           0  SetEndOfFile
           0  ReadFile
           0  MulDiv
           0  LockResource
           0  LoadResource
           0  LoadLibraryA
           0  LeaveCriticalSection
           0  InitializeCriticalSection
           0  GlobalUnlock
           0  GlobalSize
           0  GlobalReAlloc
           0  GlobalHandle
           0  GlobalLock
           0  GlobalFree
           0  GlobalDeleteAtom
           0  GlobalAlloc
           0  GlobalAddAtomA
           0  GetVersionExA
           0  GetVersion
           0  GetTickCount
           0  GetThreadLocale
           0  GetSystemInfo
           0  GetProfileStringA
           0  GetProcAddress
           0  GetModuleHandleA
           0  GetModuleFileNameA
           0  GetLocaleInfoA
           0  GetLocalTime
           0  GetLastError
           0  GetDiskFreeSpaceA
           0  GetCurrentThreadId
           0  GetCurrentProcessId
           0  GetCPInfo
           0  FreeResource
           0  FreeLibrary
           0  FormatMessageA
           0  FindResourceA
           0  FindNextFileA
           0  FindFirstFileA
           0  FindClose
           0  FileTimeToLocalFileTime
           0  FileTimeToDosDateTime
           0  EnumCalendarInfoA
           0  EnterCriticalSection
           0  DeleteCriticalSection
           0  CreateThread
           0  CreateFileA
           0  CreateEventA
           0  CompareStringA
           0  CloseHandle

  lookup 00000000 time 00000000 fwd 00000000 name 0027509e addr 0027433c

    DLL Name: gdi32.dll
    Hint/Ord  Name
           0  UnrealizeObject
           0  StretchBlt
           0  StartPage
           0  StartDocA
           0  SetWindowOrgEx
           0  SetWindowExtEx
           0  SetWinMetaFileBits
           0  SetViewportOrgEx
           0  SetViewportExtEx
           0  SetTextColor
           0  SetStretchBltMode
           0  SetROP2
           0  SetPixel
           0  SetMapMode
           0  SetEnhMetaFileBits
           0  SetDIBColorTable
           0  SetBrushOrgEx
           0  SetBkMode
           0  SetBkColor
           0  SetAbortProc
           0  SelectPalette
           0  SelectObject
           0  SaveDC
           0  RoundRect
           0  RestoreDC
           0  Rectangle
           0  RectVisible
           0  RealizePalette
           0  Polyline
           0  Polygon
           0  PolyPolyline
           0  PlayEnhMetaFile
           0  Pie
           0  PatBlt
           0  MoveToEx
           0  MaskBlt
           0  LineTo
           0  IntersectClipRect
           0  GetWindowOrgEx
           0  GetWinMetaFileBits
           0  GetTextMetricsA
           0  GetTextExtentPointA
           0  GetTextExtentPoint32A
           0  GetSystemPaletteEntries
           0  GetStockObject
           0  GetPixel
           0  GetPaletteEntries
           0  GetObjectA
           0  GetEnhMetaFilePaletteEntries
           0  GetEnhMetaFileHeader
           0  GetEnhMetaFileBits
           0  GetDeviceCaps
           0  GetDIBits
           0  GetDIBColorTable
           0  GetDCOrgEx
           0  GetCurrentPositionEx
           0  GetClipBox
           0  GetBrushOrgEx
           0  GetBkMode
           0  GetBitmapBits
           0  ExtTextOutA
           0  ExtCreatePen
           0  ExcludeClipRect
           0  EndPage
           0  EndDoc
           0  Ellipse
           0  DeleteObject
           0  DeleteEnhMetaFile
           0  DeleteDC
           0  CreateSolidBrush
           0  CreatePenIndirect
           0  CreatePalette
           0  CreateICA
           0  CreateHalftonePalette
           0  CreateFontIndirectA
           0  CreateDIBitmap
           0  CreateDIBSection
           0  CreateDCA
           0  CreateCompatibleDC
           0  CreateCompatibleBitmap
           0  CreateBrushIndirect
           0  CreateBitmap
           0  CopyEnhMetaFileA
           0  BitBlt
           0  Arc
           0  AbortDoc

  lookup 00000000 time 00000000 fwd 00000000 name 00275620 addr 00274498

    DLL Name: user32.dll
    Hint/Ord  Name
           0  WindowFromPoint
           0  WinHelpA
           0  WaitMessage
           0  ValidateRect
           0  UpdateWindow
           0  UnregisterClassA
           0  UnionRect
           0  UnhookWindowsHookEx
           0  TranslateMessage
           0  TranslateMDISysAccel
           0  TrackPopupMenu
           0  SystemParametersInfoA
           0  ShowWindow
           0  ShowScrollBar
           0  ShowOwnedPopups
           0  ShowCursor
           0  SetWindowsHookExA
           0  SetWindowTextA
           0  SetWindowPos
           0  SetWindowPlacement
           0  SetWindowLongA
           0  SetTimer
           0  SetScrollRange
           0  SetScrollPos
           0  SetScrollInfo
           0  SetRect
           0  SetPropA
           0  SetMenuItemInfoA
           0  SetMenu
           0  SetKeyboardState
           0  SetForegroundWindow
           0  SetFocus
           0  SetCursor
           0  SetClipboardData
           0  SetClassLongA
           0  SetCapture
           0  SetActiveWindow
           0  SendMessageA
           0  ScrollWindowEx
           0  ScrollWindow
           0  ScreenToClient
           0  RemovePropA
           0  RemoveMenu
           0  ReleaseDC
           0  ReleaseCapture
           0  RegisterWindowMessageA
           0  RegisterClipboardFormatA
           0  RegisterClassA
           0  PtInRect
           0  PostQuitMessage
           0  PostMessageA
           0  PeekMessageA
           0  OpenClipboard
           0  OffsetRect
           0  OemToCharA
           0  MessageBoxA
           0  MessageBeep
           0  MapWindowPoints
           0  MapVirtualKeyA
           0  LoadStringA
           0  LoadKeyboardLayoutA
           0  LoadIconA
           0  LoadCursorA
           0  LoadBitmapA
           0  KillTimer
           0  IsZoomed
           0  IsWindowVisible
           0  IsWindowEnabled
           0  IsWindow
           0  IsRectEmpty
           0  IsIconic
           0  IsDialogMessageA
           0  IsChild
           0  IsCharAlphaNumericA
           0  IsCharAlphaA
           0  InvalidateRect
           0  IntersectRect
           0  InsertMenuItemA
           0  InsertMenuA
           0  InflateRect
           0  GetWindowThreadProcessId
           0  GetWindowTextA
           0  GetWindowRect
           0  GetWindowPlacement
           0  GetWindowLongA
           0  GetWindowDC
           0  GetTopWindow
           0  GetSystemMetrics
           0  GetSystemMenu
           0  GetSysColor
           0  GetSubMenu
           0  GetScrollRange
           0  GetScrollPos
           0  GetScrollInfo
           0  GetPropA
           0  GetParent
           0  GetWindow
           0  GetMessageTime
           0  GetMenuStringA
           0  GetMenuState
           0  GetMenuItemInfoA
           0  GetMenuItemID
           0  GetMenuItemCount
           0  GetMenu
           0  GetLastActivePopup
           0  GetKeyboardState
           0  GetKeyboardLayoutList
           0  GetKeyboardLayout
           0  GetKeyState
           0  GetKeyNameTextA
           0  GetIconInfo
           0  GetForegroundWindow
           0  GetFocus
           0  GetDoubleClickTime
           0  GetDlgItem
           0  GetDesktopWindow
           0  GetDCEx
           0  GetDC
           0  GetCursorPos
           0  GetCursor
           0  GetClipboardData
           0  GetClientRect
           0  GetClassInfoA
           0  GetCapture
           0  GetActiveWindow
           0  FrameRect
           0  FindWindowA
           0  FillRect
           0  EqualRect
           0  EnumWindows
           0  EnumThreadWindows
           0  EnumClipboardFormats
           0  EndPaint
           0  EndDeferWindowPos
           0  EnableWindow
           0  EnableScrollBar
           0  EnableMenuItem
           0  EmptyClipboard
           0  DrawTextA
           0  DrawMenuBar
           0  DrawIconEx
           0  DrawIcon
           0  DrawFrameControl
           0  DrawFocusRect
           0  DrawEdge
           0  DispatchMessageA
           0  DestroyWindow
           0  DestroyMenu
           0  DestroyIcon
           0  DestroyCursor
           0  DeleteMenu
           0  DeferWindowPos
           0  DefWindowProcA
           0  DefMDIChildProcA
           0  DefFrameProcA
           0  CreateWindowExA
           0  CreatePopupMenu
           0  CreateMenu
           0  CreateIcon
           0  CloseClipboard
           0  ClientToScreen
           0  CheckMenuItem
           0  CallWindowProcA
           0  CallNextHookEx
           0  BeginPaint
           0  BeginDeferWindowPos
           0  CharLowerBuffA
           0  CharLowerA
           0  CharUpperBuffA
           0  AdjustWindowRectEx
           0  ActivateKeyboardLayout

  lookup 00000000 time 00000000 fwd 00000000 name 00276134 addr 00274748

    DLL Name: ole32.dll
    Hint/Ord  Name
           0  IsEqualGUID

  lookup 00000000 time 00000000 fwd 00000000 name 0027614c addr 00274750

    DLL Name: comctl32.dll
    Hint/Ord  Name
           0  ImageList_SetIconSize
           0  ImageList_GetIconSize
           0  ImageList_Write
           0  ImageList_Read
           0  ImageList_GetDragImage
           0  ImageList_DragShowNolock
           0  ImageList_SetDragCursorImage
           0  ImageList_DragMove
           0  ImageList_DragLeave
           0  ImageList_DragEnter
           0  ImageList_EndDrag
           0  ImageList_BeginDrag
           0  ImageList_Remove
           0  ImageList_DrawEx
           0  ImageList_Replace
           0  ImageList_Draw
           0  ImageList_GetBkColor
           0  ImageList_SetBkColor
           0  ImageList_ReplaceIcon
           0  ImageList_Add
           0  ImageList_GetImageCount
           0  ImageList_Destroy
           0  ImageList_Create

  lookup 00000000 time 00000000 fwd 00000000 name 00276358 addr 002747b0

    DLL Name: winspool.drv
    Hint/Ord  Name
           0  OpenPrinterA
           0  EnumPrintersA
           0  DocumentPropertiesA
           0  ClosePrinter

  lookup 00000000 time 00000000 fwd 00000000 name 002763ac addr 002747c4

    DLL Name: comdlg32.dll
    Hint/Ord  Name
           0  PrintDlgA
           0  GetSaveFileNameA
           0  GetOpenFileNameA

  lookup 00000000 time 00000000 fwd 00000000 name 002763ee addr 002747d4

    DLL Name: kernel32.dll
    Hint/Ord  Name
           0  MulDiv


Sections:
Idx Name          Size     VMA      Type
  0 CODE          0025763c 00401000 TEXT
  1 DATA          00003254 00659000 DATA
  2 BSS           00000000 0065d000
  3 .idata        00002406 00674000 DATA
  4 .tls          00000000 00677000
  5 .rdata        00000018 00678000 DATA
  6 .reloc        00019484 00679000 DATA
  7 .rsrc         00016400 00693000 DATA

SYMBOL TABLE:

## Asistente

Con esta salida ya podemos caracterizar `ARMON.EXE` con bastante precisión. Hay un dato especialmente importante: **no es un ejecutable de Windows 3.x/16-bit**; es un **PE32 Win32 de 32 bits**, y además tiene los símbolos eliminados.


### 1. Qué contiene ARMON.EXE


| Característica | Resultado |
| --- | --- |
| Formato | PE32 |
| CPU | Intel i386 / x86 32-bit |
| GUI | Sí |
| Punto de entrada | `0x00258520` |
| ImageBase | `0x00400000` |
| Código | ~2,4 MB |
| Datos inicializados | ~218 KB |
| Recursos | ~91 KB |
| Secciones | 8 |
| Símbolos | **eliminados** |
| Debug info | **ausente** |
| .NET/CLR | **no** |
| Relocaciones | Sí |
| TLS | Sí |


La ausencia de:


```
Entry 14 ... CLR Runtime Header
```


confirma que **no es .NET**. Por tanto, ILSpy/dnSpy no nos servirían.


---


## 2. Lo más importante: perdió los nombres originales


Tu salida dice:


```
Characteristics 0x818e
        ...
        line numbers stripped
        symbols stripped
```


y al final:


```
SYMBOL TABLE:
```


sin símbolos.


Eso significa que el ejecutable ya no contiene cosas como:


```
main()
CrearVentana()
CalcularArmonico()
...
```


En su lugar encontraremos direcciones:


```
asm00401000
00401020
00401035
...
```


Por eso la recuperación del código fuente será **ingeniería inversa**, no una conversión directa `.exe → .c`.


---


## 3. Pero tenemos una ventaja enorme: los imports


ARMON utiliza directamente APIs de:


```
kernel32.dll
user32.dll
gdi32.dll
advapi32.dll
oleaut32.dll
ole32.dll
comctl32.dll
winspool.drv
comdlg32.dll
```


Y la selección es bastante reveladora.


Por ejemplo, utiliza:


```
CreateWindowExA
RegisterClassA
DispatchMessageA
DefWindowProcA
CreateMenu
CreatePopupMenu
CreateIcon
CreateCursor
Dialog...
```


Eso indica que el programa tiene una **interfaz gráfica Win32 tradicional**, con ventanas, menús, diálogos, mensajes, etc.


También utiliza una cantidad considerable de GDI:


```
BitBlt
CreateCompatibleBitmap
CreateDIBSection
CreateFontIndirectA
Ellipse
LineTo
Polygon
Polyline
Rectangle
TextOut...
```


Esto es particularmente interesante para ARMON porque significa que podemos localizar posteriormente las partes del programa responsables de **dibujar gráficos**.


---


## 4. Hay otra pista interesante: OLE Automation


Aparecen:


```
VariantChangeTypeEx
VariantCopyInd
VariantClear
SysStringLen
SysFreeString
SysReAllocStringLen
SysAllocStringLen
```


Eso significa que ARMON utiliza **VARIANT/BSTR y mecanismos de automatización OLE**.


No significa necesariamente que sea Visual Basic, Delphi, C++, etc., pero sí proporciona una pista sobre la tecnología utilizada.


Por ahora **no afirmaría todavía qué lenguaje lo generó**.


---


# 5. Podemos intentar identificar el compilador


Este debería ser nuestro siguiente paso antes de desensamblar los 2,4 MB.


Ejecuta:


```
Bashstrings -a /storage/emulated/0/Download/armon3/ARMON.EXE | grep -Ei \
'borland|delphi|turbo|microsoft|visual basic|msvc|visual c|watcom|powerbasic|gcc|mingw|symantec|metrowerks|runtime|copyright'
```


Y también:


```
Bashstrings -a /storage/emulated/0/Download/armon3/ARMON.EXE | grep -Ei \
'ARMON|ARMON 3|armonia|autor|version|copyright|199|200|201'
```


Esto podría encontrar directamente información incrustada por el compilador o por el propio programa.


---


# 6. Extraer los recursos es muy interesante


Tu PE tiene:


```
Resource Directory
RVA: 00293000
Size: 00016400
```


Es decir, unos **89 KB de recursos**.


Ahí podrían estar:


- iconos
- menús
- diálogos
- bitmaps
- cursores
- cadenas
- información de versión
- aceleradores


Y esto puede ser extremadamente útil para reconstruir cómo estaba diseñado ARMON.


Podemos intentar:


```
Bashobjdump -p /storage/emulated/0/Download/armon3/ARMON.EXE | grep -A100 -i resource
```


Pero hay una herramienta aún mejor si está disponible en tu Termux:


```
Bashwhich wrestool
```


Si responde con una ruta, podemos hacer:


```
Bashwrestool -l /storage/emulated/0/Download/armon3/ARMON.EXE
```


---


# 7. Podemos localizar el código de entrada


Tenemos:


```
AddressOfEntryPoint = 0x00258520
ImageBase             = 0x00400000
```


Por tanto, la dirección virtual de entrada es:


```
0x00400000 + 0x00258520
```


es decir:


```
0x00658520
```


Podemos desensamblar alrededor de ella:


```
Bashobjdump -D -Mintel /storage/emulated/0/Download/armon3/ARMON.EXE \
| grep -A80 -B20 '658520'
```


Pero hay una consideración: **el punto de entrada no necesariamente es `main()`**.


Normalmente tenemos algo conceptualmente parecido a:


```
Windows loader
      ↓
Entry Point
      ↓
runtime del compilador
      ↓
inicialización
      ↓
main / WinMain
      ↓
programa ARMON
```


Por eso primero debemos identificar el runtime.


---


# 8. Podemos producir el ensamblador completo


El código ejecutable ocupa aproximadamente:


```
0x0025763c ≈ 2,46 MB
```


Podemos extraerlo a un archivo:


```
Bashobjdump -d -Mintel /storage/emulated/0/Download/armon3/ARMON.EXE \
> ~/ARMON.asm
```


Después:


```
Bashls -lh ~/ARMON.asm
```


y podremos buscar patrones concretos sin tener que mostrar todo.


Por ejemplo:


```
Bashgrep -n "call" ~/ARMON.asm | head -50
```


---


# 9. Pero yo no empezaría todavía por `objdump -d`


Para **recuperar el funcionamiento del programa**, una herramienta de ingeniería inversa como **Ghidra, radare2/Cutter o Binary Ninja** es mucho más adecuada que `objdump`.


La diferencia conceptual es:


```
objdump
   ↓
ensamblador

Ghidra / radare2
   ↓
ensamblador
   ↓
funciones
   ↓
flujo de control
   ↓
referencias
   ↓
pseudocódigo
```


Y nuestro objetivo sería llegar a algo como:


```
Cvoid funcion_00412340(...)
{
    ...
}
```


y posteriormente determinar qué hace esa función.


---


## 10. Hay una pista especialmente interesante en tu ARMON


El programa tiene:


```
gdi32.dll
```


con una cantidad enorme de funciones de dibujo:


```
CreateDIBSection
BitBlt
StretchBlt
Polyline
Polygon
Pie
Ellipse
Rectangle
TextOut
CreateFontIndirectA
...
```


Eso significa que podemos hacer algo mucho más útil que simplemente "descompilar": **buscar las rutinas que dibujan la interfaz y los gráficos de ARMON y reconstruir progresivamente su arquitectura interna**.


Y dado que tienes el programa funcionando mediante Wine/Box64, podemos incluso contrastar posteriormente el comportamiento observado con el código desensamblado.


### Siguiente paso


Antes de instalar nada pesado, ejecutaría **estos tres comandos**:


```
Bashstrings -a /storage/emulated/0/Download/armon3/ARMON.EXE | grep -Ei \
'borland|delphi|turbo|microsoft|visual basic|msvc|visual c|watcom|powerbasic|gcc|mingw|symantec'
```


```
Bashwhich wrestool
```


```
Bashobjdump -p /storage/emulated/0/Download/armon3/ARMON.EXE | grep -A20 -B5 -i 'resource'
```


Con esas tres salidas podemos intentar determinar **qué tecnología/compilador produjo ARMON, qué recursos contiene y cuál sería la ruta más eficaz para recuperar pseudocódigo**.

## Usuario

Espera te voy a proporcionar informacion de otro ejecutable, para que me indiques cual es el mejor opcionado para obtener el codigo fuente original.

## Asistente

Perfecto. Proporcióname la información del otro ejecutable y compararé ambos.


Para determinar cuál tiene **mayor posibilidad de recuperar código fuente cercano al original**, me fijaré especialmente en:


- tipo de ejecutable: **PE32, NE, .NET, etc.**
- lenguaje/compilador probable
- presencia de **símbolos**
- información de depuración
- imports y runtime
- recursos
- nivel de optimización
- si conserva nombres de funciones/variables
- si existe bytecode intermedio (.NET, Delphi, VB6, etc.)
- fecha y características del enlazado


No asumiré que el ejecutable más grande o más moderno sea mejor candidato: **la estructura interna y la información que haya sobrevivido a la compilación son mucho más importantes**.


Puedes proporcionarme el `objdump -x` del segundo ejecutable completo, como hiciste con `ARMON.EXE`.

## Usuario

~ $ file /storage/emulated/0/Download/armon/ARMON.EXE
/storage/emulated/0/Download/armon/ARMON.EXE: MS-DOS executable, NE version 6.01 for MS Windows 3.10 (EXE) (GUI)
~ $ xxd -l 128 /storage/emulated/0/Download/armon3/ARMON.EXE
00000000: 4d5a 5000 0200 0000 0400 0f00 ffff 0000  MZP.............
00000010: b800 0000 0000 0000 4000 1a00 0000 0000  ........@.......
00000020: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000030: 0000 0000 0000 0000 0000 0000 0001 0000  ................
00000040: ba10 000e 1fb4 09cd 21b8 014c cd21 9090  ........!..L.!..
00000050: 5468 6973 2070 726f 6772 616d 206d 7573  This program mus
00000060: 7420 6265 2072 756e 2075 6e64 6572 2057  t be run under W
00000070: 696e 3332 0d0a 2437 0000 0000 0000 0000  in32..$7........

## Asistente

Sí: **este segundo ejecutable cambia mucho la comparación**. Pero hay un detalle que debemos corregir antes de concluir.


### Lo que ya sabemos


El segundo archivo:


```
/storage/emulated/0/Download/armon/ARMON.EXE
```


es:


```
MS-DOS executable, NE version 6.01 for MS Windows 3.10 (EXE) (GUI)
```


Por tanto:


|  | `armon3/ARMON.EXE` | `armon/ARMON.EXE` |
| --- | --- | --- |
| Formato | PE32 | **NE 16-bit** |
| Plataforma | Win32 | **Windows 3.1** |
| CPU | i386 | probablemente 16-bit x86 |
| CLR/.NET | No | No |
| Símbolos | eliminados | **aún por comprobar** |
| Época | 1992 según PE timestamp | Windows 3.1 |
| Relación probable | versión Win32 | **posible versión original** |


Y hay una pista muy importante: **el segundo está identificado explícitamente como NE 6.01 para Windows 3.10**. Si estamos intentando descubrir cuál pudo haber sido el ejecutable original de ARMÓN para Windows 3.x, este es el candidato que debemos investigar primero.


### Pero hay un error en el comando


Dijiste:


```
Bashfile /storage/emulated/0/Download/armon/ARMON.EXE
```


pero después ejecutaste:


```
Bashxxd -l 128 /storage/emulated/0/Download/armon3/ARMON.EXE
```


Ese `xxd` corresponde al **primer PE32**, no al segundo NE.


Por eso todavía no debemos sacar conclusiones sobre cuánto código recuperable conserva el NE.


### Haz esto sobre el segundo archivo


Primero:


```
Bashxxd -l 256 /storage/emulated/0/Download/armon/ARMON.EXE
```


Después:


```
Bashobjdump -x /storage/emulated/0/Download/armon/ARMON.EXE
```


Si `objdump` no consigue interpretarlo correctamente, **no pasa nada**; es normal con ejecutables NE antiguos.


También ejecuta:


```
Bashstrings -a /storage/emulated/0/Download/armon/ARMON.EXE | head -100
```


Y:


```
Bashstrings -a /storage/emulated/0/Download/armon/ARMON.EXE | grep -Ei \
'borland|delphi|microsoft|visual basic|quickbasic|turbo|watcom|symantec|metaware|copyright|version'
```


### Qué vamos a comparar


La pregunta realmente importante no es simplemente:


> "¿Cuál se puede descompilar?"


sino:


> **"¿Cuál conserva más información que permita reconstruir el código fuente original de ARMÓN?"**


Y aquí hay una posibilidad especialmente interesante:


```
ARMÓN original
      │
      ▼
Windows 3.1 NE
      │
      │ posible evolución/recompilación
      ▼
ARMON3 PE32
```


Si `armon/ARMON.EXE` es efectivamente una versión anterior del mismo programa, podemos comparar **ambos binarios** para identificar qué partes permanecieron iguales.


Eso podría ser mucho más poderoso que analizar solamente el PE32.


**Pásame el `objdump -x` del NE**. A partir de ahí podemos determinar cuál de los dos está mejor posicionado para reconstruir el código fuente y, sobre todo, si el NE conserva alguna información que el PE32 perdió.

## Usuario

~ $ xxd -l 128 /storage/emulated/0/Download/armon/ARMON.EXE

00000000: 4d5a 0001 0100 0000 0800 1000 ffff 0800  MZ..............
00000010: 0001 0000 0000 0000 4000 0000 0000 0000  ........@.......
00000020: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000030: 0000 0000 0000 0000 0000 0000 0001 0000  ................
00000040: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000050: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000060: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000070: 0000 0000 0000 0000 0000 0000 0000 0000  ................

## Asistente

Esto confirma una diferencia importante entre ambos archivos.


### Segundo `ARMON.EXE`


El encabezado:


```
4d 5a
```


es `MZ`, pero a diferencia del primero **no aparece el stub DOS estándar**:


```
This program must be run under Win32
```


Además, `file` ya lo identificó como:


```
MS-DOS executable, NE version 6.01 for MS Windows 3.10 (EXE) (GUI)
```


Por tanto, estamos ante un **NE genuino de Windows 3.1 de 16 bits**.


Y esto hace que el segundo archivo sea **especialmente interesante para nuestro objetivo de recuperar el código de ARMÓN original**.


### Pero todavía falta el dato decisivo


Necesitamos saber qué información conserva el NE: segmentos, nombres de módulos, imports, exports, recursos, nombres de funciones, etc.


Ejecuta:


```
Bashobjdump -x /storage/emulated/0/Download/armon/ARMON.EXE
```


Si `objdump` devuelve un error del tipo:


```
file format not recognized
```


o no interpreta correctamente el NE, prueba:


```
Bashobjdump -f /storage/emulated/0/Download/armon/ARMON.EXE
```


y:


```
Bashstrings -a /storage/emulated/0/Download/armon/ARMON.EXE | head -200
```


También es muy útil:


```
Bashstrings -a /storage/emulated/0/Download/armon/ARMON.EXE | grep -Ei \
'borland|microsoft|visual|basic|turbo|watcom|symantec|copyright|version|ARMON'
```


### Algo importante sobre el `xxd`


Los primeros 128 bytes **no contienen todavía la firma `NE`**, porque en los ejecutables NE la cabecera NE se encuentra mediante el puntero ubicado en el encabezado DOS, no necesariamente dentro de esos primeros 128 bytes.


En tu caso podemos localizarla directamente:


```
Bashxxd -g 1 /storage/emulated/0/Download/armon/ARMON.EXE | grep -i '4e 45'
```


`4e 45` = ASCII:


```
NE
```


O, más directamente:


```
Bashgrep -abo $'NE' /storage/emulated/0/Download/armon/ARMON.EXE
```


Esto nos dará la **posición exacta de la cabecera NE**.


---


### Mi evaluación provisional


Con lo que tenemos hasta ahora:


**`armon/ARMON.EXE` es el candidato que merece prioridad para buscar el código fuente histórico de ARMÓN.**


No porque NE sea automáticamente más fácil de descompilar, sino porque es un ejecutable **16-bit de Windows 3.1**, y si corresponde a una versión anterior del mismo programa, puede conservar estructuras y características del programa original que fueron transformadas o eliminadas al crear el `PE32`.


Pero todavía **no podemos decir que contenga más información recuperable**. Para eso necesitamos inspeccionar su cabecera NE y sus tablas.


Pásame el resultado de:


```
Bashobjdump -x /storage/emulated/0/Download/armon/ARMON.EXE
```


y podremos hacer ya una **comparación técnica directa NE vs PE32** para determinar cuál ofrece mejores posibilidades de reconstrucción.

## Usuario

~ $ objdump -x /storage/emulated/0/Download/armon/ARMON.EXE
objdump: error: '/storage/emulated/0/Download/armon/ARMON.EXE': The file was not recognized as a valid object file
~ $ objdump -f /storage/emulated/0/Download/armon/ARMON.EXE
objdump: error: '/storage/emulated/0/Download/armon/ARMON.EXE': The file was not recognized as a valid object file
~ $ strings -a /storage/emulated/0/Download/armon/ARMON.EXE | head -200
This program requires Microsoft Windows.
$                                                                     NE
>?4*
iD[j
~ao6
MAINICON        TMISGRAPH
TFORM1
TLISTAWS
TFORM2
TGPSELECTOR     TABOUTBOX
TGEODE
TENTRETENER
TFORM3
TAUXMENU        TPROGRESO
TFORM4
TMLISTA
SPINDOWN
SPINUP
BBHELP
BBNO
BBOK
BBYES
BBCANCEL
BBCLOSE
BBRETRY
BBALL
BBABORT
BBIGNORE
ARMON
COMMDLG
KERNEL
KERNEL
TOOLHELP
KERNEL
USER
KEYBOARD
KERNEL
USER
WIN87EM
KEYBOARD
?M@^
?M)b
        ARMON.EXE
TCommonDialog
TCommonDialog"
Dialogs
Ctl3DZ
HelpContext
TOpenOption
ofReadOnly
ofOverwritePrompt
ofHideReadOnly
ofNoChangeDir
ofShowHelp
ofNoValidate
ofAllowMultiSelect
ofExtensionDifferent
ofPathMustExist
ofFileMustExist
ofCreatePrompt
ofShareAware
ofNoReadOnlyReturn
ofNoTestFileCreate
TOpenOptions
TFileExt
TFileEditStyle
fsEdit
fsComboBox
TDlgControl
TComboButton
TOpenDialog
TOpenDialog
Dialogs
DefaultExt
FileEditStyle
FileName
Filterw%
FilterIndex
InitialDir
HistoryList
Options
Title
TSaveDialog
Execute
TSaveDialogx
Dialogs
TPrinterSetupDialog
TPrinterSetupDialog
Dialogs
TPrintRange
prAllPages
prSelection
prPageNums
TPrintDialogOption
poPrintToFile
poPageNums
poSelection     poWarning
poHelp
poDisablePrintToFile
TPrintDialogOptions
TPrintDialog
TPrintDialog^
Dialogs
Collatew%
Copiesw%#
FromPagew%C
MinPagew%
MaxPage$
Options
PrintToFile|
PrintRangew%
ToPage
TDropListBox
TDlgEditControl
TCommonDlg
<(ug
(u#&
@jAj
64%j
ZXRP
HHPj
64%j
E!tq
DropListButton
0 RP
TCommonDialogList
U2RP
U2RP
pu)&
`&;E
`ZXRP
cqf+
""l
""{
WjHj
E'@t
=\u&
G"G(
Wj4j
+*PW
Wj4j
E$ t
MS Sans Serif
Message
Image
m+RP
%/RPj
a,RP
j.RP
(/h/
.       0j7
&;E$}]j
E Z+
Q[-2
|2z2
S+43
MS Sans Serif
T3RP
S%z3
YB.4
>Ed4j
k4RP
,q5j
,S6j
M/a6
~6jM
""'7
f8RP
V8RP
m8RP
TButtonLayout
blGlyphLeft
blGlyphRight
blGlyphTop
blGlyphBottom
TButtonStyle
bsAutoDetect
bsWin31
bsNew
TNumGlyphs
TSpeedButton
TSpeedButton
Buttons
AllowAllUpw%
GroupIndex
Downf
Caption
Enabled"
Font
Glyph
Layoutw%
Margin
        NumGlyphs
ParentFont
ParentShowHint
ShowHintw%W
Spacing
Visibleq
OnClickq
OnDblClickq
~ $ strings -a /storage/emulated/0/Download/armon/ARMON.EXE | grep -Ei \
> 'borland|microsoft|visual|basic|turbo|watcom|symantec|copyright|version|ARMON'
This program requires Microsoft Windows.
ARMON
        ARMON.EXE
Borland Delphi
Portions Copyright (c) 1983,95 Borland
Version
        Copyright
armon
ArmonicoChange
AjustarArmonicas1Click
OpcionesparaFlorArmonica1Click
Armonico`
`AjustarArmonicas1
OpcionesparaFlorArmonica1
Escala para Ondas Armonicas
Armonicos Activos por Defecto
Opciones para la Flor Armonica
armonicoU
armonicoU
armonico        armonico*       armonico+
ARMONICOU
 cartas   Armonico
turboU
armon
armon
Harmonic.txt
armonic.mod
Calculando Armonicos ...
Seleccione el Armonico
armonico recto (Ramses)
armonico polar (Rose)
armonico circular (Kanji)
Clavicula Armonica
Armonico
ArmonicoChange
Aumentar/Disminuir Subarmonico
AjustarArmonicas1
Escala para &Ondas Armonicas
AjustarArmonicas1Click
OpcionesparaFlorArmonica1
%Opciones para la Flor Armonica (p&wf)
OpcionesparaFlorArmonica1Click
Version
Version 1.0
TLabel  Copyright
(Copyright: Miguel Garc
BG#Cannot load older version of TShape  Metafiles

## Asistente

Ahora sí tenemos una conclusión mucho más fuerte. **El segundo `ARMON.EXE` es claramente un ejecutable de Borland Delphi 1**, y contiene una cantidad extraordinariamente útil de información de alto nivel.


La evidencia más directa es:


```
Borland Delphi
Portions Copyright (c) 1983,95 Borland
```


y, sobre todo, los nombres:


```
TFORM1
TLISTAWS
TFORM2
TGPSELECTOR
TABOUTBOX
TGEODE
TENTRETENER
TFORM3
TAUXMENU
TPROGRESO
TFORM4
TMLISTA
```


Además aparecen **nombres de eventos y métodos de la aplicación**:


```
ArmonicoChange
AjustarArmonicas1Click
OpcionesparaFlorArmonica1Click
```


## Por tanto, mi comparación queda así


| Característica | ARMON Win3.1 | ARMON3 Win32 |
| --- | --- | --- |
| Formato | **NE 16-bit** | PE32 32-bit |
| Compilador | **Delphi 1** | probablemente Delphi, por confirmar |
| Código nativo | Sí | Sí |
| Código fuente Pascal original | No conservado íntegramente | No conservado íntegramente |
| Nombres de formularios | **Sí** | posiblemente |
| Nombres de componentes | **Sí** | posiblemente |
| Nombres de eventos | **Sí** | posiblemente |
| Información de formularios DFM | **Sí, potencialmente recuperable** | potencialmente |
| Recursos Delphi | **Sí** | sí |
| Pseudocódigo | posible | posible |
| Reconstrucción del proyecto | **muy prometedora** | posible |
| Candidato para estudiar primero | **★★★★★** | ★★★ |


### Lo extraordinario de este archivo


Por ejemplo, encontramos:


```
TFORM1
TLISTAWS
TFORM2
TGPSELECTOR
TABOUTBOX
TGEODE
TENTRETENER
...
```


Eso no son simples cadenas arbitrarias. Son nombres de clases/componentes asociados a la estructura de una aplicación Delphi.


También:


```
TCommonDialog
TOpenDialog
TSaveDialog
TPrinterSetupDialog
TPrintDialog
TSpeedButton
```


Esto permite reconstruir buena parte de la **estructura visual de la aplicación**.


Y tenemos nombres muy reveladores del propio ARMÓN:


```
ArmonicoChange
AjustarArmonicas1Click
OpcionesparaFlorArmonica1Click
```


Es decir, podemos llegar a saber que existían manejadores de eventos concretos como:


```
pascalprocedure AjustarArmonicas1Click(...);
procedure OpcionesparaFlorArmonica1Click(...);
```


Lo que **no podemos recuperar automáticamente** es el cuerpo original de esos procedimientos. La compilación convirtió ese Pascal en código máquina.


Esto coincide con la documentación histórica sobre Delphi: los formularios/DFM contienen propiedades y estructura de componentes, mientras que el código de los manejadores permanece en los módulos Pascal compilados.


---


# Una conclusión todavía más importante


Tu archivo tiene:


```
Version 1.0
(Copyright: Miguel Garc...
```


y:


```
Harmonic.txt
armonic.mod
```


además de:


```
Calculando Armonicos ...
Seleccione el Armonico
armonico recto (Ramses)
armonico polar (Rose)
armonico circular (Kanji)
Clavicula Armonica
```


Esto significa que **no solamente podemos intentar recuperar código**.


Podemos intentar reconstruir una parte significativa de la **arquitectura original del proyecto ARMÓN**:


```
ARMON.EXE
│
├── formularios
│   ├── TFORM1
│   ├── TFORM2
│   ├── TFORM3
│   └── TFORM4
│
├── componentes
│   ├── TGPSELECTOR
│   ├── TGEODE
│   ├── TPROGRESO
│   ├── TMLISTA
│   └── ...
│
├── eventos
│   ├── ArmonicoChange
│   ├── AjustarArmonicas1Click
│   └── OpcionesparaFlorArmonica1Click
│
├── recursos/DFM
│
├── código Pascal compilado
│
└── datos/textos
    ├── Harmonic.txt
    └── armonic.mod
```


## Y aquí está la parte interesante


Hay herramientas históricas para Delphi que pueden reconstruir **DFM, información de formularios y estructuras Delphi**, aunque no puedan recuperar literalmente el `.PAS` original. La literatura sobre recuperación de proyectos Delphi distingue precisamente entre reconstruir formularios/estructura y recuperar los cuerpos originales de los procedimientos.


Además, existe investigación/herramientas actuales que reconocen específicamente **Borland Delphi 1.0 sobre NE de 16 bits**, lo que confirma que este formato puede analizarse específicamente en lugar de tratarlo como un PE moderno.


### Mi recomendación


**No toquemos todavía el `ARMON.EXE`.**


El siguiente paso debería ser **extraer primero todos los recursos Delphi/DFM del NE**, porque eso puede darnos los nombres, propiedades, jerarquía y eventos de los formularios prácticamente en su forma de diseño original.


Después atacaríamos el código máquina de Delphi 1.


Y hay una ventaja adicional: **ahora que sabemos que es Delphi 1, podemos buscar específicamente herramientas/formato de Delphi 1 en lugar de utilizar herramientas genéricas de PE.**


Si quieres, en el siguiente paso podemos hacerlo **exclusivamente desde Termux**, empezando por identificar y extraer los recursos DFM de este `ARMON.EXE`, sin modificar el ejecutable original.

## Usuario

Identificar y extraer los recursos DFM de este ARMON.EXE desde Termux

## Asistente

Sí. Y hay que hacer una precisión: **`wrestool` puede leer recursos de ejecutables NE de 16 bits**, pero no identifica necesariamente los recursos Delphi como “DFM”; los recursos arbitrarios pueden salir como datos crudos.


Para tu `ARMON.EXE`, propongo este procedimiento, conservando intacto el original.


### 1. Comprueba si tienes `wrestool`


```
Bashwhich wrestool
```


Si no existe, intenta:


```
Bashpkg install icoutils
```


`icoutils` incluye `wrestool`, y el proyecto está diseñado para ejecutarse en sistemas POSIX.


Comprueba:


```
Bashwrestool --version
```


### 2. Lista todos los recursos del ARMON original


```
Bashwrestool -l /storage/emulated/0/Download/armon/ARMON.EXE
```


**No uses todavía `-x`**. Primero queremos ver qué tipos de recursos contiene.


Guarda además la salida:


```
Bashwrestool -l /storage/emulated/0/Download/armon/ARMON.EXE \
> ~/armon_resources.txt
```


Y:


```
Bashcat ~/armon_resources.txt
```


### 3. Busca específicamente recursos `RCDATA`


Delphi puede almacenar datos de formularios como recursos personalizados. `wrestool` identifica `RCDATA` como tipo 10 y permite extraerlos en formato bruto.


Prueba:


```
Bashwrestool -l -t10 /storage/emulated/0/Download/armon/ARMON.EXE
```


Si aparecen recursos, **no los extraigas todavía individualmente**. Primero quiero ver sus nombres, tamaños y offsets.


### 4. Buscar directamente la firma de DFM


Hay una pista particularmente buena: los formularios binarios de Delphi utilizan normalmente el encabezado:


```
TPF0
```


Por tanto podemos buscarlo directamente dentro de `ARMON.EXE`:


```
Bashgrep -abo 'TPF0' /storage/emulated/0/Download/armon/ARMON.EXE
```


Y también:


```
Bashxxd -g 1 /storage/emulated/0/Download/armon/ARMON.EXE | grep '54 50 46 30'
```


`54 50 46 30` corresponde a:


```
TPF0
```


### 5. Buscar todos los posibles formularios Delphi


Tu `strings` ya nos mostró:


```
TFORM1
TFORM2
TFORM3
TFORM4
TABOUTBOX
TGPSELECTOR
TGEODE
...
```


Podemos localizar sus posiciones exactas:


```
Bashgrep -abo 'TFORM1' /storage/emulated/0/Download/armon/ARMON.EXE
grep -abo 'TFORM2' /storage/emulated/0/Download/armon/ARMON.EXE
grep -abo 'TFORM3' /storage/emulated/0/Download/armon/ARMON.EXE
grep -abo 'TFORM4' /storage/emulated/0/Download/armon/ARMON.EXE
grep -abo 'TABOUTBOX' /storage/emulated/0/Download/armon/ARMON.EXE
```


Esto nos permitirá correlacionar las cadenas con los recursos.


---


### Importante: no recomiendo usar todavía `DfmExtractor`


Encontré `DfmExtractor`, que está específicamente diseñado para extraer DFM de ejecutables Delphi, pero su documentación indica que se construye con Lazarus/Free Pascal y utiliza `JclPeImage`, es decir, está orientado principalmente a ejecutables PE modernos.


**Nuestro archivo es NE/Delphi 1**, así que prefiero no asumir que esa herramienta podrá interpretar correctamente este formato.


Primero vamos a determinar cómo Delphi 1 almacenó los formularios en **este ejecutable concreto**.


### Haz solamente estos cuatro comandos


```
Bashpkg install icoutils
```


```
Bashwrestool -l /storage/emulated/0/Download/armon/ARMON.EXE
```


```
Bashwrestool -l -t10 /storage/emulated/0/Download/armon/ARMON.EXE
```


```
Bashgrep -abo 'TPF0' /storage/emulated/0/Download/armon/ARMON.EXE
```


**Pásame las cuatro salidas.** Con ellas podremos determinar si los DFM están como recursos estándar, `RCDATA` o embebidos de otra manera, y entonces te daré el comando exacto para extraerlos.

## Usuario

~ $ wrestool -l /storage/emulated/0/Download/armon/ARMON.EXE
--type=3 --name=1 [type=icon offset=0x2edfc0 size=768]
--type=14 --name='MAINICON' [type=group_icon offset=0x2ee2c0 size=256]
--type=10 --name='TMISGRAPH' [type=rcdata offset=0x2ee3c0 size=1536]
--type=10 --name='TFORM1' [type=rcdata offset=0x2ee9c0 size=40192]
--type=10 --name='TLISTAWS' [type=rcdata offset=0x2f86c0 size=10240]
--type=10 --name='TFORM2' [type=rcdata offset=0x2faec0 size=1280]
--type=10 --name='TGPSELECTOR' [type=rcdata offset=0x2fb3c0 size=1024]
--type=10 --name='TABOUTBOX' [type=rcdata offset=0x2fb7c0 size=2304]
--type=10 --name='TGEODE' [type=rcdata offset=0x2fc0c0 size=7680]
--type=10 --name='TENTRETENER' [type=rcdata offset=0x2fdec0 size=512]
--type=10 --name='TFORM3' [type=rcdata offset=0x2fe0c0 size=2048]
--type=10 --name='TAUXMENU' [type=rcdata offset=0x2fe8c0 size=768]
--type=10 --name='TPROGRESO' [type=rcdata offset=0x2febc0 size=512]
--type=10 --name='TFORM4' [type=rcdata offset=0x2fedc0 size=768]
--type=10 --name='TMLISTA' [type=rcdata offset=0x2ff0c0 size=1024]
--type=2 --name='SPINDOWN' [type=bitmap offset=0x2ff4c0 size=256]
--type=2 --name='SPINUP' [type=bitmap offset=0x2ff5c0 size=256]
--type=2 --name='BBHELP' [type=bitmap offset=0x2ff6c0 size=512]
--type=2 --name='BBNO' [type=bitmap offset=0x2ff8c0 size=512]
--type=2 --name='BBOK' [type=bitmap offset=0x2ffac0 size=512]
--type=2 --name='BBYES' [type=bitmap offset=0x2ffcc0 size=512]
--type=2 --name='BBCANCEL' [type=bitmap offset=0x2ffec0 size=512]
--type=2 --name='BBCLOSE' [type=bitmap offset=0x3000c0 size=512]
--type=2 --name='BBRETRY' [type=bitmap offset=0x3002c0 size=512]
--type=2 --name='BBALL' [type=bitmap offset=0x3004c0 size=512]
--type=2 --name='BBABORT' [type=bitmap offset=0x3006c0 size=512]
--type=2 --name='BBIGNORE' [type=bitmap offset=0x3008c0 size=512]
--type=1 --name=1 [type=cursor offset=0x300ac0 size=512]
--type=1 --name=2 [type=cursor offset=0x300dc0 size=512]
--type=1 --name=3 [type=cursor offset=0x3010c0 size=512]
--type=1 --name=4 [type=cursor offset=0x3013c0 size=512]
--type=1 --name=5 [type=cursor offset=0x3016c0 size=512]
--type=1 --name=6 [type=cursor offset=0x3019c0 size=512]
--type=12 --name=32763 [type=group_cursor offset=0x300cc0 size=256]
--type=12 --name=32764 [type=group_cursor offset=0x300fc0 size=256]
--type=12 --name=32765 [type=group_cursor offset=0x3012c0 size=256]
--type=12 --name=32766 [type=group_cursor offset=0x3015c0 size=256]
--type=12 --name=32767 [type=group_cursor offset=0x3018c0 size=256]
--type=12 --name=32762 [type=group_cursor offset=0x301bc0 size=256]
--type=6 --name=3841 [type=string offset=0x301cc0 size=512]
--type=6 --name=3842 [type=string offset=0x301ec0 size=512]
--type=6 --name=3843 [type=string offset=0x3020c0 size=512]
--type=6 --name=3844 [type=string offset=0x3022c0 size=512]
--type=6 --name=3845 [type=string offset=0x3024c0 size=512]
--type=6 --name=3846 [type=string offset=0x3026c0 size=512]
--type=6 --name=3847 [type=string offset=0x3028c0 size=512]
--type=6 --name=3848 [type=string offset=0x302ac0 size=256]
--type=6 --name=3849 [type=string offset=0x302bc0 size=256]
--type=6 --name=3850 [type=string offset=0x302cc0 size=512]
--type=6 --name=3851 [type=string offset=0x302ec0 size=512]
--type=6 --name=4089 [type=string offset=0x3030c0 size=512]
--type=6 --name=4090 [type=string offset=0x3032c0 size=512]
--type=6 --name=4091 [type=string offset=0x3034c0 size=256]
--type=6 --name=4093 [type=string offset=0x3035c0 size=256]
--type=6 --name=4094 [type=string offset=0x3036c0 size=256]
--type=6 --name=4095 [type=string offset=0x3037c0 size=256]

## Asistente

Excelente. **Esto confirma que encontramos los recursos internos de Delphi 1**. Y, sobre todo, `wrestool` ya nos da exactamente los límites de cada formulario.


Los recursos clave son:


| Recurso | Tamaño | Interpretación |
| --- | --- | --- |
| `TMISGRAPH` | 1.536 B | Recurso gráfico/datos |
| `TFORM1` | **40.192 B** | Formulario principal o muy grande |
| `TLISTAWS` | 10.240 B | Lista/componente |
| `TFORM2` | 1.280 B | Formulario |
| `TGPSELECTOR` | 1.024 B | Formulario/componente |
| `TABOUTBOX` | 2.304 B | About Box |
| `TGEODE` | 7.680 B | Formulario/componente |
| `TENTRETENER` | 512 B | Formulario/componente |
| `TFORM3` | 2.048 B | Formulario |
| `TAUXMENU` | 768 B | Menú auxiliar |
| `TPROGRESO` | 512 B | Formulario/progreso |
| `TFORM4` | 768 B | Formulario |
| `TMLISTA` | 1.024 B | Lista |


La parte especialmente importante es esta:


```
--type=10 --name='TFORM1'      offset=0x2ee9c0 size=40192
--type=10 --name='TFORM2'      offset=0x2faec0 size=1280
--type=10 --name='TGPSELECTOR' offset=0x2fb3c0 size=1024
--type=10 --name='TABOUTBOX'   offset=0x2fb7c0 size=2304
...
```


`type=10` significa **RCDATA**, precisamente donde este ejecutable está almacenando los datos de los formularios/componentes. `wrestool` permite extraer esos recursos directamente de ejecutables NE de Windows 3.x.


### Ahora no extraigamos todo todavía


Primero quiero determinar **cómo están codificados los formularios Delphi 1**.


Ejecuta:


```
Bashfor n in TFORM1 TFORM2 TGPSELECTOR TABOUTBOX TGEODE TENTRETENER TFORM3 TAUXMENU TPROGRESO TFORM4 TMLISTA; do
    echo "===== $n ====="
    wrestool -x -t10 -n "$n" /storage/emulated/0/Download/armon/ARMON.EXE | xxd -l 128
done
```


Esto nos mostrará los primeros 128 bytes de cada RCDATA.


Pero hay una prueba todavía más importante. Ejecuta:


```
Bashgrep -abo 'TPF0' /storage/emulated/0/Download/armon/ARMON.EXE
```


y también:


```
Bashgrep -abo 'TPF1' /storage/emulated/0/Download/armon/ARMON.EXE
```


### ¿Qué estamos buscando?


Si aparece algo como:


```
xxxxxxx:TPF0
```


habremos encontrado directamente la **firma binaria del formulario Delphi**.


Entonces podremos hacer algo mucho más interesante:


```
ARMON.EXE
   │
   ├── TFORM1
   │     ├── Caption
   │     ├── Width
   │     ├── Height
   │     ├── componentes
   │     ├── nombres
   │     ├── propiedades
   │     └── eventos
   │
   ├── TFORM2
   ├── TGPSELECTOR
   ├── TABOUTBOX
   ├── TGEODE
   ├── ...
```


Y potencialmente reconstruir una representación equivalente a los antiguos **DFM de Delphi**, aunque **no necesariamente recuperar el `.pas` original**.


De hecho, el recurso de `TFORM1` de **40 KB** es especialmente prometedor: probablemente contiene una cantidad considerable de la estructura visual y propiedades del formulario principal.


**Pásame la salida de esos dos comandos (`TPF0` y `TPF1`) primero.** Con eso podemos determinar exactamente cómo proceder antes de modificar o extraer nada del EXE.

## Usuario

~ $ grep -abo 'TPF0' /storage/emulated/0/Download/armon/ARMON.EXE
227088:TPF0
3072960:TPF0
3074496:TPF0
3114688:TPF0
3124928:TPF0
3126208:TPF0
3127232:TPF0
3129536:TPF0
3137216:TPF0
3137728:TPF0
3139776:TPF0
3140544:TPF0
3141056:TPF0
3141824:TPF0
~ $ grep -abo 'TPF1' /storage/emulated/0/Download/armon/ARMON.EXE

## Asistente

Perfecto. Esto confirma algo muy importante: **los formularios Delphi están realmente presentes en el EXE en formato `TPF0`**.


Los offsets relevantes son:


```
227088
3072960
3074496
3114688
3124928
3126208
3127232
3129536
3137216
3137728
3139776
3140544
3141056
3141824
```


Y no existe `TPF1`, así que estamos ante el formato `TPF0`.


### Lo más interesante


Los primeros offsets:


```
3072960
3074496
3114688
3124928
...
```


están prácticamente dentro de la zona que `wrestool` identificó como recursos:


```
TFORM1       0x2ee9c0 = 3074496
TLISTAWS     0x2f86c0 = 3114688
TFORM2       0x2faec0 = 3124928
TGPSELECTOR  0x2fb3c0 = 3126208
TABOUTBOX    0x2fb7c0 = 3127232
TGEODE       0x2fc0c0 = 3131584
...
```


Es decir, **los `TPF0` están coincidiendo con los límites de los recursos Delphi que acabamos de descubrir**.


Ahora podemos pasar de la simple identificación a la **extracción de los formularios**.


### 1. Extrae directamente todos los RCDATA


Primero crea una carpeta:


```
Bashmkdir -p ~/armon_forms
```


Después:


```
Bashfor n in TMISGRAPH TFORM1 TLISTAWS TFORM2 TGPSELECTOR TABOUTBOX TGEODE TENTRETENER TFORM3 TAUXMENU TPROGRESO TFORM4 TMLISTA; do
    wrestool -x -t10 -n "$n" /storage/emulated/0/Download/armon/ARMON.EXE > ~/armon_forms/$n.bin
done
```


Comprueba:


```
Bashls -lh ~/armon_forms
```


Deberíamos obtener aproximadamente:


```
TFORM1       40K
TLISTAWS     10K
TFORM2       1.3K
TGPSELECTOR  1.0K
TABOUTBOX    2.3K
...
```


### 2. Comprueba cuáles contienen `TPF0`


```
Bashfor f in ~/armon_forms/*.bin; do
    printf "%-30s " "$(basename "$f")"
    grep -abo 'TPF0' "$f"
done
```


Esto es especialmente útil porque ahora **los offsets empiezan desde cero dentro de cada recurso**, en lugar de desde el EXE completo.


### 3. Y hagamos una prueba con `TFORM1`


Ejecuta:


```
Bashxxd -l 256 ~/armon_forms/TFORM1.bin
```


y:


```
Bashstrings -a ~/armon_forms/TFORM1.bin | head -100
```


Si `TFORM1` empieza aproximadamente así:


```
TPF0....
```


entonces ya tenemos el recurso Delphi aislado.


El siguiente paso será **decodificar `TPF0`**, no simplemente tratarlo como texto. Ahí podremos empezar a recuperar cosas como:


```
TForm
  Caption
  Width
  Height
  Left
  Top
  ...
  TButton
    Caption
    Name
    ...
```


y, especialmente importante para ARMÓN, **los nombres de componentes y asociaciones de eventos**.


Esto nos acerca mucho más a reconstruir la estructura original de la aplicación que cualquier desensamblado genérico del EXE.

## Usuario

for f in ~/armon_forms/*.bin; do
    printf "%-30s " "$(basename "$f")"
    grep -abo 'TPF0' "$f"
done~ $ for f in ~/armon_forms/*.bin; do
>     printf "%-30s " "$(basename "$f")"
>     grep -abo 'TPF0' "$f"
> done
TABOUTBOX.bin                  TAUXMENU.bin                   TENTRETENER.bin                TFORM1.bin                     TFORM2.bin                     TFORM3.bin                     TFORM4.bin                     TGEODE.bin                     TGPSELECTOR.bin                TLISTAWS.bin                   TMISGRAPH.bin                  TMLISTA.bin                    TPROGRESO.bin                  ~ $ bash ~/rotar_sesion.sh

## Asistente

Sí. Esa salida indica que **ninguno de los `.bin` extraídos contiene literalmente `TPF0`**.


Eso es importante porque los `TPF0` que encontramos antes están **dentro de los recursos**, pero aparentemente `wrestool -x -t10` está extrayendo una representación que no conserva esos bytes al principio del recurso, o los recursos tienen una estructura adicional.


Además, tu bucle quedó correcto; simplemente `grep` no encontró coincidencias y por eso solo imprimió los nombres.


Ahora no hagamos más extracciones. Vamos a comparar **los bytes originales del EXE con los bytes que `wrestool` extrajo**.


Ejecuta exactamente:


```
Bashxxd -l 64 ~/armon_forms/TFORM1.bin
```


y:


```
Bashxxd -l 64 /storage/emulated/0/Download/armon/ARMON.EXE
```


Después ejecuta:


```
Bashgrep -abo 'TPF0' ~/armon_forms/TFORM1.bin
```


Y, finalmente, esta comparación será decisiva:


```
Bashwc -c ~/armon_forms/TFORM1.bin
```


Debería dar aproximadamente:


```
40192
```


porque `wrestool` informó:


```
TFORM1 size=40192
```


### Hay una pista muy fuerte


El primer `TPF0` que encontramos fue:


```
227088:TPF0
```


pero **no corresponde a ninguno de los recursos RCDATA que listó `wrestool`**, que empiezan alrededor de:


```
0x2ee3c0 = 3070912
```


Por tanto, ese primer `TPF0` probablemente pertenece a **otra estructura del ejecutable**, posiblemente código/datos Delphi, y no a un formulario.


Los demás sí están en la zona de recursos.


Así que el siguiente objetivo es averiguar exactamente qué hace `wrestool` con uno de estos recursos. **No borres ni modifiques ningún archivo todavía.**


Pásame solamente la salida de:


```
Bashxxd -l 64 ~/armon_forms/TFORM1.bin
grep -abo 'TPF0' ~/armon_forms/TFORM1.bin
wc -c ~/armon_forms/TFORM1.bin
```


Con esos tres datos podemos determinar el formato real que está entregando `wrestool` y continuar con la recuperación de los formularios.

## Usuario

Dame todos los comandos que debo ejecutar en una sola caja

## Asistente

Claro. Ejecuta **todo este bloque de una sola vez**:


```
Bashecho "===== 1. TAMAÑOS DE LOS RECURSOS EXTRAÍDOS ====="
ls -lh ~/armon_forms/*.bin

echo
echo "===== 2. PRIMEROS 64 BYTES DE TFORM1 ====="
xxd -l 64 ~/armon_forms/TFORM1.bin

echo
echo "===== 3. BUSCAR TPF0 DENTRO DE TFORM1 ====="
grep -abo 'TPF0' ~/armon_forms/TFORM1.bin

echo
echo "===== 4. TAMAÑO EXACTO DE TFORM1 ====="
wc -c ~/armon_forms/TFORM1.bin

echo
echo "===== 5. PRIMEROS 64 BYTES DEL EXE ====="
xxd -l 64 /storage/emulated/0/Download/armon/ARMON.EXE

echo
echo "===== 6. TODOS LOS TPF0 DEL EXE ====="
grep -abo 'TPF0' /storage/emulated/0/Download/armon/ARMON.EXE

echo
echo "===== 7. TPF0 EN CADA RECURSO EXTRAÍDO ====="
for f in ~/armon_forms/*.bin; do
    printf "%-25s " "$(basename "$f")"
    grep -abo 'TPF0' "$f" || echo "sin TPF0"
done

echo
echo "===== 8. STRINGS DE TFORM1 ====="
strings -a ~/armon_forms/TFORM1.bin | head -100
```


**No modifiques ni borres nada** después de ejecutarlo. Pásame toda la salida y con ella determinamos cómo están empaquetados los formularios Delphi `TPF0`.

## Usuario

~ $ echo "===== 1. TAMAÑOS DE LOS RECURSOS EXTRÍDOS ===
=="
===== 1. TAMAÑOS DE LOS RECURSOS EXTRAÍDOS =====
~ $ ls -lh ~/armon_forms/*.bin
n

echo
echo "===== 4. TAMAÑO EXACTO DE TFORM1 ====="
wc -c ~/armon_forms/TFORM1.bin

echo
echo "===== 5. PRIMEROS 64 BYTES DEL EXE ====="
xxd -l 64 /storage/emulated/0/Download/armon/ARMON.EXE

echo
echo "===== 6. TODOS LOS TPF0 DEL EXE ====="
grep -abo 'TPF0' /storage/emulated/0/Download/armon/ARMON.EXE

echo
echo "===== 7. TPF0 EN CADA RECURSO EXTRAÍDO ====="
for f in ~/armon_forms/*.bin; do
    printf "%-25s " "$(basename "$f")"
    grep -abo 'TPF0' "$f" || echo "sin TPF0"
done

echo
echo "===== 8. STRINGS DE TFORM1 ====="
strings -a ~/armon_forms/TFORM1.bin | head -100-rw-------. 1 u0_a197 u0_a197 0 Sep 26 22:22 /data/data/com.termux/files/home/armon_forms/TABOUTBOX.bin
-rw-------. 1 u0_a197 u0_a197 0 Sep 26 22:22 /data/data/com.termux/files/home/armon_forms/TAUXMENU.bin
-rw-------. 1 u0_a197 u0_a197 0 Sep 26 22:22 /data/data/com.termux/files/home/armon_forms/TENTRETENER.bin
-rw-------. 1 u0_a197 u0_a197 0 Sep 26 22:22 /data/data/com.termux/files/home/armon_forms/TFORM1.bin
-rw-------. 1 u0_a197 u0_a197 0 Sep 26 22:22 /data/data/com.termux/files/home/armon_forms/TFORM2.bin
-rw-------. 1 u0_a197 u0_a197 0 Sep 26 22:22 /data/data/com.termux/files/home/armon_forms/TFORM3.bin
-rw-------. 1 u0_a197 u0_a197 0 Sep 26 22:22 /data/data/com.termux/files/home/armon_forms/TFORM4.bin
-rw-------. 1 u0_a197 u0_a197 0 Sep 26 22:22 /data/data/com.termux/files/home/armon_forms/TGEODE.bin
-rw-------. 1 u0_a197 u0_a197 0 Sep 26 22:22 /data/data/com.termux/files/home/armon_forms/TGPSELECTOR.bin
-rw-------. 1 u0_a197 u0_a197 0 Sep 26 22:22 /data/data/com.termux/files/home/armon_forms/TLISTAWS.bin
-rw-------. 1 u0_a197 u0_a197 0 Sep 26 22:22 /data/data/com.termux/files/home/armon_forms/TMISGRAPH.bin
-rw-------. 1 u0_a197 u0_a197 0 Sep 26 22:22 /data/data/com.termux/files/home/armon_forms/TMLISTA.bin
-rw-------. 1 u0_a197 u0_a197 0 Sep 26 22:22 /data/data/com.termux/files/home/armon_forms/TPROGRESO.bin
~ $
~ $ echo

~ $ echo "===== 2. PRIMEROS 64 BYTES DE TFORM1 ====="
===== 2. PRIMEROS 64 BYTES DE TFORM1 =====
~ $ xxd -l 64 ~/armon_forms/TFORM1.bin
~ $
~ $ echo

~ $ echo "===== 3. BUSCAR TPF0 DENTRO DE TFORM1 ====="
===== 3. BUSCAR TPF0 DENTRO DE TFORM1 =====
~ $ grep -abo 'TPF0' ~/armon_forms/TFORM1.bin
~ $
~ $ echo

~ $ echo "===== 4. TAMAÑO EXACTO DE TFORM1 ====="
===== 4. TAMAÑO EXACTO DE TFORM1 =====
~ $ wc -c ~/armon_forms/TFORM1.bin
0 /data/data/com.termux/files/home/armon_forms/TFORM1.bin
~ $
~ $ echo

~ $ echo "===== 5. PRIMEROS 64 BYTES DEL EXE ====="
===== 5. PRIMEROS 64 BYTES DEL EXE =====
~ $ xxd -l 64 /storage/emulated/0/Download/armon/ARMON.EXE
00000000: 4d5a 0001 0100 0000 0800 1000 ffff 0800  MZ..............
00000010: 0001 0000 0000 0000 4000 0000 0000 0000  ........@.......
00000020: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000030: 0000 0000 0000 0000 0000 0000 0001 0000  ................
~ $
~ $ echo

~ $ echo "===== 6. TODOS LOS TPF0 DEL EXE ====="
===== 6. TODOS LOS TPF0 DEL EXE =====
~ $ grep -abo 'TPF0' /storage/emulated/0/Download/armon/ARMON.EXE
227088:TPF0
3072960:TPF0
3074496:TPF0
3114688:TPF0
3124928:TPF0
3126208:TPF0
3127232:TPF0
3129536:TPF0
3137216:TPF0
3137728:TPF0
3139776:TPF0
3140544:TPF0
3141056:TPF0
3141824:TPF0
~ $
~ $ echo

~ $ echo "===== 7. TPF0 EN CADA RECURSO EXTRAÍDO ====="
===== 7. TPF0 EN CADA RECURSO EXTRAÍDO =====
~ $ for f in ~/armon_forms/*.bin; do
>     printf "%-25s " "$(basename "$f")"
>     grep -abo 'TPF0' "$f" || echo "sin TPF0"
> done
TABOUTBOX.bin             sin TPF0
TAUXMENU.bin              sin TPF0
TENTRETENER.bin           sin TPF0
TFORM1.bin                sin TPF0
TFORM2.bin                sin TPF0
TFORM3.bin                sin TPF0
TFORM4.bin                sin TPF0
TGEODE.bin                sin TPF0
TGPSELECTOR.bin           sin TPF0
TLISTAWS.bin              sin TPF0
TMISGRAPH.bin             sin TPF0
TMLISTA.bin               sin TPF0
TPROGRESO.bin             sin TPF0
~ $
~ $ echo

~ $ echo "===== 8. STRINGS DE TFORM1 ====="
===== 8. STRINGS DE TFORM1 =====
~ $ strings -a ~/armon_forms/TFORM1.bin | head -100

## Asistente

Esto aclara el problema: **`wrestool` no extrajo los recursos a stdout con ese comando**. Los archivos `.bin` quedaron creados pero de **0 bytes**, así que no debemos interpretar todavía nada sobre el formato Delphi.


Los `TPF0` originales siguen intactos dentro de `ARMON.EXE`.


Vamos a corregir la extracción usando la opción de salida de `wrestool`. Ejecuta este bloque completo:


```
Bashecho "===== 1. AYUDA DE WRESTOOL ====="
wrestool --help

echo
echo "===== 2. ELIMINAR SOLO LOS BIN VACÍOS ====="
find ~/armon_forms -type f -name '*.bin' -size 0 -delete

echo
echo "===== 3. EXTRAER TFORM1 ====="
cd ~/armon_forms
wrestool -x -t10 -n TFORM1 /storage/emulated/0/Download/armon/ARMON.EXE

echo
echo "===== 4. CONTENIDO GENERADO ====="
ls -lah ~/armon_forms

echo
echo "===== 5. BUSCAR TPF0 EN LOS ARCHIVOS EXTRAÍDOS ====="
for f in ~/armon_forms/*; do
    [ -f "$f" ] || continue
    printf "%-35s " "$(basename "$f")"
    grep -abo 'TPF0' "$f" || echo "sin TPF0"
done

echo
echo "===== 6. TODOS LOS TPF0 DEL EXE ====="
grep -abo 'TPF0' /storage/emulated/0/Download/armon/ARMON.EXE
```


**Importante:** no vuelvas a usar por ahora:


```
Bashwrestool ... > archivo.bin
```


porque en este caso estamos obteniendo archivos vacíos.


Después de ver la salida de `wrestool --help`, podremos usar exactamente su mecanismo de extracción correcto. Los offsets que ya encontramos (`3072960`, `3074496`, etc.) siguen siendo válidos y no hemos alterado el EXE.

## Usuario

~ $ echo "===== 1. AYUDA DE WRESTOOL ====="
===== 1. AYUDA DE WRESTOOL =====
~ $ wrestool --help
ind ~/armon_forms -type f -name '*.bin' -size 0 -delete

echo
echo "===== 3. EXTRAER TFORM1 ====="
cd ~/armon_forms
wrestool -x -t10 -n TFORM1 /storage/emulated/0/Download/armon/ARMON.EXE

echo
echo "===== 4. CONTENIDO GENERADO ====="
ls -lah ~/armon_forms

echo
echo "===== 5. BUSCAR TPF0 EN LOS ARCHIVOS EXTRAÍDOS ====="
for f in ~/armon_forms/*; do
    [ -f "$f" ] || continue
    printf "%-35s " "$(basename "$f")"
    grep -abo 'TPF0' "$f" || echo "sin TPF0"
done

echo
echo "===== 6. TODOS LOS TPF0 DEL EXE ====="
grep -abo 'TPF0' /storage/emulated/0/Download/armon/ARMON.EXEUsage: wrestool [OPTION]... [FILE]...
Extract resources from Microsoft Windows(R) binaries.

Commands:
  -x, --extract           extract resources
  -l, --list              output list of resources (default)

Filters:
  -t, --type=[+|-]ID      resource type identifier
  -n, --name=[+|-]ID      resource name identifier
  -L, --language=[+|-]ID  resource language identifier
  -a, --all               perform operation on all resource (default)

Miscellaneous:
  -o, --output=PATH       where to place extracted files
  -R, --raw               do not parse resource contents
  -v, --verbose           explain what is being done
      --help              display this help and exit
      --version           output version information and exit

A leading `+' in --type, name or language options indicates a true string
`identifier. Similarly, `-' indicates a true numeric identifier.
`
Report bugs to frank.richter@gmail.com.
~ $
~ $ echo

~ $ echo "===== 2. ELIMINAR SOLO LOS BIN VACÍOS ====="
===== 2. ELIMINAR SOLO LOS BIN VACÍOS =====
~ $ find ~/armon_forms -type f -name '*.bin' -size 0 -delete
~ $
~ $ echo

~ $ echo "===== 3. EXTRAER TFORM1 ====="
===== 3. EXTRAER TFORM1 =====
~ $ cd ~/armon_forms
~/armon_forms $ wrestool -x -t10 -n TFORM1 /storage/emulated/0/Download/armon/ARMON.EXE
wrestool: /storage/emulated/0/Download/armon/ARMON.EXE: don't know how to extract resource, try `--raw'
`~/armon_forms $
~/armon_forms $ echo
                                                       "
===== 4. CONTENIDO GENERADO =====NTENIDO GENERADO ====="
~/armon_forms $ ls -lah ~/armon_forms
total 16K
drwx------.  2 u0_a197 u0_a197 4.0K Sep 26 22:25 .
drwx------. 19 u0_a197 u0_a197  12K Sep 26 22:25 ..
~/armon_forms $
~/armon_forms $ echo

~/armon_forms $ echo "===== 5. BUSCAR TPF0 EN LOS ARCHIVOS EXTRAÍDOS ====="
===== 5. BUSCAR TPF0 EN LOS ARCHIVOS EXTRAÍDOS =====
~/armon_forms $ for f in ~/armon_forms/*; do
>     [ -f "$f" ] || continue
>     printf "%-35s " "$(basename "$f")"
>     grep -abo 'TPF0' "$f" || echo "sin TPF0"
> done
~/armon_forms $
~/armon_forms $ echo

~/armon_forms $ echo "===== 6. TODOS LOS TPF0 DEL EXE ====="
===== 6. TODOS LOS TPF0 DEL EXE =====
~/armon_forms $ grep -abo 'TPF0' /storage/emulated/0/Download/armon/ARMON.EXE
227088:TPF0
3072960:TPF0
3074496:TPF0
3114688:TPF0
3124928:TPF0
3126208:TPF0
3127232:TPF0
3129536:TPF0
3137216:TPF0
3137728:TPF0
3139776:TPF0
3140544:TPF0
3141056:TPF0
3141824:TPF0

## Asistente

Perfecto. Ahora tenemos la explicación exacta: `wrestool` **sí reconoce los recursos**, pero no sabe interpretar esos `RCDATA` de Delphi 1. El propio programa nos indica la solución:


```
don't know how to extract resource, try `--raw'
```


Por tanto, debemos usar `-R`. Eso además es lo que queremos: **extraer los bytes originales sin que `wrestool` intente interpretar el recurso**.


Ejecuta este bloque completo:


```
Bashcd ~

rm -rf ~/armon_forms
mkdir -p ~/armon_forms

echo "===== EXTRAER TFORM1 EN RAW ====="
wrestool -x -R -t10 -n TFORM1 \
  -o ~/armon_forms \
  /storage/emulated/0/Download/armon/ARMON.EXE

echo
echo "===== ARCHIVOS EXTRAÍDOS ====="
ls -lah ~/armon_forms

echo
echo "===== TPF0 EN TFORM1 ====="
for f in ~/armon_forms/*; do
    echo "--- $(basename "$f") ---"
    wc -c "$f"
    grep -abo 'TPF0' "$f" || echo "sin TPF0"
done

echo
echo "===== PRIMEROS 128 BYTES ====="
for f in ~/armon_forms/*; do
    echo "--- $(basename "$f") ---"
    xxd -l 128 "$f"
done
```


### Qué esperamos


Esta vez **no deberíamos obtener un archivo de 0 bytes**.


Y si `wrestool` extrae correctamente `TFORM1`, deberíamos ver algo cercano a:


```
TPF0
```


dentro del archivo.


Además, hay un detalle muy interesante en tus offsets:


```
3072960:TPF0
3074496:TPF0
3114688:TPF0
3124928:TPF0
...
```


Los recursos empiezan exactamente en:


```
TMISGRAPH   0x2ee3c0 = 3070912
TFORM1      0x2ee9c0 = 3074496
TLISTAWS    0x2f86c0 = 3114688
TFORM2      0x2faec0 = 3124928
```


Por tanto, **los `TPF0` están asociados directamente con los recursos Delphi**.


Una vez que tengamos `TFORM1` en bruto, podemos pasar a la parte realmente interesante: **interpretar la estructura binaria `TPF0` y convertirla en una representación legible de los formularios de ARMÓN**.

## Usuario

~ $ cd ~
~ $
~ $ rm -rf ~/armon_forms
AW ====="
wrestool -x -R -t10 -n TFORM1 \
  -o ~/armon_forms \
  /storage/emulated/0/Download/armon/ARMON.EXE

echo
echo "===== ARCHIVOS EXTRAÍDOS ====="
ls -lah ~/armon_forms

echo
echo "===== TPF0 EN TFORM1 ====="
for f in ~/armon_forms/*; do
    echo "--- $(basename "$f") ---"
    wc -c "$f"
    grep -abo 'TPF0' "$f" || echo "sin TPF0"
done

echo
echo "===== PRIMEROS 128 BYTES ====="
for f in ~/armon_forms/*; do
    echo "--- $(basename "$f") ---"
    xxd -l 128 "$f"
done~ $ mkdir -p ~/armon_forms
~ $
~ $ echo "===== EXTRAER TFORM1 EN RAW ====="
===== EXTRAER TFORM1 EN RAW =====
~ $ wrestool -x -R -t10 -n TFORM1 \
>   -o ~/armon_forms \
>   /storage/emulated/0/Download/armon/ARMON.EXE
~ $
~ $ echo

~ $ echo "===== ARCHIVOS EXTRAÍDOS ====="
===== ARCHIVOS EXTRAÍDOS =====
~ $ ls -lah ~/armon_forms
total 56K
drwx------.  2 u0_a197 u0_a197 4.0K Sep 26 22:26 .
drwx------. 19 u0_a197 u0_a197  12K Sep 26 22:26 ..
-rw-------.  1 u0_a197 u0_a197  40K Sep 26 22:26 ARMON.EXE_10_TFORM1
~ $
~ $ echo

~ $ echo "===== TPF0 EN TFORM1 ====="
===== TPF0 EN TFORM1 =====
~ $ for f in ~/armon_forms/*; do
>     echo "--- $(basename "$f") ---"
>     wc -c "$f"
>     grep -abo 'TPF0' "$f" || echo "sin TPF0"
> done
--- ARMON.EXE_10_TFORM1 ---
40192 /data/data/com.termux/files/home/armon_forms/ARMON.EXE_10_TFORM1
0:TPF0
~ $
~ $ echo

~ $ echo "===== PRIMEROS 128 BYTES ====="
===== PRIMEROS 128 BYTES =====
~ $ for f in ~/armon_forms/*; do
>     echo "--- $(basename "$f") ---"
>     xxd -l 128 "$f"
> done
--- ARMON.EXE_10_TFORM1 ---
00000000: 5450 4630 0654 466f 726d 3105 466f 726d  TPF0.TForm1.Form
00000010: 3104 4c65 6674 0201 0354 6f70 0262 0557  1.Left...Top.b.W
00000020: 6964 7468 03b5 0206 4865 6967 6874 030e  idth....Height..
00000030: 0204 4869 6e74 060b 4d65 6efa 2072 e170  ..Hint..Men. r.p
00000040: 6964 6f07 4361 7074 696f 6e06 1345 7870  ido.Caption..Exp
00000050: 6c6f 7261 646f 7220 4172 6df3 6e69 636f  lorador Arm.nico
00000060: 0a46 6f6e 742e 436f 6c6f 7207 0c63 6c57  .Font.Color..clW
00000070: 696e 646f 7754 6578 740b 466f 6e74 2e48  indowText.Font.H

## Asistente

Perfecto. **Ahora sí tenemos el formulario Delphi real y completo.** El archivo extraído mide exactamente `40192` bytes, coincide con el tamaño indicado por `wrestool`, y comienza con:


```
TPF0
```


Además, ya podemos leer directamente propiedades Delphi.


Por ejemplo, los primeros bytes se interpretan claramente como:


```
TForm1
Form1
Left
Top
Width
Height
Hint
Caption
Font.Color
Font.Height
...
```


Y especialmente:


```
Caption = "Explorador Armónico"
```


Esto demuestra que podemos reconstruir la estructura visual del programa.


### Ahora vamos a extraer TODOS los formularios correctamente


Ejecuta este único bloque:


```
Bashcd ~

rm -rf ~/armon_forms
mkdir -p ~/armon_forms

EXE=/storage/emulated/0/Download/armon/ARMON.EXE

echo "===== EXTRAYENDO RECURSOS DELPHI ====="

for n in TMISGRAPH TFORM1 TLISTAWS TFORM2 TGPSELECTOR TABOUTBOX TGEODE TENTRETENER TFORM3 TAUXMENU TPROGRESO TFORM4 TMLISTA; do
    echo "Extrayendo $n ..."
    wrestool -x -R -t10 -n "$n" -o ~/armon_forms "$EXE"
done

echo
echo "===== ARCHIVOS ====="
ls -lh ~/armon_forms

echo
echo "===== TAMAÑOS ====="
for f in ~/armon_forms/*; do
    printf "%-35s " "$(basename "$f")"
    wc -c < "$f"
done

echo
echo "===== CABECERA TPF0 ====="
for f in ~/armon_forms/*; do
    printf "%-35s " "$(basename "$f")"
    xxd -l 4 "$f"
done

echo
echo "===== STRINGS ASCII DE LOS FORMULARIOS ====="
for f in ~/armon_forms/*; do
    echo
    echo "========== $(basename "$f") =========="
    strings -a "$f"
done
```


### Qué conseguiremos


Esto nos permitirá obtener, entre otras cosas:


```
TFORM1
TFORM2
TFORM3
TFORM4
TABOUTBOX
TGEODE
TGPSELECTOR
TENTRETENER
TPROGRESO
TMLISTA
TLISTAWS
TAUXMENU
```


y sus propiedades:


```
Caption
Left
Top
Width
Height
Font
Color
Name
Visible
Enabled
...
```


Pero hay algo todavía más importante: **los componentes hijos de cada formulario**.


Por ejemplo, dentro de `TFORM1` probablemente encontraremos estructuras como:


```
TButton
TLabel
TPanel
TMenu
TEdit
TListBox
TImage
...
```


junto con sus nombres y captions.


Y más adelante podemos buscar específicamente:


```
OnClick
OnChange
OnCreate
OnClose
OnPaint
```


porque esos nombres de eventos pueden permitir relacionar la interfaz con las funciones que todavía existen en el código máquina.


**Este es ya el camino correcto para reconstruir ARMÓN:**


```
ARMON.EXE
   │
   ├── código máquina
   │
   ├── TFORM1 ──→ estructura visual
   ├── TFORM2 ──→ estructura visual
   ├── TFORM3 ──→ estructura visual
   ├── TFORM4 ──→ estructura visual
   ├── TABOUTBOX
   ├── TGEODE
   ├── TGPSELECTOR
   └── ...
          │
          ▼
   propiedades + componentes
          │
          ▼
   eventos Delphi
          │
          ▼
   código máquina asociado
```


No estamos recuperando todavía el `.PAS` original, pero **ya estamos recuperando información de muy alto nivel que el compilador dejó dentro del EXE**.

## Usuario

~ $ cd ~
~ $
~ $ rm -rf ~/armon_forms
/armon_forms

EXE=/storage/emulated/0/Download/armon/ARMON.EXE

echo "===== EXTRAYENDO RECURSOS DELPHI ====="

for n in TMISGRAPH TFORM1 TLISTAWS TFORM2 TGPSELECTOR TABOUTBOX TGEODE TENTRETENER TFORM3 TAUXMENU TPROGRESO TFORM4 TMLISTA; do
    echo "Extrayendo $n ..."
    wrestool -x -R -t10 -n "$n" -o ~/armon_forms "$EXE"
done

echo
echo "===== ARCHIVOS ====="
ls -lh ~/armon_forms

echo
echo "===== TAMAÑOS ====="
for f in ~/armon_forms/*; do
    printf "%-35s " "$(basename "$f")"
    wc -c < "$f"
done

echo
echo "===== CABECERA TPF0 ====="
for f in ~/armon_forms/*; do
    printf "%-35s " "$(basename "$f")"
    xxd -l 4 "$f"
done

echo
echo "===== STRINGS ASCII DE LOS FORMULARIOS ====="
for f in ~/armon_forms/*; do
    echo
    echo "========== $(basename "$f") =========="
    strings -a "$f"
done~ $ mkdir -p ~/armon_forms
~ $
~ $ EXE=/storage/emulated/0/Download/armon/ARMON.EXE
~ $
~ $ echo "===== EXTRAYENDO RECURSOS DELPHI ====="
===== EXTRAYENDO RECURSOS DELPHI =====
~ $
~ $ for n in TMISGRAPH TFORM1 TLISTAWS TFORM2 TGPSELECTOR TABOUTBOX TGEODE TENTRETENER TFORM3 TAUXMENU TPROGRESO TFORM4 TMLISTA; do
>     echo "Extrayendo $n ..."
>     wrestool -x -R -t10 -n "$n" -o ~/armon_forms "$EXE"
> done
Extrayendo TMISGRAPH ...
Extrayendo TFORM1 ...
Extrayendo TLISTAWS ...
Extrayendo TFORM2 ...
Extrayendo TGPSELECTOR ...
Extrayendo TABOUTBOX ...
Extrayendo TGEODE ...
Extrayendo TENTRETENER ...
Extrayendo TFORM3 ...
Extrayendo TAUXMENU ...
Extrayendo TPROGRESO ...
Extrayendo TFORM4 ...
Extrayendo TMLISTA ...
~ $
~ $ echo

~ $ echo "===== ARCHIVOS ====="
===== ARCHIVOS =====
~ $ ls -lh ~/armon_forms
total 100K
-rw-------. 1 u0_a197 u0_a197 2.3K Sep 26 22:26 ARMON.EXE_10_TABOUTBOX
-rw-------. 1 u0_a197 u0_a197  768 Sep 26 22:26 ARMON.EXE_10_TAUXMENU
-rw-------. 1 u0_a197 u0_a197  512 Sep 26 22:26 ARMON.EXE_10_TENTRETENER
-rw-------. 1 u0_a197 u0_a197  40K Sep 26 22:26 ARMON.EXE_10_TFORM1
-rw-------. 1 u0_a197 u0_a197 1.3K Sep 26 22:26 ARMON.EXE_10_TFORM2
-rw-------. 1 u0_a197 u0_a197 2.0K Sep 26 22:26 ARMON.EXE_10_TFORM3
-rw-------. 1 u0_a197 u0_a197  768 Sep 26 22:26 ARMON.EXE_10_TFORM4
-rw-------. 1 u0_a197 u0_a197 7.5K Sep 26 22:26 ARMON.EXE_10_TGEODE
-rw-------. 1 u0_a197 u0_a197 1.0K Sep 26 22:26 ARMON.EXE_10_TGPSELECTOR
-rw-------. 1 u0_a197 u0_a197  10K Sep 26 22:26 ARMON.EXE_10_TLISTAWS
-rw-------. 1 u0_a197 u0_a197 1.5K Sep 26 22:26 ARMON.EXE_10_TMISGRAPH
-rw-------. 1 u0_a197 u0_a197 1.0K Sep 26 22:26 ARMON.EXE_10_TMLISTA
-rw-------. 1 u0_a197 u0_a197  512 Sep 26 22:26 ARMON.EXE_10_TPROGRESO
~ $
~ $ echo

~ $ echo "===== TAMAÑOS ====="
===== TAMAÑOS =====
~ $ for f in ~/armon_forms/*; do
>     printf "%-35s " "$(basename "$f")"
>     wc -c < "$f"
> done
ARMON.EXE_10_TABOUTBOX              2304
ARMON.EXE_10_TAUXMENU               768
ARMON.EXE_10_TENTRETENER            512
ARMON.EXE_10_TFORM1                 40192
ARMON.EXE_10_TFORM2                 1280
ARMON.EXE_10_TFORM3                 2048
ARMON.EXE_10_TFORM4                 768
ARMON.EXE_10_TGEODE                 7680
ARMON.EXE_10_TGPSELECTOR            1024
ARMON.EXE_10_TLISTAWS               10240
ARMON.EXE_10_TMISGRAPH              1536
ARMON.EXE_10_TMLISTA                1024
ARMON.EXE_10_TPROGRESO              512
~ $
~ $ echo

~ $ echo "===== CABECERA TPF0 ====="
===== CABECERA TPF0 =====
~ $ for f in ~/armon_forms/*; do
>     printf "%-35s " "$(basename "$f")"
>     xxd -l 4 "$f"
> done
ARMON.EXE_10_TABOUTBOX              00000000: 5450 4630                                TPF0
ARMON.EXE_10_TAUXMENU               00000000: 5450 4630                                TPF0
ARMON.EXE_10_TENTRETENER            00000000: 5450 4630                                TPF0
ARMON.EXE_10_TFORM1                 00000000: 5450 4630                                TPF0
ARMON.EXE_10_TFORM2                 00000000: 5450 4630                                TPF0
ARMON.EXE_10_TFORM3                 00000000: 5450 4630                                TPF0
ARMON.EXE_10_TFORM4                 00000000: 5450 4630                                TPF0
ARMON.EXE_10_TGEODE                 00000000: 5450 4630                                TPF0
ARMON.EXE_10_TGPSELECTOR            00000000: 5450 4630                                TPF0
ARMON.EXE_10_TLISTAWS               00000000: 5450 4630                                TPF0
ARMON.EXE_10_TMISGRAPH              00000000: 5450 4630                                TPF0
ARMON.EXE_10_TMLISTA                00000000: 5450 4630                                TPF0
ARMON.EXE_10_TPROGRESO              00000000: 5450 4630                                TPF0
~ $
~ $ echo

~ $ echo "===== STRINGS ASCII DE LOS FORMULARIOS ====="
===== STRINGS ASCII DE LOS FORMULARIOS =====
~ $ for f in ~/armon_forms/*; do
>     echo
>     echo "========== $(basename "$f") =========="
>     strings -a "$f"
> done

========== ARMON.EXE_10_TABOUTBOX ==========
TPF0    TAboutBox
AboutBox
Left
ActiveControl
OKButton
BorderStyle
bsDialog
Caption
Autor
ClientHeight
ClientWidth
Font.Color
clWindowText
Font.Height
        Font.Name
System
Font.Style
PixelsPerInch
Position
poScreenCenter
TextHeight
TPanel
Panel1
Left
Width
Height
BevelInner
bvRaised
BevelOuter
        bvLowered
TabOrder
TImage
ProgramIcon
Left
Width
Height
Picture.Data
TBitmapv
xxwx
wwwwwwp7
wwww
Stretch         IsControl
TLabel
ProductName
Left
Width
Height
Caption
Explorador Arm
nico
Font.Color
clBlack
Font.Height
        Font.Name
MS Sans Serif
Font.Style
fsBold
ParentFont
        IsControl
TLabel
Version
Left
Width
Height
Caption
Version 1.0
Font.Color
clBlack
Font.Height
        Font.Name
MS Sans Serif
Font.Style
fsBold
ParentFont
        IsControl
TLabel  Copyright
Left
Width
Height
Caption
(Copyright: Miguel Garc
a Ferr
ndez, 1997
Font.Color
clBlack
Font.Height
        Font.Name
MS Sans Serif
Font.Style
fsBold
ParentFont
        IsControl
TLabel
Comments
Left
Width
Height
Caption
C/ Doctor Nieto, 42  (8
 Iz)
Font.Color
clBlack
Font.Height
        Font.Name
MS Sans Serif
Font.Style
fsBold
ParentFont
        IsControl
TLabel
Label1
Left
Width
Height
Caption
03013 ALICANTE
TLabel
Label2
Left
Width
Height
Caption
ESPA
A (SPAIN)
TLabel
Label3
Left
Width
Height
Caption
E-mail:  Miguel.Garcia@ua.es
TBitBtn
OKButton
Left
Width
Height
Font.Color
clBlack
Font.Height
        Font.Name
MS Sans Serif
Font.Style
fsBold
ParentFont
TabOrder
Kind
bkOK
Margin
Spacing
        IsControl

========== ARMON.EXE_10_TAUXMENU ==========
TPF0
TAuxMenu
AuxMenu
Left
Width
Height
Caption
AuxMenu
Font.Color
clWindowText
Font.Height
        Font.Name
System
Font.Style
PixelsPerInch
Position
poScreenCenter
OnActivate
FormActivate
TextHeight
TRadioGroup
pull
Left
Width
Height
Align
alClient
Caption
pull
TabOrder
OnClick
        pullClick
TPanel
PDatos
Left
Width
Height
Align
alBottom
TabOrder
TEdit
Datos
Left
Width
Height
Hint
Orbe para Aspectos
ParentShowHint
ShowHint
TabOrder
Text
Auto
Visible

========== ARMON.EXE_10_TENTRETENER ==========
TPF0
TEntretener
Entretener
Left
Width
Height
Caption
Entretener
Font.Color
clWindowText
Font.Height
        Font.Name
System
Font.Style
PixelsPerInch
TextHeight
TGauge
LogGauge
Left
Width
Height
Progress
TMemo
LogMemo
Left
Width
Height
Lines.Strings
LogMemo
ReadOnly
TabOrder

========== ARMON.EXE_10_TFORM1 ==========
TPF0
TForm1
Form1
Left
Width
Height
Hint
pido
Caption
Explorador Arm
nico
Font.Color
clWindowText
Font.Height
        Font.Name
System
Font.Style
Menu
        MainMenu1
PixelsPerInch
Position
poScreenCenter
WindowState
wsMaximized
OnActivate
FormResize
OnCreate
FormCreate
TextHeight
        TGroupBox
DatosHarmo
Left
Width
Height
Caption
mica Arm
nica
TabOrder
TButton
DHGoHarm
Left
Width
Height
Hint
Iniciar Trazado
Caption
OK Calcular
ParentShowHint
ShowHint
TabOrder
OnClick
DHGoHarmClick
TStringGrid
DHWhat
Left
Width
Height
Hint
9<fecha>, MES, HOY, SECUNDARIA (+/-<numero de dias/meses>)
Align
alBottom
ColCount
DefaultColWidth
DefaultRowHeight
        FixedCols
        FixedRows
Options
goFixedVertLine
goFixedHorzLine
goVertLine
goHorzLine
goColSizing     goEditing
goAlwaysShowEditor
goThumbTracking
ParentShowHint
RowCount
ShowHint
TabOrder
        ColWidths
RowHeights
TEdit
DHModelo
Left
Width
Height
Hint
9Modelo de Din
mica Arm
nica (use el bot
n para cambiarlo)
Ctl3D
ParentCtl3D
ParentShowHint
ReadOnly
ShowHint
TabOrder
Text
Modelo
TButton
DHMOtro
Left
Width
Height
Hint
Cargar otro Modelo del disco
Caption
Otro
ParentShowHint
ShowHint
TabOrder
OnClick
DHMOtroClick
TButton
DHMSave
Left
Width
Height
Hint
Guardar este Modelo en disco
Caption
Salvar
ParentShowHint
ShowHint
TabOrder
OnClick
DHMSaveClick
TEdit
DHComentario
Left
Width
Height
Hint
1Cualquier comantario que se archiva con los datos
Font.Color
clWindowText
Font.Height
        Font.Name
System
Font.Style
ParentFont
ParentShowHint
ShowHint
TabOrder
Text
Comentario
TButton
InsCol
Left
Width
Height
Caption
InsCol
TabOrder
OnClick
InsColClick
TButton
SupCol
Left
Width
Height
Caption
SupCol
TabOrder
OnClick
SupColClick
        TCheckBox
Cuadrado
Left
Width
Height
Hint
Activar para dibujo cuadrado
Caption
Cuadrado
ParentShowHint
ShowHint
TabOrder
TPanel
SoportaImagen
Left
Width
Height
Caption
SoportaImagen
TabOrder
TImage
Imagen
Left
Width
Height
Align
alClient
OnDblClick
DiseodePgina1Click
        TGroupBox
DatosNatales
Left
Width
Height
Caption
Datos Natales
TabOrder
TEdit
Fecha
Left
Width
Height
Hint
Fecha de Nacimiento
ParentShowHint
ShowHint
TabOrder
Text
27-11-1952
OnChange
DatosChanged
TSpinButton
SBDia
Left
Width
Height
Hint
Mover el D
DownGlyph.Data
ParentShowHint
ShowHint
TabOrder
UpGlyph.Data
OnDownClick
SBDiaDownClick  OnUpClick
SBDiaUpClick
TSpinButton
SBMes
Left
Width
Height
Hint
Mover el Mes
DownGlyph.Data
ParentShowHint
ShowHint
TabOrder
UpGlyph.Data
OnDownClick
SBMesDownClick  OnUpClick
SBMesUpClick
TSpinButton
SBAnno
Left
Width
Height
Hint
Mover el A
DownGlyph.Data
ParentShowHint
ShowHint
TabOrder
UpGlyph.Data
OnDownClick
SBAnnoDownClick OnUpClick
SBAnnoUpClick
TSpinButton
SBDecada
Left
Width
Height
Hint
Mover la Decada
DownGlyph.Data
ParentShowHint
ShowHint
TabOrder
UpGlyph.Data
OnDownClick
SBDecadaDownClick       OnUpClick
SBDecadaUpClick
TSpinButton
SBCent
Left
Width
Height
Hint
Mover el Siglo
DownGlyph.Data
ParentShowHint
ShowHint
TabOrder
UpGlyph.Data
OnDownClick
SBCentDownClick OnUpClick
SBCentUpClick
TEdit
Hora
Left
Width
Height
Hint
Hora de Nacimiento
ParentShowHint
ShowHint
TabOrder
Text
17:10:00
OnChange
DatosChanged
TSpinButton
SBHora
Left
Width
Height
Hint
Mover la Hora
DownGlyph.Data
ParentShowHint
ShowHint
TabOrder
UpGlyph.Data
OnDownClick
SBHoraDownClick OnUpClick
SBHoraUpClick
TSpinButton
SBMin
Left
Width
Height
Hint
Mover los Minutos
DownGlyph.Data
ParentShowHint
ShowHint
TabOrder
UpGlyph.Data
OnDownClick
SBMinDownClick  OnUpClick
SBMinUpClick
TSpinButton
SBSeg
Left
Width
Height
Hint
#Avanzar/Retroceder segun Incremento
DownGlyph.Data
ParentShowHint
ShowHint
TabOrder
UpGlyph.Data
OnDownClick
SBSegDownClick  OnUpClick
SBSegUpClick
TEdit
DifHGMT
Left
Width
Height
Hint
)Diferencia con el Meridiano de GreenWhich
ParentShowHint
ShowHint
TabOrder
Text
-1:00:00
OnChange
DatosChanged
OnDblClick
DifHGMTDblClick
TEdit
Latitud
Left
Width
Height
Hint
Latitud Geogr
fica
ParentShowHint
ShowHint
TabOrder
Text
38:06 N
OnChange
DatosChanged
OnDblClick
LugarArchivo
TEdit
Longitud
Left
Width
Height
Hint
Longitud Geogr
fica
ParentShowHint
ShowHint
TabOrder
Text
0:57 W
OnChange
DatosChanged
OnDblClick
LugarArchivo
        TCheckBox
UseHelio
Left
Width
Height
Hint
Cambiar Coordenadas
TabStop
Caption
&Helio
ParentShowHint
ShowHint
TabOrder
OnClick
UseHelioClick
        TCheckBox
UseSider
Left
Width
Height
Hint
Cambiar Zodiaco
TabStop
Caption
&Sider
ParentShowHint
ShowHint
TabOrder
OnClick
UseSiderClick
TButton
Calcular
Left
Width
Height
Hint
!... con los datos que se muestran
Caption
        &Calcular
ParentShowHint
ShowHint
TabOrder
TabStop
OnClick
CalcularClick
TButton
Ahora
Left
Width
Height
Hint
Fecha y Hora actual
Caption
ParentShowHint
ShowHint
TabOrder
TabStop
OnClick
AhoraClick
TEdit
Nombre
Left
Width
Height
Hint
Nombre
ParentShowHint
ShowHint
TabOrder
Text
Miguel Garc
OnDblClick
NombreDblClick
OnKeyUp
NombreKeyUp
TEdit
Lugar
Left
Width
Height
Hint
Lugar de Nacimiento
ParentShowHint
ShowHint
TabOrder
Text
Orihuela
OnChange
DatosChanged
OnDblClick
LugarDblClick
OnKeyUp
LugarKeyUp
TButton
Buscar
Left
Width
Height
Hint
Buscar otros datos
Caption
&Buscar
ParentShowHint
ShowHint
TabOrder
TabStop
OnClick
        BuscarClk
        TComboBox
Zodiaco
Left
Width
Height
Hint
Dise
o de la Rueda
ItemHeight
Items.Strings
        5,2,3/M,1
4,1,2/1
5,6,3/1,M,H,1
4,6,3/1,2,3,4,5,6,7
ParentShowHint
ShowHint
TabOrder
Text
        5,2,3/M,1
OnChange
ZodiacoChange
OnExit
ZodiacoChange
        TComboBox
Aspectos
Left
Width
Height
Hint
Modelo de Aspectos
ItemHeight
Items.Strings
clasico
moderno
x7,x8,x9
7,8,9
ParentShowHint
ShowHint
TabOrder
Text
clasico
OnExit
ZodiacoChange
        TCheckBox
UseAR
Left
Width
Height
TabStop
Caption
        A. &Recta
TabOrder
OnClick
UseARClick
TEdit
Armonico
Left
Width
Height
Hint
nico Global
ParentShowHint
ShowHint
TabOrder
Text
OnChange
ArmonicoChange
TSpinButton
SpinButton1
Left
Width
Height
Hint
Aumentar/Disminuir el Arm
nico
DownGlyph.Data
ParentShowHint
ShowHint
TabOrder
UpGlyph.Data
OnDownClick
SpinButton1DownClick    OnUpClick
SpinButton1UpClick
TEdit
EdadActual
Left
Width
Height
Hint
*Edad para Direcciones o fecha (dd-mm-aaaa)
ParentShowHint
ShowHint
TabOrder
Text
OnChange
EdadActualChange
OnDblClick
EdadActualDblClick
TButton
EstiloProg
Left
Width
Height
Hint
AEstilo de Progresi
n (Ninguno,Truncado,Redondeo,Simb
lico,eXacto)
Caption
ParentShowHint
ShowHint
TabOrder
OnClick
EstiloProgClick
TEdit
OndaArm
Left
Width
Height
Hint
Subarm
nico en banda 'h'
ParentShowHint
ShowHint
TabOrder
Text
OnChange
OndaArmChange
TButton
AbrirLista
Left
Width
Height
Hint
#Abrir la lista de cartas en memoria
Caption
ParentShowHint
ShowHint
TabOrder
TabStop
OnClick
NombreDblClick
TButton
ListaLugares
Left
Width
Height
Hint
"Lista de localidades y coordenadas
Caption
ParentShowHint
ShowHint
TabOrder
TabStop
OnClick
LugarArchivo
TSpinButton
SpinButton2
Left
Width
Height
Hint
Incremento por a
DownGlyph.Data
ParentShowHint
ShowHint
TabOrder
UpGlyph.Data
OnDownClick
SpinButton2DownClick    OnUpClick
SpinButton2UpClick
TButton
UpDownBy
Left
Width
Height
Hint
Incremento +/-
Caption
ParentShowHint
ShowHint
TabOrder
OnClick
UpDownByClick
        TCheckBox       UseACIMUT
Left
Width
Height
TabStop
Caption
Ac&imut
TabOrder
OnClick
UseACIMUTClick
        TCheckBox
UseDOM
Left
Width
Height
TabStop
Caption
D&omal
TabOrder
OnClick
UseDOMClick
TButton
casas
Left
Width
Height
Hint
Sistema de Casas
Caption
casas
ParentShowHint
ShowHint
TabOrder
OnClick
casasClick
TSpinButton
MoveWave
Left
Width
Height
Hint
Aumentar/Disminuir Subarmonico
DownGlyph.Data
ParentShowHint
ShowHint
TabOrder
UpGlyph.Data
OnDownClick
MoveWaveDownClick       OnUpClick
MoveWaveUpClick
TSpinButton
SpinLat
Left
Width
Height
Hint
Variar la Latitud
DownGlyph.Data
ParentShowHint
ShowHint
TabOrder
UpGlyph.Data
OnDownClick
SpinLatDownClick        OnUpClick
SpinLatUpClick
TSpinButton
SpinLon
Left
Width
Height
Hint
Variar la Longitud
DownGlyph.Data
ParentShowHint
ShowHint
TabOrder
UpGlyph.Data
OnDownClick
SpinLonDownClick        OnUpClick
SpinLonUpClick
TButton
sinonoff
Left
Width
Height
Hint
        Sinastria
Caption
ParentShowHint
ShowHint
TabOrder
OnClick
Sinastra1Click
TButton
flipper
Left
Width
Height
Hint
Alternar Cartas
Caption
ParentShowHint
ShowHint
TabOrder
OnClick
AlternarCartas1Click
TButton
RSLugar
Left
Width
Height
Hint
Lugar para Cumplea
Caption
ParentShowHint
ShowHint
TabOrder
TabStop
OnClick
RSLugarClick
TButton
FineSelector
Left
Width
Height
Hint
Dise
o de P
gina
Caption
ParentShowHint
ShowHint
TabOrder
TabStop
OnClick
DiseodePgina1Click
TSpinButton
SpinButton3
Left
Width
Height
Hint
Incremento por d
DownGlyph.Data
ParentShowHint
ShowHint
TabOrder
UpGlyph.Data
OnDownClick
SpinButton3DownClick    OnUpClick
SpinButton3UpClick
TSpinButton
SpinButton4
Left
Width
Height
Hint
Incremento por meses lunares
DownGlyph.Data
ParentShowHint
ShowHint
TabOrder
UpGlyph.Data
OnDownClick
SpinButton4DownClick    OnUpClick
SpinButton4UpClick
TButton
MenuRapido
Left
Width
Height
Caption
TabOrder
OnClick
MenuRapidoClick
TEdit
DurCiclo
Left
Width
Height
Hint
$Duraci
n de Ciclos para Profecciones
ParentShowHint
ShowHint
TabOrder
Text
OnExit
DurCicloChange
        TComboBox
Estilos
Left
Width
Height
ItemHeight
TabOrder
Text
Estilos
Visible
        TMainMenu       MainMenu1
Left
        TMenuItem
Archivo1
Caption
&Archivo
        TMenuItem       Imprimir1
Caption
        Im&primir
OnClick
Imprimir1Click
        TMenuItem
Impresora1
Caption
&Impresora
OnClick
Impresora1Click
        TMenuItem
MrgenesdePgina2
Caption
rgenes de P
gina y Otros
OnClick
MrgenesdePgina1Click
        TMenuItem
PartesDefinidos2
Caption
Partes Definidos
OnClick
PartesDefinidos1Click
        TMenuItem
Caption
        TMenuItem
SuiteBergamasque1
Caption
S&uite Bergamasque
OnClick
SuiteBergamasque1Click
        TMenuItem
Restringir1
Caption
&Restringir
OnClick
Restringir1Click
        TMenuItem
ImprimirAutomtico1
Caption
Imprimir/Archivar Au&tom
tico
OnClick
ImprimirAutomtico1Click
        TMenuItem
Caption
        TMenuItem
ArchivarDibujo1
Caption
&Archivar Dibujo
OnClick
ArchivarDibujo1Click
        TMenuItem
Caption
        TMenuItem
Salir1
Caption
&Salir
OnClick
Salir1Click
        TMenuItem
Configurar1
Caption
Con&Figurar
        TMenuItem
DiseodePgina1
Caption
Di&se
o de P
gina
OnClick
DiseodePgina1Click
        TMenuItem
CalcularSiempre1
Caption
Calcular S&iempre al Cambiar
OnClick
CalcularSiempre1Click
        TMenuItem
Caption
        TMenuItem
NodosPlanetarios1
Caption
Nodos &Planetarios
OnClick
NodosPlanetarios1Click
        TMenuItem
PlanetasqueseDibujan1
Caption
Planetas que se &Dibujan
OnClick
PlanetasqueseDibujan1Click
        TMenuItem
PlanetasqueseNumeran1
Caption
Planetas que se N&umeran
OnClick
PlanetasqueseNumeran1Click
        TMenuItem
PlanetasqueseAspectan1
Caption
Planetas que &Reciben Aspectos
OnClick
PlanetasqueseAspectan1Click
        TMenuItem
PlanetasqueEmitenAspectos1
Caption
Planetas que &Emiten Aspectos
OnClick
PlanetasqueEmitenAspectos1Click
        TMenuItem
PartesDefinidos1
Caption
Partes Definidos
OnClick
PartesDefinidos1Click
        TMenuItem
DinmicaArmnica1
Caption
Dominios &Arm
nicos
OnClick
DinmicaArmnica1Click
        TMenuItem
ArmnicosActivos1
Caption
nicos Acti&vos
OnClick
ArmnicosActivos1Click
        TMenuItem
AjustarArmonicas1
Caption
Escala para &Ondas Armonicas
OnClick
AjustarArmonicas1Click
        TMenuItem
OpcionesparaFlorArmonica1
Caption
%Opciones para la Flor Armonica (p&wf)
OnClick
OpcionesparaFlorArmonica1Click
        TMenuItem
Caption
        TMenuItem
PeriododeTiempoparatrnsitos1
Caption
!Periodo de &Tiempo para tr
nsitos
OnClick
!PeriododeTiempoparatrnsitos1Click
        TMenuItem
Caption
        TMenuItem
PantallaenColor1
Caption
Pa&ntalla en Color
OnClick
PantallaenColor1Click
        TMenuItem
IndicadorparaGrises1
Caption
Indicador para &Grises
OnClick
IndicadorparaGrises1Click
        TMenuItem
GlifosenNegro1
Caption
Gli&fos en Negro
OnClick
GlifosenNegro1Click
        TMenuItem
ComposicindelaRueda1
Caption
&Composici
n de la Rueda
OnClick
ComposicindelaRueda1Click
        TMenuItem
MrgenesdePgina1
Caption
rgenes de P
gina y Otros
Hint
Fuentes y Estrellas
OnClick
MrgenesdePgina1Click
        TMenuItem
ElNegroes1
Caption
El Negro es (R,G,&B)
Hint
0Color para todas las l
neas sin color espec
fico
OnClick
ElNegroes1Click
        TMenuItem
Fuenteparadibujo1
Caption
Fuente para dibu&jo
OnClick
Fuenteparadibujo1Click
        TMenuItem       Graficos1
Caption
        &Gr
ficos
        TMenuItem
DiseodePgina2
Caption
&Dise
o de P
gina
OnClick
DiseodePgina1Click
        TMenuItem
Caption
        TMenuItem       Sinastra1
Caption
&Sinastr
OnClick
Sinastra1Click
        TMenuItem
AlternarCartas1
Caption
&Alternar Cartas
OnClick
AlternarCartas1Click
        TMenuItem
Caption
        TMenuItem
CartaRadical1
Caption
Carta &Radical
OnClick
CartaRadical1Click
        TMenuItem
Harmogramas1
Caption
&Harmogramas
OnClick
Harmogramas1Click
        TMenuItem
Caption
        TMenuItem
Estrellas1
Caption
&Estrellas (experimento)
OnClick
Estrellas1Click
        TMenuItem
OtrosGraficos1
Caption
&Otros Graficos  (experimento)
OnClick
OtrosGraficos1Click
        TMenuItem
Autor1
Caption
A&utor
        TMenuItem
MiguelGarcaFerrndez1
Caption
 Datos de Miguel Garc
a Ferr
ndez
Hint
pulse para mas datos
OnClick
MiguelGarcaFerrndez1Click
TPrintDialog
PrintDialog1
Left
TPrinterSetupDialog
PrinterSetupDialog1
Left
TSaveDialog
SaveDialog1
DefaultExt
Filter
"Archivos Gr
ficos de Windows|*.bmp
Title
Archivo para guardar Gr
fico
Left
TOpenDialog
OpenDialog1
InitialDir
c:\cpa
Left

========== ARMON.EXE_10_TFORM2 ==========
TPF0
TForm2
Form2
Left
Width
Height
Caption
Editor de Desplazamientos
Font.Color
clWindowText
Font.Height
        Font.Name
System
Font.Style
PixelsPerInch
OnCreate
FormCreate
TextHeight
TStringGrid
StringGrid1
Left
Width
Height
Align
alClient
ColCount
        FixedCols
        FixedRows
Options
goFixedVertLine
goFixedHorzLine
goVertLine
goHorzLine
goRangeSelect
goColSizing     goEditing
goTabs
goAlwaysShowEditor
RowCount
TabOrder
        ColWidths
RowHeights
TPanel
Panel1
Left
Width
Height
Align
alBottom
TabOrder
TButton
Left
Width
Height
Caption
ModalResult
ParentShowHint
ShowHint
TabOrder
TButton
Insertar
Left
Width
Height
Hint
Insertar Una Fila
Caption
Insertar
ParentShowHint
ShowHint
TabOrder
TButton
Borrar
Left
Width
Height
Hint
Borrar Fila Actual
Caption
Borrar
ParentShowHint
ShowHint
TabOrder
TButton
Cancelar
Left
Width
Height
Hint
No Instalada
Caption
Cancelar
ModalResult
ParentShowHint
ShowHint
TabOrder
TMemo
Memo1
Left
Width
Height
Align
alClient
Lines.Strings
Memo1
TabOrder
Visible

========== ARMON.EXE_10_TFORM3 ==========
TPF0
TForm3
Form3
Left
Width
Height
Caption
Form3
Font.Color
clWindowText
Font.Height
        Font.Name
System
Font.Style
PixelsPerInch
WindowState
wsMaximized
OnActivate
FormActivate
OnResize
FormResize
TextHeight
TLabel
Archivo
Left
Width
Height
Caption
Archivo
TLabel
Label1
Left
Width
Height
Caption
Te&xto
TEdit
buscartexto
Left
Width
Height
Hint
#Texto que debe figurar en los datos
ParentShowHint
ShowHint
TabOrder
Text
OnDblClick
buscartextoDblClick
OnKeyUp
buscartextoKeyUp
TStringGrid
buscarastro
Left
Width
Height
Hint
Haga Doble Click para ayuda
ColCount
DefaultColWidth
        FixedCols
Options
goFixedVertLine
goFixedHorzLine
goVertLine
goHorzLine
goColSizing     goEditing
goAlwaysShowEditor
goThumbTracking
ParentShowHint
RowCount
ScrollBars
ssHorizontal
ShowHint
TabOrder
OnDblClick
buscarastroDblClick     ColWidths
TBitBtn
Left
Width
Height
Hint
Caption
ModalResult
ParentShowHint
ShowHint
TabOrder
TabStop
TBitBtn
Cancelar
Left
Width
Height
Caption
        &Cancelar
ModalResult
TabOrder
TabStop
TBitBtn
Todos
Left
Width
Height
Hint
#Seleccionar todos los que encuentre
Caption
&Todos
ModalResult
ParentShowHint
ShowHint
TabOrder
TabStop
TMemo
encontrado
Left
Width
Height
TabStop
Lines.Strings
encontrado
TabOrder
TButton
negativo
Left
Width
Height
Caption
ModalResult
TabOrder
TabStop
TEdit
desplazamiento
Left
Width
Height
Hint
'Desplazamiento de la Fecha Natal (dias)
ParentShowHint
ShowHint
TabOrder
Text
TEdit
perturbacion
Left
Width
Height
Hint
%Perturbacion de la Fecha Natal (dias)
ParentShowHint
ShowHint
TabOrder
Text
TEdit
repeticiones
Left
Width
Height
Hint
Numero de Replicaciones
ParentShowHint
ShowHint
TabOrder
Text
TButton
Abuscar
Left
Width
Height
Caption
&Buscar
ModalResult
TabOrder

========== ARMON.EXE_10_TFORM4 ==========
TPF0
TForm4
Form4
Left
Width
Height
Caption
Form4
Font.Color
clWindowText
Font.Height
        Font.Name
System
Font.Style
PixelsPerInch
OnCreate
FormCreate
TextHeight
TImage
Dibujo
Left
Width
Height
Align
alClient
TBitBtn
Left
Width
Height
Hint
Visto
Caption
Font.Color
clBlack
Font.Height
        Font.Name
System
Font.Style
fsBold
ModalResult
ParentFont
ParentShowHint
ShowHint
TabOrder
OnClick
OKClick
TButton
Left
Width
Height
Hint
Imprimir
Caption
ParentShowHint
ShowHint
TabOrder
OnClick
PrClick
TPrintDialog
PrintDialog1
Left

========== ARMON.EXE_10_TGEODE ==========
TPF0
TGeode
Geode
Left
Width
Height
Caption
Archivo de Coordenadas
Font.Color
clWindowText
Font.Height
        Font.Name
System
Font.Style
PixelsPerInch
TextHeight
TPanel
Panel1
Left
Width
Height
BevelInner
        bvLowered
BorderStyle
bsSingle
Caption
Panel1
ParentShowHint
ShowHint
TabOrder
TEdit
Lugar
Left
Width
Height
Font.Color
clBlack
Font.Height
        Font.Name
Courier New
Font.Style
ParentFont
TabOrder
Text
Lugar
TEdit
Left
Width
Height
Hint
Latitud
Enabled
Font.Color
clBlack
Font.Height
        Font.Name
Courier New
Font.Style
ParentFont
TabOrder
Text
TEdit
Long
Left
Width
Height
Hint
Longitud
Enabled
Font.Color
clBlack
Font.Height
        Font.Name
Courier New
Font.Style
ParentFont
TabOrder
Text
Long
TEdit
Pais
Left
Width
Height
Hint
Pais
Enabled
Font.Color
clBlack
Font.Height
        Font.Name
Courier New
Font.Style
ParentFont
TabOrder
Text
Pais
TEdit
difHGMT
Left
Width
Height
Hint
Huso Horario
Enabled
Font.Color
clBlack
Font.Height
        Font.Name
Courier New
Font.Style
ParentFont
TabOrder
Text
difHGMT
TSpinButton
SpinButton1
Left
Width
Height
Hint
Avance de L
DownGlyph.Data
TabOrder
UpGlyph.Data
OnDownClick
SpinButton1DownClick    OnUpClick
SpinButton1UpClick
TSpinButton
SpinButton2
Left
Width
Height
Hint
Avance de P
gina
DownGlyph.Data
TabOrder
UpGlyph.Data
OnDownClick
SpinButton2DownClick    OnUpClick
SpinButton2UpClick
TButton
Left
Width
Height
Hint
Elegir la Ciudad actual
Caption
TabOrder
OnClick
ElegirEste
TButton
Cancelar
Left
Width
Height
Hint
No cambiar
Caption
Cancelar
TabOrder
OnClick
CancelarClick
TButton
Editar
Left
Width
Height
Hint
n no implementado
Caption
Editar
TabOrder
TButton
Insertar
Left
Width
Height
Hint
n no implementado
Caption
Insertar
TabOrder
TButton
Borrar
Left
Width
Height
Hint
n no implementado
Caption
Borrar
TabOrder
TEdit
Leader
Left
Width
Height
Hint
(Escriba las prmeras letras de una ciudad
Font.Color
clBlack
Font.Height
        Font.Name
Courier New
Font.Style
ParentFont
TabOrder
Text
Leader
OnChange
LeaderChange
TPanel
Panel2
Left
Width
Height
Align
alBottom
BevelInner
        bvLowered
BorderStyle
bsSingle
Caption
Panel2
TabOrder
TStringGrid
ListaGeo
Left
Width
Height
Hint
 Contenido del Archivo Geogr
fico
Align
alClient
DefaultRowHeight
        FixedCols
Font.Color
clBlack
Font.Height
        Font.Name
Courier New
Font.Style
ParentFont
ParentShowHint
RowCount
ScrollBars
ssHorizontal
ShowHint
TabOrder
OnClick
MostrarEste
OnDblClick
ElegirEste      ColWidths

========== ARMON.EXE_10_TGPSELECTOR ==========
TPF0
TGPSelector
GPSelector
Left
Width
Height
Caption
GPSelector
Font.Color
clWindowText
Font.Height
        Font.Name
System
Font.Style
PixelsPerInch
WindowState
wsMaximized
OnActivate
FormActivate
OnResize
FormActivate
TextHeight
TListBox
ListBox1
Left
Width
Height
Align
alTop
ExtendedSelect
Font.Color
clBlack
Font.Height
        Font.Name
Courier New
Font.Style
ItemHeight
ParentFont
TabOrder
OnClick
ListBox1Click
OnDblClick
ListBox1DblClick
TBitBtn
BitBtn1
Left
Width
Height
TabOrder
Kind
bkOK
TBitBtn
BitBtn2
Left
Width
Height
TabOrder
Kind
bkCancel
TButton
cambiarSCR
Left
Width
Height
Hint
Cambiar Archivo de Dibujos
Caption
Archivo
ParentShowHint
ShowHint
TabOrder
OnClick
cambiarSCRClick
TOpenDialog
OpenDialog1
Left

========== ARMON.EXE_10_TLISTAWS ==========
TPF0
TListaWS
ListaWS
Left
Width
Height
Caption
ListaWS
Font.Color
clWindowText
Font.Height
        Font.Name
System
Font.Style
PixelsPerInch
WindowState
wsMaximized
OnActivate
FormActivate
OnCreate
FormCreate
OnKeyUp
        FormKeyUp
OnResize
FormResize
TextHeight
TPanel
Dibujo
Left
Width
Height
BevelInner
        bvLowered
Caption
Dibujo
TabOrder
TImage
IDibujo
Left
Width
Height
Align
alLeft
TButton
DMode
Left
Width
Height
Hint
Cambiar modo de dibujo
Caption
ParentShowHint
ShowHint
TabOrder
OnClick
DModeClick
TPanel
browser
Left
Width
Height
Align
alLeft
BevelInner
        bvLowered
Caption
browser
TabOrder
TImage
IBrowser
Left
Width
Height
Align
alLeft
OnMouseDown
IBrowserMouseDown
TScrollBar
ScrollFocus
Left
Width
Height
Kind
sbVertical
LargeChange
TabOrder
OnScroll
ScrollFocusScroll
TButton
Resumen
Left
Width
Height
Hint
Resumen de las cartas marcadas
Caption
ParentShowHint
ShowHint
TabOrder
OnClick
ResumenClick
TButton
Buscar
Left
Width
Height
Hint
Buscar
Caption
ParentShowHint
ShowHint
TabOrder
OnClick
BuscarClick
TButton
Listar
Left
Width
Height
Hint
(Listar en impresora los nombres marcados
Caption
ParentShowHint
ShowHint
TabOrder
OnClick
ListarClick
TButton
Archivar
Left
Width
Height
Hint
Archivar Datos Natales Marcados
Caption
ParentShowHint
ShowHint
TabOrder
OnClick
ArchivarClick
TPanel
Sujeto
Left
Width
Height
BevelInner
        bvLowered
Caption
Sujeto
TabOrder
TSpinButton
SpinButton1
Left
Width
Height
Hint
Pulse para Cambiar de Carta
DownGlyph.Data
ParentShowHint
ShowHint
TabOrder
UpGlyph.Data
OnDownClick
SpinButton1DownClick    OnUpClick
SpinButton1UpClick
TSpinButton
SpinButton2
Left
Width
Height
Hint
Desplazar Pagina
DownGlyph.Data
ParentShowHint
ShowHint
TabOrder
UpGlyph.Data
OnDownClick
SpinButton2DownClick    OnUpClick
SpinButton2UpClick
TSpinButton
SpinButton3
Left
Width
Height
Hint
%Pulse para ir al Principio o al Final
DownGlyph.Data
ParentShowHint
ShowHint
TabOrder
UpGlyph.Data
OnDownClick
SpinButton3DownClick    OnUpClick
SpinButton3UpClick
TButton
Regresar
Left
Width
Height
Hint
(Pulse para Volver a la Secci
n Principal
Caption
        R&egresar
ModalResult
ParentShowHint
ShowHint
TabOrder
OnClick
RegresarClick
TMemo
MSujeto
Left
Width
Height
Font.Color
clWindowText
Font.Height
        Font.Name
MS LineDraw
Font.Style
Lines.Strings
MSujeto
ParentFont
TabOrder
WordWrap
TButton
Borrar
Left
Width
Height
Hint
!Suprimir todos entre Verde y Rojo
Caption
&Borrar
ParentShowHint
ShowHint
TabOrder
OnClick
BorrarClick
TButton
Marcar
Left
Width
Height
Hint
"Marcar entre los dos seleccionados
Caption
&Marcar
ParentShowHint
ShowHint
TabOrder
OnClick
MarcarClick
TPrintDialog
PrintDialog1
Left
TSaveDialog
Archivo
DefaultExt
Filter
8Datos Natales (*.dat)|*.DAT|Todos los archivos (*.*)|*.*
Left

========== ARMON.EXE_10_TMISGRAPH ==========
TPF0    TMisGraph
MisGraph
Left
Width
Height
Caption
Miscel
nea de Gr
ficos
Font.Color
clWindowText
Font.Height
        Font.Name
System
Font.Style
PixelsPerInch
OnCreate
FormCreate
OnResize
FormResize
TextHeight
TRadioGroup
Otros
Left
Width
Height
Caption
Elija uno y pulse Trazar
Ctl3D
Font.Color
clBlack
Font.Height
        Font.Name
Arial
Font.Style
        ItemIndex
Items.Strings
voronoi recto
voronoi polar
voronoi circular
armonico recto (Ramses)
armonico polar (Rose)
armonico circular (Kanji)
Clavicula Armonica
Test (Luz Verdadera)
Espiral
ParentCtl3D
ParentFont
TabOrder
OnClick
OtrosClick
TPanel
Panel1
Left
Width
Height
BevelInner
        bvLowered
BevelWidth
BorderStyle
bsSingle
TabOrder
TLabel
Label1
Left
Width
Height
        Alignment
taCenter
Caption
TLabel
Label2
Left
Width
Height
        Alignment
taCenter
Caption
TButton
Cancelar
Left
Width
Height
Caption
Cancelar
TabOrder
OnClick
CancelarClick
TButton
Avanti
Left
Width
Height
Caption
Trazar
TabOrder
OnClick
AvantiClick
        TCheckBox
encolor
Left
Width
Height
Caption
En Color
TabOrder
        TSpinEdit
SpinWY
Left
Width
Height
MaxValue
MinValue
TabOrder
Value
        TSpinEdit
SpinWX
Left
Width
Height
MaxValue
MinValue
TabOrder
Value

========== ARMON.EXE_10_TMLISTA ==========
TPF0
TMLista
MLista
Left
Width
Height
Caption
Font.Color
clWindowText
Font.Height
        Font.Name
System
Font.Style
PixelsPerInch
WindowState
wsMaximized
OnActivate
FormActivate
OnResize
FormActivate
TextHeight
TStringGrid
Lista
Left
Width
Height
Align
alTop
ColCount
        FixedCols
        FixedRows
Options
goFixedVertLine
goFixedHorzLine
goVertLine
goHorzLine
goRangeSelect   goEditing
TabOrder
        ColWidths
TBitBtn
Vale
Left
Width
Height
TabOrder
OnClick
        ValeClick
Kind
bkOK
TBitBtn
Cancelar
Left
Width
Height
TabOrder
OnClick
CancelarClick
Kind
bkCancel
TButton
Insertar
Left
Width
Height
Hint
Insertar una Fila en blanco
Caption
Insertar
ParentShowHint
ShowHint
TabOrder
Visible
OnClick
InsertarClick
TButton
Borrar
Left
Width
Height
Hint
#Borrar la Fila donde est
 el cursor
Caption
Borrar
ParentShowHint
ShowHint
TabOrder
OnClick
BorrarClick

========== ARMON.EXE_10_TPROGRESO ==========
TPF0    TProgreso
Progreso
Left
BorderIcons
BorderStyle
bsDialog
Caption
Calculando ...
ClientHeight
ClientWidth
Font.Color
clWindowText
Font.Height
        Font.Name
System
Font.Style
PixelsPerInch
OnActivate
FormActivate
OnClick
        FormClick
OnHide
FormHide
TextHeight
TGauge
Gauge1
Left
Width
Height
Progress

## Asistente

Esto es **mucho más valioso de lo que esperábamos**. Ya no estamos simplemente descompilando: hemos recuperado una parte sustancial de la **estructura original de la aplicación Delphi**.


### Lo que acabamos de recuperar


`TFORM1` contiene:


- nombre de clase: `TForm1`
- nombre del formulario: `Form1`
- título: **Explorador Armónico**
- eventos:
  - `FormResize`
  - `FormCreate`
- decenas de componentes
- nombres internos de esos componentes
- propiedades
- captions
- textos de ayuda
- menús completos
- **nombres de los métodos de eventos**


Por ejemplo:


```
TButton
Calcular

Caption
&Calcular

OnClick
CalcularClick
```


Eso significa que conocemos que existía un método:


```
pascalprocedure TForm1.CalcularClick(Sender: TObject);
```


aunque **el cuerpo del método no está dentro del recurso `TPF0`**.


---


## El hallazgo más importante


Tenemos una correspondencia extraordinariamente útil:


```
COMPONENTE                 EVENTO
────────────────────────────────────────
Calcular                   CalcularClick
Ahora                      AhoraClick
DHMOtro                    DHMOtroClick
DHMSave                    DHMSaveClick
InsCol                     InsColClick
SupCol                     SupColClick
UseHelio                   UseHelioClick
UseSider                   UseSiderClick
Buscar                     BuscarClk
Zodiaco                    ZodiacoChange
Aspectos                   ZodiacoChange
Armonico                   ArmonicoChange
SpinButton1                SpinButton1DownClick
                           SpinButton1UpClick
MoveWave                   MoveWaveDownClick
                           MoveWaveUpClick
SpinLat                    SpinLatDownClick
                           SpinLatUpClick
SpinLon                    SpinLonDownClick
                           SpinLonUpClick
Sinastria                  Sinastra1Click
flipper                    AlternarCartas1Click
```


Esto nos proporciona **nombres simbólicos del código original**.


Y tenemos todavía el código máquina del EXE.


Por tanto, ahora podemos intentar hacer:


```
TPF0
 │
 ├── componente
 │
 └── evento
       │
       ▼
nombre del procedimiento Pascal
       │
       ▼
código máquina
       │
       ▼
pseudocódigo Pascal/C
```


Ese es el camino para aproximarnos al **código fuente funcional**.


---


# Siguiente paso: extraer los nombres de eventos


No quiero que hagamos todavía ingeniería inversa del código máquina de todo el programa. Primero vamos a construir un inventario preciso de todos los procedimientos que aparecen en los formularios.


Ejecuta este bloque:


```
Bashcd ~/armon_forms

echo "===== EVENTOS DEL FORMULARIO PRINCIPAL ====="

strings -a ARMON.EXE_10_TFORM1 |
grep -E 'Click$|Change$|DblClick$|DownClick$|UpClick$|KeyUp$|Resize$|Create$|Activate$|Close$|Hide$|Paint$|Scroll$|Exit$' |
sort -u

echo
echo "===== EVENTOS DE TODOS LOS FORMULARIOS ====="

for f in *; do
    echo
    echo "========== $f =========="
    strings -a "$f" |
    grep -E 'Click$|Change$|DblClick$|DownClick$|UpClick$|KeyUp$|Resize$|Create$|Activate$|Close$|Hide$|Paint$|Scroll$|Exit$' |
    sort -u
done
```


Pero quiero además algo todavía más importante: **buscar esos nombres directamente dentro del EXE completo**.


Ejecuta también:


```
Bashecho
echo "===== PROCEDIMIENTOS EN EL EXE ====="

strings -a /storage/emulated/0/Download/armon/ARMON.EXE |
grep -E 'Click$|Change$|DblClick$|DownClick$|UpClick$|KeyUp$|Resize$|Create$|Activate$|Close$|Hide$|Paint$|Scroll$|Exit$' |
sort -u
```


### ¿Por qué hacemos esto?


Porque podemos obtener una tabla como:


```
CalcularClick
AhoraClick
FormCreate
FormResize
DatosChanged
UseHelioClick
UseSiderClick
ArmonicoChange
...
```


y después buscar **dónde aparecen físicamente dentro del EXE**.


Por ejemplo:


```
Bashgrep -abo 'CalcularClick' /storage/emulated/0/Download/armon/ARMON.EXE
```


Eso nos da el offset de la cadena.


Después podemos investigar las **referencias desde el código máquina hacia esa cadena**.


---


## Y hay otro dato extraordinario


El `TFORM1` contiene datos reales del autor:


```
Explorador Armónico
Version 1.0
Miguel García Ferrández, 1997
C/ Doctor Nieto, 42
03013 ALICANTE
ESPAÑA
E-mail: Miguel.Garcia@ua.es
```


y además:


```
Text
27-11-1952

Text
17:10:00

Text
38:06 N

Text
0:57 W

Text
Miguel García

Text
Orihuela
```


Es decir, **el formulario conserva incluso los valores predeterminados de la interfaz que tenía el programa cuando fue compilado**.


---


### En resumen


Hemos pasado de:


```
ARMON.EXE
    ↓
"es un EXE Delphi"
```


a:


```
ARMON.EXE
    ↓
recursos NE
    ↓
RCDATA
    ↓
TPF0
    ↓
formularios Delphi
    ↓
componentes
    ↓
propiedades
    ↓
eventos
    ↓
nombres de procedimientos Pascal
```


El siguiente objetivo lógico es **mapear esos procedimientos al código máquina del EXE**. Esa es la parte que puede permitirnos reconstruir la lógica de ARMÓN, no solamente su interfaz.

## Usuario

~ $ cd ~/armon_forms
~/armon_forms $
~/armon_forms $ echo "===== EVENTOS DEL FORMULARIO PRINCIPAL ====="
===== EVENTOS DEL FORMULARIO PRINCIPAL =====
~/armon_forms $
~/armon_forms $ strings -a ARMON.EXE_10_TFORM1 |
> grep -E 'Click$|Change$|DblClick$|DownClick$|UpClick$|KeyUp$|Resize$|Create$|Activate$|Close$|Hide$|Paint$|Scroll$|Exit$' |
> sort -u
!PeriododeTiempoparatrnsitos1Click
AhoraClick
AjustarArmonicas1Click
AlternarCartas1Click
ArchivarDibujo1Click
ArmnicosActivos1Click
ArmonicoChange
CalcularClick
CalcularSiempre1Click
CartaRadical1Click
ComposicindelaRueda1Click
DHGoHarmClick
DHMOtroClick
DHMSaveClick
DifHGMTDblClick
DinmicaArmnica1Click
DiseodePgina1Click
DurCicloChange
EdadActualChange
EdadActualDblClick
ElNegroes1Click
EstiloProgClick
Estrellas1Click
FormCreate
FormResize
Fuenteparadibujo1Click
GlifosenNegro1Click
Harmogramas1Click
Impresora1Click
Imprimir1Click
ImprimirAutomtico1Click
IndicadorparaGrises1Click
InsColClick
LugarDblClick
LugarKeyUp
MenuRapidoClick
MiguelGarcaFerrndez1Click
MoveWaveDownClick       OnUpClick
MoveWaveUpClick
MrgenesdePgina1Click
NodosPlanetarios1Click
NombreDblClick
NombreKeyUp
OnActivate
OnChange
OnClick
OnCreate
OnDblClick
OnDownClick
OnExit
OnKeyUp
OndaArmChange
OpcionesparaFlorArmonica1Click
OtrosGraficos1Click
PantallaenColor1Click
PartesDefinidos1Click
PlanetasqueEmitenAspectos1Click
PlanetasqueseAspectan1Click
PlanetasqueseDibujan1Click
PlanetasqueseNumeran1Click
RSLugarClick
Restringir1Click
SBAnnoDownClick OnUpClick
SBAnnoUpClick
SBCentDownClick OnUpClick
SBCentUpClick
SBDecadaDownClick       OnUpClick
SBDecadaUpClick
SBDiaDownClick  OnUpClick
SBDiaUpClick
SBHoraDownClick OnUpClick
SBHoraUpClick
SBMesDownClick  OnUpClick
SBMesUpClick
SBMinDownClick  OnUpClick
SBMinUpClick
SBSegDownClick  OnUpClick
SBSegUpClick
Salir1Click
Sinastra1Click
SpinButton1DownClick    OnUpClick
SpinButton1UpClick
SpinButton2DownClick    OnUpClick
SpinButton2UpClick
SpinButton3DownClick    OnUpClick
SpinButton3UpClick
SpinButton4DownClick    OnUpClick
SpinButton4UpClick
SpinLatDownClick        OnUpClick
SpinLatUpClick
SpinLonDownClick        OnUpClick
SpinLonUpClick
SuiteBergamasque1Click
SupColClick
UpDownByClick
UseACIMUTClick
UseARClick
UseDOMClick
UseHelioClick
UseSiderClick
ZodiacoChange
casasClick
~/armon_forms $
~/armon_forms $ echo

~/armon_forms $ echo "===== EVENTOS DE TODOS LOS FORMULARIOS ====="
===== EVENTOS DE TODOS LOS FORMULARIOS =====
~/armon_forms $
~/armon_forms $ for f in *; do
>     echo
>     echo "========== $f =========="
>     strings -a "$f" |
>     grep -E 'Click$|Change$|DblClick$|DownClick$|UpClick$|KeyUp$|Resize$|Create$|Activate$|Close$|Hide$|Paint$|Scroll$|Exit$' |
>     sort -u
> done

========== ARMON.EXE_10_TABOUTBOX ==========

========== ARMON.EXE_10_TAUXMENU ==========
        pullClick
FormActivate
OnActivate
OnClick

========== ARMON.EXE_10_TENTRETENER ==========

========== ARMON.EXE_10_TFORM1 ==========
!PeriododeTiempoparatrnsitos1Click
AhoraClick
AjustarArmonicas1Click
AlternarCartas1Click
ArchivarDibujo1Click
ArmnicosActivos1Click
ArmonicoChange
CalcularClick
CalcularSiempre1Click
CartaRadical1Click
ComposicindelaRueda1Click
DHGoHarmClick
DHMOtroClick
DHMSaveClick
DifHGMTDblClick
DinmicaArmnica1Click
DiseodePgina1Click
DurCicloChange
EdadActualChange
EdadActualDblClick
ElNegroes1Click
EstiloProgClick
Estrellas1Click
FormCreate
FormResize
Fuenteparadibujo1Click
GlifosenNegro1Click
Harmogramas1Click
Impresora1Click
Imprimir1Click
ImprimirAutomtico1Click
IndicadorparaGrises1Click
InsColClick
LugarDblClick
LugarKeyUp
MenuRapidoClick
MiguelGarcaFerrndez1Click
MoveWaveDownClick       OnUpClick
MoveWaveUpClick
MrgenesdePgina1Click
NodosPlanetarios1Click
NombreDblClick
NombreKeyUp
OnActivate
OnChange
OnClick
OnCreate
OnDblClick
OnDownClick
OnExit
OnKeyUp
OndaArmChange
OpcionesparaFlorArmonica1Click
OtrosGraficos1Click
PantallaenColor1Click
PartesDefinidos1Click
PlanetasqueEmitenAspectos1Click
PlanetasqueseAspectan1Click
PlanetasqueseDibujan1Click
PlanetasqueseNumeran1Click
RSLugarClick
Restringir1Click
SBAnnoDownClick OnUpClick
SBAnnoUpClick
SBCentDownClick OnUpClick
SBCentUpClick
SBDecadaDownClick       OnUpClick
SBDecadaUpClick
SBDiaDownClick  OnUpClick
SBDiaUpClick
SBHoraDownClick OnUpClick
SBHoraUpClick
SBMesDownClick  OnUpClick
SBMesUpClick
SBMinDownClick  OnUpClick
SBMinUpClick
SBSegDownClick  OnUpClick
SBSegUpClick
Salir1Click
Sinastra1Click
SpinButton1DownClick    OnUpClick
SpinButton1UpClick
SpinButton2DownClick    OnUpClick
SpinButton2UpClick
SpinButton3DownClick    OnUpClick
SpinButton3UpClick
SpinButton4DownClick    OnUpClick
SpinButton4UpClick
SpinLatDownClick        OnUpClick
SpinLatUpClick
SpinLonDownClick        OnUpClick
SpinLonUpClick
SuiteBergamasque1Click
SupColClick
UpDownByClick
UseACIMUTClick
UseARClick
UseDOMClick
UseHelioClick
UseSiderClick
ZodiacoChange
casasClick

========== ARMON.EXE_10_TFORM2 ==========
FormCreate
OnCreate

========== ARMON.EXE_10_TFORM3 ==========
FormActivate
FormResize
OnActivate
OnDblClick
OnKeyUp
OnResize
buscartextoDblClick
buscartextoKeyUp

========== ARMON.EXE_10_TFORM4 ==========
FormCreate
OKClick
OnClick
OnCreate
PrClick

========== ARMON.EXE_10_TGEODE ==========
CancelarClick
LeaderChange
OnChange
OnClick
OnDblClick
OnDownClick
SpinButton1DownClick    OnUpClick
SpinButton1UpClick
SpinButton2DownClick    OnUpClick
SpinButton2UpClick

========== ARMON.EXE_10_TGPSELECTOR ==========
FormActivate
ListBox1Click
ListBox1DblClick
OnActivate
OnClick
OnDblClick
OnResize
cambiarSCRClick

========== ARMON.EXE_10_TLISTAWS ==========
        FormKeyUp
ArchivarClick
BorrarClick
BuscarClick
DModeClick
FormActivate
FormCreate
FormResize
LargeChange
ListarClick
MarcarClick
OnActivate
OnClick
OnCreate
OnDownClick
OnKeyUp
OnResize
OnScroll
RegresarClick
ResumenClick
ScrollFocusScroll
SpinButton1DownClick    OnUpClick
SpinButton1UpClick
SpinButton2DownClick    OnUpClick
SpinButton2UpClick
SpinButton3DownClick    OnUpClick
SpinButton3UpClick

========== ARMON.EXE_10_TMISGRAPH ==========
AvantiClick
CancelarClick
FormCreate
FormResize
OnClick
OnCreate
OnResize
OtrosClick

========== ARMON.EXE_10_TMLISTA ==========
        ValeClick
BorrarClick
CancelarClick
FormActivate
InsertarClick
OnActivate
OnClick
OnResize

========== ARMON.EXE_10_TPROGRESO ==========
        FormClick
FormActivate
FormHide
OnActivate
OnClick
OnHide
~/armon_forms $ echo

~/armon_forms $ echo "===== PROCEDIMIENTOS EN EL EXE ====="
===== PROCEDIMIENTOS EN EL EXE =====
~/armon_forms $
~/armon_forms $ strings -a /storage/emulated/0/Download/armon/ARMON.EXE |
> grep -E 'Click$|Change$|DblClick$|DownClick$|UpClick$|KeyUp$|Resize$|Create$|Activate$|Close$|Hide$|Paint$|Scroll$|Exit$' |
> sort -u
        FechaExit
        FormClick
        FormKeyUp
        OnUpClick
        ValeClick
        pullClick
!PeriododeTiempoparatrnsitos1Click
&Close
.       pullClick
AhoraClick
AjustarArmonicas1Click
AlternarCartas1Click
ArchivarClick
ArchivarDibujo1Click
ArmnicosActivos1Click
ArmonicoChange
AvantiClick
BorrarClick
BuscarClick
CalcularClick
CalcularSiempre1Click
CanClose
CancelarClick
CartaRadical1Click
ComposicindelaRueda1Click
Ctl3DColorChange
Ctl3dDlgFramePaint
DHGoHarmClick
DHMOtroClick
DHMSaveClick
DModeClick
DifHGMTDblClick
DinmicaArmnica1Click
DiseodePgina1Click
DurCicloChange
EdadActualChange
EdadActualDblClick
ElNegroes1Click
EstiloProgClick
Estrellas1Click
FormActivate
FormCreate
FormHide
FormResize
Fuenteparadibujo1Click
GlifosenNegro1Click
Harmogramas1Click
Impresora1Click
Imprimir1Click
ImprimirAutomtico1Click
ImprimirenColor1Click
IndicadorparaGrises1Click
InsColClick
InsertarClick
LargeChange
LeaderChange
ListBox1Click
ListBox1DblClick
ListarClick
LugarDblClick
LugarKeyUp
MarcarClick
MenuRapidoClick
MiguelGarcaFerrndez1Click
MoveWaveDownClick       OnUpClick
MoveWaveUpClick
MrgenesdePgina1Click
NatFileKeyUp
NodosPlanetarios1Click
NombreDblClick
NombreKeyUp
OKClick
OnActivate
OnChange
OnClick
OnClose
OnCreate
OnDblClick
OnDownClick
OnExit
OnHide
OnKeyUp
OnResize
OnScroll
OndaArmChange
OpcionesparaFlorArmonica1Click
OtrosClick
OtrosGraficos1Click
PantallaenColor1Click
PartesDefinidos1Click
PlanetasqueEmitenAspectos1Click
PlanetasqueseAspectan1Click
PlanetasqueseDibujan1Click
PlanetasqueseNumeran1Click
PrClick
RSLugarClick
RegresarClick
Restringir1Click
ResumenClick
SBAnnoDownClick
SBAnnoDownClick OnUpClick
SBAnnoUpClick
SBCentDownClick
SBCentDownClick OnUpClick
SBCentUpClick
SBDecadaDownClick       OnUpClick
SBDecadaUpClick
SBDiaDownClick
SBDiaDownClick  OnUpClick
SBDiaUpClick
SBHoraDownClick
SBHoraDownClick OnUpClick
SBHoraUpClick
SBMesDownClick
SBMesDownClick  OnUpClick
SBMesUpClick
SBMinDownClick
SBMinDownClick  OnUpClick
SBMinUpClick
SBSegDownClick  OnUpClick
SBSegUpClick
Salir1Click
ScrollFocusScroll
Sinastra1Click
SpinButton1DownClick
SpinButton1DownClick    OnUpClick
SpinButton1UpClick
SpinButton2DownClick
SpinButton2DownClick    OnUpClick
SpinButton2UpClick
SpinButton3DownClick
SpinButton3DownClick    OnUpClick
SpinButton3UpClick
SpinButton4DownClick
SpinButton4DownClick    OnUpClick
SpinButton4UpClick
SpinLatDownClick
SpinLatDownClick        OnUpClick
SpinLatUpClick
SpinLonDownClick
SpinLonDownClick        OnUpClick
SpinLonUpClick
SuiteBergamasque1Click
SupColClick
UpDownByClick
UseACIMUTClick
UseARClick
UseDOMClick
UseHelioClick
UseSiderClick
ZodiacoChange
bkClose
buscarastroDblClick
buscartextoDblClick
buscartextoKeyUp
cambiarSCRClick
casasClick
ofNoTestFileCreate
s>">    FormClick

## Asistente

Esto confirma algo importante: **los nombres de los manejadores de eventos también están físicamente dentro del EXE**, no solo dentro de los recursos `TPF0`.


Pero hay que separar dos cosas:


1. **Lo que ya podemos recuperar directamente**
  - formularios;
  - componentes;
  - propiedades;
  - menús;
  - textos;
  - nombres de eventos;
  - estructura de la interfaz.
2. **Lo que todavía debemos reconstruir**
  - el código que ejecuta, por ejemplo, `CalcularClick`;
  - `ArmonicoChange`;
  - `ZodiacoChange`;
  - `DHGoHarmClick`;
  - etc.


Y aquí hay una pista especialmente valiosa: el `strings` del **EXE completo** contiene algunos nombres que no aparecen en los recursos que acabamos de extraer, por ejemplo:


```
FechaExit
NatFileKeyUp
OnClose
ImprimirenColor1Click
Ctl3dDlgFramePaint
ofNoTestFileCreate
```


Eso indica que **el EXE contiene más información de ejecución que la que vemos en los formularios individuales**.


### El siguiente paso que recomiendo


No intentaría todavía desensamblar todo el EXE. Primero debemos descubrir **cómo están representados los métodos Delphi dentro del NE**.


En particular queremos responder:


```
CalcularClick
     ↓
¿dónde está su dirección de código?
     ↓
¿en qué segmento NE?
     ↓
¿qué bytes de código corresponden?
     ↓
desensamblado 16-bit x86
     ↓
pseudocódigo Delphi
```


Para ello, lo más útil ahora es analizar la **tabla de segmentos del formato NE**.


Ejecuta esto sobre el EXE original:


```
Bashcd /storage/emulated/0/Download/armon

echo "===== CABECERA NE ====="
xxd -g 1 -l 128 ARMON.EXE

echo
echo "===== CABECERA NE COMO TEXTO ====="
xxd -g 1 -s 0x3c -l 128 ARMON.EXE
```


Y también:


```
Bashecho
echo "===== POSICIONES DE LAS CADENAS DE EVENTOS ====="

for s in \
CalcularClick \
ArmonicoChange \
ZodiacoChange \
DHGoHarmClick \
FormCreate \
FormResize \
UseHelioClick \
Sinastra1Click \
AjustarArmonicas1Click
do
    echo "--- $s ---"
    grep -abo "$s" ARMON.EXE
done
```


### ¿Por qué esto es importante?


El NE de Windows 3.1 tiene una estructura aproximadamente así:


```
ARMON.EXE
│
├── cabecera DOS
│
├── cabecera NE
│   │
│   ├── tabla de segmentos
│   ├── tabla de recursos
│   ├── tabla de residentes
│   ├── tabla de nombres
│   └── tabla de importación
│
├── segmentos de código
│
├── segmentos de datos
│
└── recursos
    ├── TFORM1
    ├── TFORM2
    ├── TGEODE
    └── ...
```


Ya conocemos perfectamente la última parte.


Ahora necesitamos descubrir la primera parte:


```
NE
 ↓
segmentos
 ↓
código Delphi
 ↓
métodos
```


Además, **no debemos confundir las cadenas `CalcularClick` con el código de `CalcularClick`**. La cadena es un nombre utilizado por el sistema de formularios para localizar el método; necesitamos encontrar la estructura que asocia ese nombre con una dirección de procedimiento.


Ese enlace es precisamente lo que nos interesa recuperar.


Con las salidas de esos dos bloques podemos empezar a construir un **mapa real del código de ARMÓN**, en lugar de limitarlo a `strings`.

## Usuario

Recuerda siempre darme los comandos a ejecutar en una sola caja

## Asistente

Sí. A partir de ahora, cuando te dé comandos para ejecutar en Termux, **los pondré todos juntos en una sola caja de código**, listos para copiar y pegar.


Para el paso que acabamos de plantear, sería así:


```
Bashcd /storage/emulated/0/Download/armon

echo "===== CABECERA DOS + NE ====="
xxd -g 1 -l 128 ARMON.EXE

echo
echo "===== ZONA DE CABECERA NE ====="
xxd -g 1 -s 0x3c -l 128 ARMON.EXE

echo
echo "===== POSICIONES DE LOS MANEJADORES ====="
for s in \
CalcularClick \
ArmonicoChange \
ZodiacoChange \
DHGoHarmClick \
FormCreate \
FormResize \
UseHelioClick \
Sinastra1Click \
AjustarArmonicas1Click
do
    echo "--- $s ---"
    grep -abo "$s" ARMON.EXE
done
```


Y mantendré ese formato para los siguientes pasos.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ echo "===== CABECERA DOS + NE ====="
===== CABECERA DOS + NE =====
.../Download/armon $ xxd -g 1 -l 128 ARMON.EXE
00000000: 4d 5a 00 01 01 00 00 00 08 00 10 00 ff ff 08 00  MZ..............
00000010: 00 01 00 00 00 00 00 00 40 00 00 00 00 00 00 00  ........@.......
00000020: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000030: 00 00 00 00 00 00 00 00 00 00 00 00 00 01 00 00  ................
00000040: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000050: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000060: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000070: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
.../Download/armon $
.../Download/armon $ echo

.../Download/armon $ echo "===== ZONA DE CABECERA NE ====="
===== ZONA DE CABECERA NE =====
.../Download/armon $ xxd -g 1 -s 0x3c -l 128 ARMON.EXE
0000003c: 00 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
0000004c: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
0000005c: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
0000006c: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
0000007c: 00 00 00 00 ba 10 00 0e 1f b4 09 cd 21 b8 01 4c  ............!..L
0000008c: cd 21 90 90 54 68 69 73 20 70 72 6f 67 72 61 6d  .!..This program
0000009c: 20 72 65 71 75 69 72 65 73 20 4d 69 63 72 6f 73   requires Micros
000000ac: 6f 66 74 20 57 69 6e 64 6f 77 73 2e 0d 0a 24 20  oft Windows...$
.../Download/armon $
.../Download/armon $ echo

.../Download/armon $ echo "===== POSICIONES DE LOS MANEJADORES ====="
===== POSICIONES DE LOS MANEJADORES =====
.../Download/armon $ for s in \
> CalcularClick \
> ArmonicoChange \
> ZodiacoChange \
> DHGoHarmClick \
> FormCreate \
> FormResize \
> UseHelioClick \
> Sinastra1Click \
> AjustarArmonicas1Click
> do
>     echo "--- $s ---"
>     grep -abo "$s" ARMON.EXE
> done
--- CalcularClick ---
308076:CalcularClick
3098283:CalcularClick
--- ArmonicoChange ---
309024:ArmonicoChange
3099673:ArmonicoChange
--- ZodiacoChange ---
308570:ZodiacoChange
3099187:ZodiacoChange
3099209:ZodiacoChange
3099424:ZodiacoChange
--- DHGoHarmClick ---
308840:DHGoHarmClick
3075033:DHGoHarmClick
--- FormCreate ---
237180:FormCreate
248608:FormCreate
308145:FormCreate
338414:FormCreate
1386668:FormCreate
3073145:FormCreate
3074778:FormCreate
3114905:FormCreate
3125110:FormCreate
3141217:FormCreate
--- FormResize ---
248593:FormResize
308094:FormResize
338595:FormResize
2681025:FormResize
3073166:FormResize
3074757:FormResize
3114945:FormResize
3137939:FormResize
--- UseHelioClick ---
308109:UseHelioClick
3097946:UseHelioClick
--- Sinastra1Click ---
309803:Sinastra1Click
3108289:Sinastra1Click
3113725:Sinastra1Click
--- AjustarArmonicas1Click ---
309630:AjustarArmonicas1Click
3112482:AjustarArmonicas1Click
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Muy bien. Los datos confirman que **`0x100` es el comienzo de la cabecera NE**: el valor de `0x3C` es `0x00000100`. Además, las dos familias de posiciones de `CalcularClick`, etc., probablemente corresponden a zonas distintas del ejecutable.


Ahora no vamos a interpretar esas posiciones a ciegas. Vamos a leer directamente los campos de la cabecera NE y localizar sus tablas.


Ejecuta **todo este bloque de una sola vez**:


```
Bashcd /storage/emulated/0/Download/armon

echo "===== CABECERA NE EN 0x100 ====="
xxd -g 1 -s 0x100 -l 128 ARMON.EXE

echo
echo "===== ANALISIS DE CAMPOS NE ====="

python3 - <<'PY'
import struct

f = "ARMON.EXE"

with open(f, "rb") as x:
    data = x.read()

ne_off = struct.unpack_from("<I", data, 0x3c)[0]

print(f"Offset NE          : 0x{ne_off:X}")

if data[ne_off:ne_off+2] != b"NE":
    print("ERROR: no se encontro firma NE")
    raise SystemExit

def u8(o):
    return data[ne_off + o]

def u16(o):
    return struct.unpack_from("<H", data, ne_off + o)[0]

print(f"Firma              : {data[ne_off:ne_off+2]!r}")
print(f"Entry Table offset : 0x{u16(0x04):04X}")
print(f"Entry Table length : 0x{u16(0x06):04X}")
print(f"Flags              : 0x{u16(0x0C):04X}")
print(f"Auto Data Segment  : 0x{u16(0x0E):04X}")
print(f"Initial SS:SP      : {u16(0x16):04X}:{u16(0x14):04X}")
print(f"Initial CS:IP      : {u16(0x1A):04X}:{u16(0x18):04X}")
print(f"Segment count      : {u16(0x1C)}")
print(f"Module refs        : {u16(0x1E)}")
print(f"Non-resident size  : {u16(0x20)}")

seg_table = u16(0x22)
res_table = u16(0x24)
resident_names = u16(0x26)
module_refs = u16(0x28)
import_names = u16(0x2A)
nonresident_names = struct.unpack_from("<I", data, ne_off + 0x2C)[0]
sector_shift = u16(0x32)

print()
print("===== TABLAS =====")
print(f"Segment table      : NE+0x{seg_table:04X} = archivo 0x{ne_off+seg_table:X}")
print(f"Resource table     : NE+0x{res_table:04X} = archivo 0x{ne_off+res_table:X}")
print(f"Resident names     : NE+0x{resident_names:04X} = archivo 0x{ne_off+resident_names:X}")
print(f"Module references  : NE+0x{module_refs:04X} = archivo 0x{ne_off+module_refs:X}")
print(f"Imported names     : NE+0x{import_names:04X} = archivo 0x{ne_off+import_names:X}")
print(f"Nonresident names  : archivo 0x{nonresident_names:X}")
print(f"Sector shift       : {sector_shift}  (tamano sector = {1<<sector_shift})")

print()
print("===== SEGMENTOS NE =====")

seg_abs = ne_off + seg_table
sector_size = 1 << sector_shift

for i in range(u16(0x1C)):
    p = seg_abs + i * 8

    off_units, length, flags, minalloc = struct.unpack_from("<HHHH", data, p)

    file_offset = off_units * sector_size

    if length == 0:
        file_size = sector_size
    else:
        file_size = length

    print(
        f"{i+1:3d}: "
        f"archivo=0x{file_offset:08X} "
        f"tam=0x{file_size:04X} "
        f"flags=0x{flags:04X} "
        f"minalloc=0x{minalloc:04X}"
    )

print()
print("===== ENTRADA DEL PROGRAMA =====")
cs = u16(0x1A)
ip = u16(0x18)
ss = u16(0x16)
sp = u16(0x14)

print(f"CS:IP = {cs:04X}:{ip:04X}")
print(f"SS:SP = {ss:04X}:{sp:04X}")
print()
print("El siguiente paso sera relacionar los segmentos de codigo")
print("con las cadenas de los manejadores Delphi.")
PY
```

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ echo "===== CABECERA NE EN 0x100 ====="
===== CABECERA NE EN 0x100 =====
.../Download/armon $ xxd -g 1 -s 0x100 -l 128 ARMON.EXE
= ANALISIS DE CAMPOS NE ====="

python3 - <<'PY'
import struct

f = "ARMON.EXE"

with open(f, "rb") as x:
    data = x.read()

ne_off = struct.unpack_from("<I", data, 0x3c)[0]

print(f"Offset NE          : 0x{ne_off:X}")

if data[ne_off:ne_off+2] != b"NE":
    print("ERROR: no se encontro firma NE")
    raise SystemExit

def u8(o):
    return data[ne_off + o]

def u16(o):
    return struct.unpack_from("<H", data, ne_off + o)[0]

print(f"Firma00000100: 4e 45 06 01 ee 06 4e 00 00 00 00 00 0a 03 51 00  NE....N.......Q.
00000110: 9c 1f 50 7b 02 00 01 00 00 00 51 00 51 00 0c 00  ..P{......Q.Q...
00000120: 0d 00 40 00 c8 02 77 06 80 06 98 06 3c 08 00 00  ..@...w.....<...
00000130: 0c 00 06 00 00 00 02 00 00 00 00 00 00 00 0a 03  ................
00000140: e5 0d f6 3f 10 1d f7 3f 29 0f 2d 9d 10 1d 2e 9d  ...?...?).-.....
00000150: cb 12 d0 6e 10 1d d0 6e a5 14 50 3f 10 1d 51 3f  ...n...n..P?..Q?
00000160: aa 15 25 af 10 1d 25 af ef 18 82 3f 10 1d 82 3f  ..%...%....?...?
00000170: 36 1a d6 62 10 1d d7 62 38 1c ed 80 10 1d ee 80  6..b...b8.......
.../Download/armon $
.../Download/armon $ echo

.../Download/armon $ echo "===== ANALISIS DE CAMPOS NE ====="
===== ANALISIS DE CAMPOS NE =====
.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> import struct
>
> f = "ARMON.EXE"
>
> with open(f, "rb") as x:
>     data = x.read()
>
0x1C)):
    p = > ne_off = struct.unpack_from("<I", data, 0x3c)[0]
>
data, p)

    f> print(f"Offset NE          : 0x{ne_off:X}")
>
> if data[ne_off:ne_off+2] != b"NE":
 length
>     print("ERROR: no se encontro firma NE")
 "
     >     raise SystemExit
>
     f"flags=> def u8(o):
lloc=0x{>     return data[ne_off + o]
>
L PROGRAMA ====> def u16(o):
>     return struct.unpack_from("<H", data, ne_off + o)[0]
>
> print(f"Firma              : {data[ne_off:ne_off+2]!r}")
> print(f"Entry Table offset : 0x{u16(0x04):04X}")
> print(f"Entry Table length : 0x{u16(0x06):04X}")
(f"SS:SP = {ss:04X}:{sp:04X}")
print()
print("El siguiente paso sera relacionar los segmentos de codigo")
print("con las cadenas de los manejadores Delphi.")
PY> print(f"Flags              : 0x{u16(0x0C):04X}")
> print(f"Auto Data Segment  : 0x{u16(0x0E):04X}")
> print(f"Initial SS:SP      : {u16(0x16):04X}:{u16(0x14):04X}")
> print(f"Initial CS:IP      : {u16(0x1A):04X}:{u16(0x18):04X}")
> print(f"Segment count      : {u16(0x1C)}")
> print(f"Module refs        : {u16(0x1E)}")
> print(f"Non-resident size  : {u16(0x20)}")
>
> seg_table = u16(0x22)
> res_table = u16(0x24)
> resident_names = u16(0x26)
> module_refs = u16(0x28)
> import_names = u16(0x2A)
> nonresident_names = struct.unpack_from("<I", data, ne_off + 0x2C)[0]
> sector_shift = u16(0x32)
>
> print()
> print("===== TABLAS =====")
> print(f"Segment table      : NE+0x{seg_table:04X} = archivo 0x{ne_off+seg_table:X}")
> print(f"Resource table     : NE+0x{res_table:04X} = archivo 0x{ne_off+res_table:X}")
> print(f"Resident names     : NE+0x{resident_names:04X} = archivo 0x{ne_off+resident_names:X}")
> print(f"Module references  : NE+0x{module_refs:04X} = archivo 0x{ne_off+module_refs:X}")
> print(f"Imported names     : NE+0x{import_names:04X} = archivo 0x{ne_off+import_names:X}")
> print(f"Nonresident names  : archivo 0x{nonresident_names:X}")
> print(f"Sector shift       : {sector_shift}  (tamano sector = {1<<sector_shift})")
>
> print()
> print("===== SEGMENTOS NE =====")
>
> seg_abs = ne_off + seg_table
> sector_size = 1 << sector_shift
>
> for i in range(u16(0x1C)):
>     p = seg_abs + i * 8
>
>     off_units, length, flags, minalloc = struct.unpack_from("<HHHH", data, p)
>
>     file_offset = off_units * sector_size
>
>     if length == 0:
>         file_size = sector_size
>     else:
>         file_size = length
>
>     print(
>         f"{i+1:3d}: "
>         f"archivo=0x{file_offset:08X} "
>         f"tam=0x{file_size:04X} "
>         f"flags=0x{flags:04X} "
>         f"minalloc=0x{minalloc:04X}"
>     )
>
> print()
> print("===== ENTRADA DEL PROGRAMA =====")
> cs = u16(0x1A)
> ip = u16(0x18)
> ss = u16(0x16)
> sp = u16(0x14)
>
> print(f"CS:IP = {cs:04X}:{ip:04X}")
> print(f"SS:SP = {ss:04X}:{sp:04X}")
> print()
> print("El siguiente paso sera relacionar los segmentos de codigo")
> print("con las cadenas de los manejadores Delphi.")
> PY
Offset NE          : 0x100
Firma              : b'NE'
Entry Table offset : 0x06EE
Entry Table length : 0x004E
Flags              : 0x030A
Auto Data Segment  : 0x0051
Initial SS:SP      : 0001:0002
Initial CS:IP      : 0051:0000
Segment count      : 81
Module refs        : 12
Non-resident size  : 13

===== TABLAS =====
Segment table      : NE+0x0040 = archivo 0x140
Resource table     : NE+0x02C8 = archivo 0x3C8
Resident names     : NE+0x0677 = archivo 0x777
Module references  : NE+0x0680 = archivo 0x780
Imported names     : NE+0x0698 = archivo 0x798
Nonresident names  : archivo 0x83C
Sector shift       : 6  (tamano sector = 64)

===== SEGMENTOS NE =====
  1: archivo=0x00037940 tam=0x3FF6 flags=0x1D10 minalloc=0x3FF7
  2: archivo=0x0003CA40 tam=0x9D2D flags=0x1D10 minalloc=0x9D2E
  3: archivo=0x0004B2C0 tam=0x6ED0 flags=0x1D10 minalloc=0x6ED0
  4: archivo=0x00052940 tam=0x3F50 flags=0x1D10 minalloc=0x3F51
  5: archivo=0x00056A80 tam=0xAF25 flags=0x1D10 minalloc=0xAF25
  6: archivo=0x00063BC0 tam=0x3F82 flags=0x1D10 minalloc=0x3F82
  7: archivo=0x00068D80 tam=0x62D6 flags=0x1D10 minalloc=0x62D7
  8: archivo=0x00070E00 tam=0x80ED flags=0x1D10 minalloc=0x80EE
  9: archivo=0x0007C400 tam=0xF0E4 flags=0x1D10 minalloc=0xF0E4
 10: archivo=0x00090B40 tam=0xF03D flags=0x1D10 minalloc=0xF03D
 11: archivo=0x000A6240 tam=0x4DF7 flags=0x1D10 minalloc=0x4DF7
 12: archivo=0x000AD180 tam=0x99C4 flags=0x1D10 minalloc=0x99C4
 13: archivo=0x000BA640 tam=0xBF4D flags=0x1D10 minalloc=0xBF4D
 14: archivo=0x000CB740 tam=0x3EA7 flags=0x1D10 minalloc=0x3EA7
 15: archivo=0x000D0FC0 tam=0xD32A flags=0x1D10 minalloc=0xD32A
 16: archivo=0x000E3880 tam=0x78F0 flags=0x1D10 minalloc=0x78F0
 17: archivo=0x000ED500 tam=0x7983 flags=0x1D10 minalloc=0x7983
 18: archivo=0x000F7E40 tam=0x86D4 flags=0x1D10 minalloc=0x86D4
 19: archivo=0x00104240 tam=0xB1FE flags=0x1D10 minalloc=0xB1FE
 20: archivo=0x00112D40 tam=0x8808 flags=0x1D10 minalloc=0x8808
 21: archivo=0x00120D80 tam=0x4305 flags=0x1D10 minalloc=0x4305
 22: archivo=0x00126A00 tam=0xEA0E flags=0x1D10 minalloc=0xEA0F
 23: archivo=0x00139480 tam=0x9B80 flags=0x1D10 minalloc=0x9B80
 24: archivo=0x00145E00 tam=0x7414 flags=0x1D10 minalloc=0x7414
 25: archivo=0x0014F040 tam=0x3D29 flags=0x1D10 minalloc=0x3D29
 26: archivo=0x00153A40 tam=0xCDC9 flags=0x1D10 minalloc=0xCDC9
 27: archivo=0x00165400 tam=0x4DB5 flags=0x1D10 minalloc=0x4DB5
 28: archivo=0x0016BC80 tam=0xEDC4 flags=0x1D10 minalloc=0xEDC4
 29: archivo=0x00180140 tam=0xC991 flags=0x1D10 minalloc=0xC991
 30: archivo=0x001916C0 tam=0xD12F flags=0x1D10 minalloc=0xD12F
 31: archivo=0x001A44C0 tam=0x4469 flags=0x1D10 minalloc=0x4469
 32: archivo=0x001A96C0 tam=0x3BC5 flags=0x1D10 minalloc=0x3BC5
 33: archivo=0x001AEE80 tam=0x3B3B flags=0x1D10 minalloc=0x3B3B
 34: archivo=0x001B2EC0 tam=0x7EBC flags=0x1D10 minalloc=0x7EBC
 35: archivo=0x001BD840 tam=0xC336 flags=0x1D10 minalloc=0xC336
 36: archivo=0x001CDB80 tam=0x3BDD flags=0x1D10 minalloc=0x3BDD
 37: archivo=0x001D28C0 tam=0xE0C1 flags=0x1D10 minalloc=0xE0C1
 38: archivo=0x001E7B00 tam=0x41F8 flags=0x1D10 minalloc=0x41F8
 39: archivo=0x001EEF00 tam=0xC820 flags=0x1D10 minalloc=0xC820
 40: archivo=0x001FFB80 tam=0x53E6 flags=0x1D10 minalloc=0x53E6
 41: archivo=0x00206B80 tam=0x5C50 flags=0x1D10 minalloc=0x5C50
 42: archivo=0x0020E380 tam=0x3EF9 flags=0x1D10 minalloc=0x3EF9
 43: archivo=0x00213300 tam=0x4874 flags=0x1D10 minalloc=0x4874
 44: archivo=0x00219B40 tam=0x3F08 flags=0x1D10 minalloc=0x3F09
 45: archivo=0x0021F980 tam=0x7B16 flags=0x1D10 minalloc=0x7B16
 46: archivo=0x00228FC0 tam=0xF600 flags=0x1D10 minalloc=0xF600
 47: archivo=0x0023E040 tam=0x5A44 flags=0x1D10 minalloc=0x5A44
 48: archivo=0x00245C40 tam=0xB3F1 flags=0x1D10 minalloc=0xB3F1
 49: archivo=0x002563C0 tam=0x79D0 flags=0x1D10 minalloc=0x79D0
 50: archivo=0x0025FB00 tam=0x3C8E flags=0x1D10 minalloc=0x3C8E
 51: archivo=0x00264640 tam=0x3AB4 flags=0x1D10 minalloc=0x3AB5
 52: archivo=0x00268640 tam=0x4147 flags=0x1D10 minalloc=0x4147
 53: archivo=0x0026CCC0 tam=0x3DB8 flags=0x1D10 minalloc=0x3DB8
 54: archivo=0x00271FC0 tam=0x45D0 flags=0x1D10 minalloc=0x45D0
 55: archivo=0x002784C0 tam=0xCFF6 flags=0x1D10 minalloc=0xCFF6
 56: archivo=0x00288F80 tam=0x3D77 flags=0x1D10 minalloc=0x3D77
 57: archivo=0x0028E800 tam=0x3B9B flags=0x1D10 minalloc=0x3B9B
 58: archivo=0x00292BC0 tam=0xBED1 flags=0x1D10 minalloc=0xBED1
 59: archivo=0x002A5B00 tam=0x3FF4 flags=0x1D10 minalloc=0x3FF5
 60: archivo=0x002AAEC0 tam=0x8732 flags=0x1D10 minalloc=0x8732
 61: archivo=0x002BD900 tam=0x3FD5 flags=0x1D10 minalloc=0x3FD5
 62: archivo=0x002C3180 tam=0x3D11 flags=0x1D10 minalloc=0x3D11
 63: archivo=0x002C7340 tam=0x30D2 flags=0x1D10 minalloc=0x30D2
 64: archivo=0x002CB180 tam=0x2670 flags=0x1D10 minalloc=0x2670
 65: archivo=0x002D0080 tam=0x390C flags=0x1D10 minalloc=0x390C
 66: archivo=0x002D6F80 tam=0x43D6 flags=0x1D10 minalloc=0x43D7
 67: archivo=0x002DC340 tam=0x376F flags=0x1D10 minalloc=0x3770
 68: archivo=0x002E1A80 tam=0x9AD0 flags=0x1D10 minalloc=0x9AD0
 69: archivo=0x002EB6C0 tam=0x2876 flags=0x1D10 minalloc=0x2877
 70: archivo=0x00000900 tam=0x3879 flags=0x1D50 minalloc=0x3879
 71: archivo=0x000043C0 tam=0x2FC1 flags=0x1D50 minalloc=0x2FC1
 72: archivo=0x00007400 tam=0x2540 flags=0x1D50 minalloc=0x2541
 73: archivo=0x00009A00 tam=0x8716 flags=0x1D50 minalloc=0x8717
 74: archivo=0x00012280 tam=0x722E flags=0x1D50 minalloc=0x722E
 75: archivo=0x00019740 tam=0x22CC flags=0x1D50 minalloc=0x22CC
 76: archivo=0x0001BAC0 tam=0x6EB3 flags=0x1D50 minalloc=0x6EB3
 77: archivo=0x00022C00 tam=0x7AAA flags=0x1D50 minalloc=0x7AAB
 78: archivo=0x0002A9C0 tam=0x51C4 flags=0x1D50 minalloc=0x51C4
 79: archivo=0x0002FC80 tam=0x2AC9 flags=0x0D50 minalloc=0x2AC9
 80: archivo=0x00032A00 tam=0x261A flags=0x0D50 minalloc=0x261A
 81: archivo=0x00035700 tam=0x21E4 flags=0x0D59 minalloc=0x62C4

===== ENTRADA DEL PROGRAMA =====
CS:IP = 0051:0000
SS:SP = 0001:0002

El siguiente paso sera relacionar los segmentos de codigo
con las cadenas de los manejadores Delphi.
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Aquí aparece una corrección importante respecto al análisis anterior: **mi interpretación de algunos campos de la cabecera NE estaba desplazada 2 bytes**. La estructura NE que estamos usando coloca `CS:IP` en `0x14` y `SS:SP` en `0x18`.


Y tus datos contienen una pista excelente:


- `CalcularClick` en `0x4B36C` → está dentro del **segmento 3**.
- `CalcularClick` en `0x2F46AB` → está en la zona de **recursos Delphi `TPF0`**.
- Lo mismo ocurre con `ArmonicoChange`, `DHGoHarmClick`, etc.


Es decir, **ya estamos separando las copias de los nombres que pertenecen a los recursos de las que están dentro de los segmentos del programa**.


Vamos a comprobarlo sistemáticamente y, además, corregir la lectura de `CS:IP`.


Ejecuta todo este bloque:


```
Bashcd /storage/emulated/0/Download/armon

python3 - <<'PY'
import struct

EXE = "ARMON.EXE"

with open(EXE, "rb") as f:
    data = f.read()

ne = struct.unpack_from("<I", data, 0x3c)[0]

def W(o):
    return struct.unpack_from("<H", data, ne + o)[0]

print("========================================")
print("        CABECERA NE CORREGIDA")
print("========================================")

print(f"NE                 : 0x{ne:X}")
print(f"Firma              : {data[ne:ne+2]!r}")
print(f"Linker             : {data[ne+2]}.{data[ne+3]}")
print(f"Entry table        : 0x{W(0x04):04X}")
print(f"Entry table size   : 0x{W(0x06):04X}")
print(f"Flags              : 0x{W(0x0C):04X}")
print(f"AutoData segment   : {W(0x0E)}")
print(f"Heap inicial       : 0x{W(0x10):04X}")
print(f"Stack inicial      : 0x{W(0x12):04X}")

# NE 0x14 = IP, 0x16 = CS
ip = W(0x14)
cs = W(0x16)

# NE 0x18 = SP, 0x1A = SS
sp = W(0x18)
ss = W(0x1A)

print(f"CS:IP              : {cs:04X}:{ip:04X}")
print(f"SS:SP              : {ss:04X}:{sp:04X}")

seg_count = W(0x1C)
seg_table = W(0x22)
shift = W(0x32)
sector_size = 1 << shift

print(f"Segmentos          : {seg_count}")
print(f"Tabla segmentos    : NE+0x{seg_table:04X}")
print(f"Sector             : {sector_size} bytes")

print()
print("========================================")
print("             SEGMENTOS NE")
print("========================================")

segments = []

for n in range(1, seg_count + 1):

    p = ne + seg_table + (n - 1) * 8

    sector, length, flags, minalloc = struct.unpack_from(
        "<HHHH", data, p
    )

    file_offset = sector * sector_size

    # En NE, length=0 representa 64 KiB
    file_size = 0x10000 if length == 0 else length

    end = file_offset + file_size

    segments.append({
        "num": n,
        "offset": file_offset,
        "size": file_size,
        "end": end,
        "flags": flags,
        "minalloc": minalloc
    })

    tipo = flags & 0x0007

    if tipo == 0:
        tipo_txt = "CODE"
    elif tipo == 1:
        tipo_txt = "DATA"
    else:
        tipo_txt = f"TIPO-{tipo}"

    print(
        f"{n:02d}  "
        f"0x{file_offset:06X}-0x{end-1:06X}  "
        f"{file_size:6d} bytes  "
        f"flags=0x{flags:04X}  "
        f"{tipo_txt}"
    )

print()
print("========================================")
print("       UBICACION DE LOS MANEJADORES")
print("========================================")

nombres = [
    "CalcularClick",
    "ArmonicoChange",
    "ZodiacoChange",
    "DHGoHarmClick",
    "FormCreate",
    "FormResize",
    "UseHelioClick",
    "Sinastra1Click",
    "AjustarArmonicas1Click"
]

for nombre in nombres:

    posiciones = []
    inicio = 0

    while True:
        p = data.find(nombre.encode("latin1"), inicio)

        if p < 0:
            break

        posiciones.append(p)
        inicio = p + 1

    print()
    print(nombre)

    for p in posiciones:

        encontrado = None

        for s in segments:
            if s["offset"] <= p < s["end"]:
                encontrado = s
                break

        if encontrado:
            interno = p - encontrado["offset"]

            print(
                f"  archivo 0x{p:06X} "
                f"(decimal {p}) "
                f"-> segmento {encontrado['num']} "
                f"+0x{interno:04X}"
            )
        else:
            print(
                f"  archivo 0x{p:06X} "
                f"(decimal {p}) "
                f"-> FUERA DE SEGMENTOS"
            )

print()
print("========================================")
print("             RESULTADO CLAVE")
print("========================================")
print()
print("Las apariciones que caen dentro de segmentos NE")
print("son las que debemos investigar para encontrar")
print("las referencias utilizadas por el codigo Delphi.")
print()
print("Las apariciones alrededor de 0x2Fxxxx pertenecen")
print("a los recursos TPF0 y no son el codigo del metodo.")
PY
```


Lo especialmente interesante será comprobar que las apariciones cercanas a:


```
0x4Bxxx
```


caen en el **segmento 3**, mientras que las de:


```
0x2Fxxxx
```


corresponden a los recursos `TPF0`.


Si eso se confirma, tendremos nuestro primer **segmento de código Delphi identificado con precisión**. El siguiente paso será mucho más interesante: extraer el segmento 3 y buscar **las instrucciones que hacen referencia a `CalcularClick`, `ArmonicoChange`, etc.**, que es lo que puede llevarnos finalmente desde los nombres de los eventos hasta el código máquina que implementa cada procedimiento.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
ct

EXE = "ARMON.EXE"

with open(EXE, "rb") as f:
    data.../Download/armon $
.../Download/armon $ python3 - <<'PY'
c)[0]

def W(o):
    > import struct
>
> EXE = "ARMON.EXE"
>
> with open(EXE, "rb") as f:
>     data = f.read()
>
> ne = struct.unpack_from("<I", data, 0x3c)[0]
>
> def W(o):
>     return struct.unpack_from("<H", data, ne + o)[0]
>
> print("========================================")
> print("        CABECERA NE CORREGIDA")
> print("========================================")
>
toData segment   : {W(0x0E)}")
print(f"Heap inicial       : 0x{W(0x10):04X}")
print(f"Stack inicial      : 0x{W(0x12):04X}")

# NE 0x14 = IP, 0x16 = CS
ip = W(0x14)
cs = W(0x16)

# NE 0x18 = SP, 0x1A = SS
sp = W(0x18)
ss = W(> print(f"NE                 : 0x{ne:X}")
> print(f"Firma              : {data[ne:ne+2]!r}")
> print(f"Linker             : {data[ne+2]}.{data[ne+3]}")
> print(f"Entry table        : 0x{W(0x04):04X}")
> print(f"Entry table size   : 0x{W(0x06):04X}")
> print(f"Flags              : 0x{W(0x0C):04X}")
> print(f"AutoData segment   : {W(0x0E)}")
> print(f"Heap inicial       : 0x{W(0x10):04X}")
> print(f"Stack inicial      : 0x{W(0x12):04X}")
>
> # NE 0x14 = IP, 0x16 = CS
> ip = W(0x14)
> cs = W(0x16)
>
> # NE 0x18 = SP, 0x1A = SS
> sp = W(0x18)
> ss = W(0x1A)
>
> print(f"CS:IP              : {cs:04X}:{ip:04X}")
> print(f"SS:SP              : {ss:04X}:{sp:04X}")
>
> seg_count = W(0x1C)
> seg_table = W(0x22)
> shift = W(0x32)
> sector_size = 1 << shift
>
> print(f"Segmentos          : {seg_count}")
> print(f"Tabla segmentos    : NE+0x{seg_table:04X}")
> print(f"Sector             : {sector_size} bytes")
>
> print()
> print("========================================")
> print("             SEGMENTOS NE")
> print("========================================")
>
> segments = []
>
> for n in range(1, seg_count + 1):
>
>     p = ne + seg_table + (n - 1) * 8
>
"
        f"{tipo_txt}"
    )

pr>     sector, length, flags, minalloc = struct.unpack_from(
>         "<HHHH", data, p
>     )
>
>     file_offset = sector * sector_size
>
>     # En NE, length=0 representa 64 KiB
>     file_size = 0x10000 if length == 0 else length
>
>     end = file_offset + file_size
>
>     segments.append({
>         "num": n,
,
   >         "offset": file_offset,
>         "size": file_size,
te",
 >         "end": end,
>         "flags": flags,
lick>         "minalloc": minalloc
>     })
Click"
]>
mbre in no>     tipo = flags & 0x0007
0

  >
  w>     if tipo == 0:
>         tipo_txt = "CODE"
o)

   >     elif tipo == 1:
eak

  >         tipo_txt = "DATA"
p)
  >     else:
>         tipo_txt = f"TIPO-{tipo}"
>

        >     print(
>         f"{n:02d}  "
>         f"0x{file_offset:06X}-0x{end-1:06X}  "
rado = s
 >         f"{file_size:6d} bytes  "
rado:
 >         f"flags=0x{flags:04X}  "
>         f"{tipo_txt}"
>     )
  archivo 0x{p:06X} >
    f"(d> print()
           > print("========================================")
> print("       UBICACION DE LOS MANEJADORES")
> print("========================================")
       f">
> nombres = [
>     "CalcularClick",
  )

pri>     "ArmonicoChange",
>     "ZodiacoChange",
>     "DHGoHarmClick",
>     "FormCreate",
>     "FormResize",
>     "UseHelioClick",
>     "Sinastra1Click",
>     "AjustarArmonicas1Click"
> ]
>
> for nombre in nombres:
>
>     posiciones = []
>     inicio = 0
>
>     while True:                                      )
>         p = data.find(nombre.encode("latin1"), inicio)
>         if p < 0:
>             break
>
>         posiciones.append(p)
>         inicio = p + 1
>
>     print()
>     print(nombre)
>
>     for p in posiciones:
>
>         encontrado = None
>
>         for s in segments:
>             if s["offset"] <= p < s["end"]:
>                 encontrado = s
>                 break
>
>         if encontrado:
>             interno = p - encontrado["offset"]
>
>             print(
>                 f"  archivo 0x{p:06X} "
>                 f"(decimal {p}) "
>                 f"-> segmento {encontrado['num']} "
>                 f"+0x{interno:04X}"
>             )
>         else:
>             print(
>                 f"  archivo 0x{p:06X} "
>                 f"(decimal {p}) "
>                 f"-> FUERA DE SEGMENTOS"
>             )
>
> print()
> print("========================================")
> print("             RESULTADO CLAVE")
> print("========================================")
> print()
> print("Las apariciones que caen dentro de segmentos N)
> print("las referencias utilizadas por el codigo Delphi.")
> print()
> print("Las apariciones alrededor de 0x2Fxxxx pertenecen")
> print("a los recursos TPF0 y no son el codigo del metodo.")
> PY
========================================
        CABECERA NE CORREGIDA
========================================
NE                 : 0x100
Firma              : b'NE'
Linker             : 6.1
Entry table        : 0x06EE
Entry table size   : 0x004E
Flags              : 0x030A
AutoData segment   : 81
Heap inicial       : 0x1F9C
Stack inicial      : 0x7B50
CS:IP              : 0001:0002
SS:SP              : 0051:0000
Segmentos          : 81
Tabla segmentos    : NE+0x0040
Sector             : 64 bytes

========================================
             SEGMENTOS NE
========================================
01  0x037940-0x03B935   16374 bytes  flags=0x1D10  CODE
02  0x03CA40-0x04676C   40237 bytes  flags=0x1D10  CODE
03  0x04B2C0-0x05218F   28368 bytes  flags=0x1D10  CODE
04  0x052940-0x05688F   16208 bytes  flags=0x1D10  CODE
05  0x056A80-0x0619A4   44837 bytes  flags=0x1D10  CODE
06  0x063BC0-0x067B41   16258 bytes  flags=0x1D10  CODE
07  0x068D80-0x06F055   25302 bytes  flags=0x1D10  CODE
08  0x070E00-0x078EEC   33005 bytes  flags=0x1D10  CODE
09  0x07C400-0x08B4E3   61668 bytes  flags=0x1D10  CODE
10  0x090B40-0x09FB7C   61501 bytes  flags=0x1D10  CODE
11  0x0A6240-0x0AB036   19959 bytes  flags=0x1D10  CODE
12  0x0AD180-0x0B6B43   39364 bytes  flags=0x1D10  CODE
13  0x0BA640-0x0C658C   48973 bytes  flags=0x1D10  CODE
14  0x0CB740-0x0CF5E6   16039 bytes  flags=0x1D10  CODE
15  0x0D0FC0-0x0DE2E9   54058 bytes  flags=0x1D10  CODE
16  0x0E3880-0x0EB16F   30960 bytes  flags=0x1D10  CODE
17  0x0ED500-0x0F4E82   31107 bytes  flags=0x1D10  CODE
18  0x0F7E40-0x100513   34516 bytes  flags=0x1D10  CODE
19  0x104240-0x10F43D   45566 bytes  flags=0x1D10  CODE
20  0x112D40-0x11B547   34824 bytes  flags=0x1D10  CODE
21  0x120D80-0x125084   17157 bytes  flags=0x1D10  CODE
22  0x126A00-0x13540D   59918 bytes  flags=0x1D10  CODE
23  0x139480-0x142FFF   39808 bytes  flags=0x1D10  CODE
24  0x145E00-0x14D213   29716 bytes  flags=0x1D10  CODE
25  0x14F040-0x152D68   15657 bytes  flags=0x1D10  CODE
26  0x153A40-0x160808   52681 bytes  flags=0x1D10  CODE
27  0x165400-0x16A1B4   19893 bytes  flags=0x1D10  CODE
28  0x16BC80-0x17AA43   60868 bytes  flags=0x1D10  CODE
29  0x180140-0x18CAD0   51601 bytes  flags=0x1D10  CODE
30  0x1916C0-0x19E7EE   53551 bytes  flags=0x1D10  CODE
31  0x1A44C0-0x1A8928   17513 bytes  flags=0x1D10  CODE
32  0x1A96C0-0x1AD284   15301 bytes  flags=0x1D10  CODE
33  0x1AEE80-0x1B29BA   15163 bytes  flags=0x1D10  CODE
34  0x1B2EC0-0x1BAD7B   32444 bytes  flags=0x1D10  CODE
35  0x1BD840-0x1C9B75   49974 bytes  flags=0x1D10  CODE
36  0x1CDB80-0x1D175C   15325 bytes  flags=0x1D10  CODE
37  0x1D28C0-0x1E0980   57537 bytes  flags=0x1D10  CODE
38  0x1E7B00-0x1EBCF7   16888 bytes  flags=0x1D10  CODE
39  0x1EEF00-0x1FB71F   51232 bytes  flags=0x1D10  CODE
40  0x1FFB80-0x204F65   21478 bytes  flags=0x1D10  CODE
41  0x206B80-0x20C7CF   23632 bytes  flags=0x1D10  CODE
42  0x20E380-0x212278   16121 bytes  flags=0x1D10  CODE
43  0x213300-0x217B73   18548 bytes  flags=0x1D10  CODE
44  0x219B40-0x21DA47   16136 bytes  flags=0x1D10  CODE
45  0x21F980-0x227495   31510 bytes  flags=0x1D10  CODE
46  0x228FC0-0x2385BF   62976 bytes  flags=0x1D10  CODE
47  0x23E040-0x243A83   23108 bytes  flags=0x1D10  CODE
48  0x245C40-0x251030   46065 bytes  flags=0x1D10  CODE
49  0x2563C0-0x25DD8F   31184 bytes  flags=0x1D10  CODE
50  0x25FB00-0x26378D   15502 bytes  flags=0x1D10  CODE
51  0x264640-0x2680F3   15028 bytes  flags=0x1D10  CODE
52  0x268640-0x26C786   16711 bytes  flags=0x1D10  CODE
53  0x26CCC0-0x270A77   15800 bytes  flags=0x1D10  CODE
54  0x271FC0-0x27658F   17872 bytes  flags=0x1D10  CODE
55  0x2784C0-0x2854B5   53238 bytes  flags=0x1D10  CODE
56  0x288F80-0x28CCF6   15735 bytes  flags=0x1D10  CODE
57  0x28E800-0x29239A   15259 bytes  flags=0x1D10  CODE
58  0x292BC0-0x29EA90   48849 bytes  flags=0x1D10  CODE
59  0x2A5B00-0x2A9AF3   16372 bytes  flags=0x1D10  CODE
60  0x2AAEC0-0x2B35F1   34610 bytes  flags=0x1D10  CODE
61  0x2BD900-0x2C18D4   16341 bytes  flags=0x1D10  CODE
62  0x2C3180-0x2C6E90   15633 bytes  flags=0x1D10  CODE
63  0x2C7340-0x2CA411   12498 bytes  flags=0x1D10  CODE
64  0x2CB180-0x2CD7EF    9840 bytes  flags=0x1D10  CODE
65  0x2D0080-0x2D398B   14604 bytes  flags=0x1D10  CODE
66  0x2D6F80-0x2DB355   17366 bytes  flags=0x1D10  CODE
67  0x2DC340-0x2DFAAE   14191 bytes  flags=0x1D10  CODE
68  0x2E1A80-0x2EB54F   39632 bytes  flags=0x1D10  CODE
69  0x2EB6C0-0x2EDF35   10358 bytes  flags=0x1D10  CODE
70  0x000900-0x004178   14457 bytes  flags=0x1D50  CODE
71  0x0043C0-0x007380   12225 bytes  flags=0x1D50  CODE
72  0x007400-0x00993F    9536 bytes  flags=0x1D50  CODE
73  0x009A00-0x012115   34582 bytes  flags=0x1D50  CODE
74  0x012280-0x0194AD   29230 bytes  flags=0x1D50  CODE
75  0x019740-0x01BA0B    8908 bytes  flags=0x1D50  CODE
76  0x01BAC0-0x022972   28339 bytes  flags=0x1D50  CODE
77  0x022C00-0x02A6A9   31402 bytes  flags=0x1D50  CODE
78  0x02A9C0-0x02FB83   20932 bytes  flags=0x1D50  CODE
79  0x02FC80-0x032748   10953 bytes  flags=0x0D50  CODE
80  0x032A00-0x035019    9754 bytes  flags=0x0D50  CODE
81  0x035700-0x0378E3    8676 bytes  flags=0x0D59  DATA

========================================
       UBICACION DE LOS MANEJADORES
========================================

CalcularClick
  archivo 0x04B36C (decimal 308076) -> segmento 3 +0x00AC
  archivo 0x2F46AB (decimal 3098283) -> FUERA DE SEGMENTOS

ArmonicoChange
  archivo 0x04B720 (decimal 309024) -> segmento 3 +0x0460
  archivo 0x2F4C19 (decimal 3099673) -> FUERA DE SEGMENTOS

ZodiacoChange
  archivo 0x04B55A (decimal 308570) -> segmento 3 +0x029A
  archivo 0x2F4A33 (decimal 3099187) -> FUERA DE SEGMENTOS
  archivo 0x2F4A49 (decimal 3099209) -> FUERA DE SEGMENTOS
  archivo 0x2F4B20 (decimal 3099424) -> FUERA DE SEGMENTOS

DHGoHarmClick
  archivo 0x04B668 (decimal 308840) -> segmento 3 +0x03A8
  archivo 0x2EEBD9 (decimal 3075033) -> FUERA DE SEGMENTOS

FormCreate
  archivo 0x039E7C (decimal 237180) -> segmento 1 +0x253C
  archivo 0x03CB20 (decimal 248608) -> segmento 2 +0x00E0
  archivo 0x04B3B1 (decimal 308145) -> segmento 3 +0x00F1
  archivo 0x0529EE (decimal 338414) -> segmento 4 +0x00AE
  archivo 0x1528AC (decimal 1386668) -> segmento 25 +0x386C
  archivo 0x2EE479 (decimal 3073145) -> FUERA DE SEGMENTOS
  archivo 0x2EEADA (decimal 3074778) -> FUERA DE SEGMENTOS
  archivo 0x2F8799 (decimal 3114905) -> FUERA DE SEGMENTOS
  archivo 0x2FAF76 (decimal 3125110) -> FUERA DE SEGMENTOS
  archivo 0x2FEE61 (decimal 3141217) -> FUERA DE SEGMENTOS

FormResize
  archivo 0x03CB11 (decimal 248593) -> segmento 2 +0x00D1
  archivo 0x04B37E (decimal 308094) -> segmento 3 +0x00BE
  archivo 0x052AA3 (decimal 338595) -> segmento 4 +0x0163
  archivo 0x28E8C1 (decimal 2681025) -> segmento 57 +0x00C1
  archivo 0x2EE48E (decimal 3073166) -> FUERA DE SEGMENTOS
  archivo 0x2EEAC5 (decimal 3074757) -> FUERA DE SEGMENTOS
  archivo 0x2F87C1 (decimal 3114945) -> FUERA DE SEGMENTOS
  archivo 0x2FE193 (decimal 3137939) -> FUERA DE SEGMENTOS

UseHelioClick
  archivo 0x04B38D (decimal 308109) -> segmento 3 +0x00CD
  archivo 0x2F455A (decimal 3097946) -> FUERA DE SEGMENTOS

Sinastra1Click
  archivo 0x04BA2B (decimal 309803) -> segmento 3 +0x076B
  archivo 0x2F6DC1 (decimal 3108289) -> FUERA DE SEGMENTOS
  archivo 0x2F82FD (decimal 3113725) -> FUERA DE SEGMENTOS

AjustarArmonicas1Click
  archivo 0x04B97E (decimal 309630) -> segmento 3 +0x06BE
  archivo 0x2F7E22 (decimal 3112482) -> FUERA DE SEGMENTOS

========================================
             RESULTADO CLAVE
========================================

Las apariciones que caen dentro de segmentos NE
son las que debemos investigar para encontrar
las referencias utilizadas por el codigo Delphi.

Las apariciones alrededor de 0x2Fxxxx pertenecen
a los recursos TPF0 y no son el codigo del metodo.
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Excelente. **Ya tenemos el dato decisivo**: las apariciones de los eventos del `TForm1` dentro de los segmentos NE están concentradas en el **segmento 3**.


Por ejemplo:


```
CalcularClick             segmento 3 + 0x00AC
ZodiacoChange             segmento 3 + 0x029A
DHGoHarmClick             segmento 3 + 0x03A8
ArmonicoChange            segmento 3 + 0x0460
AjustarArmonicas1Click    segmento 3 + 0x06BE
Sinastra1Click            segmento 3 + 0x076B
```


Eso es muy significativo.


Sin embargo, **todavía no debemos concluir que esas posiciones sean el inicio de las funciones**. Son posiciones donde aparecen las cadenas. Ahora tenemos que inspeccionar el código alrededor de ellas y comprobar cómo Delphi las utiliza.


Además, hay una anomalía en los segmentos 70–81: aparecen con offsets de archivo `0x900`, `0x43C0`, etc., que se solapan con la zona inicial del EXE. **No los vamos a utilizar todavía**; primero trabajaremos con los segmentos 1–69, que presentan una estructura coherente.


El siguiente paso es extraer el **segmento 3 completo** y desensamblarlo como x86 de 16 bits.


Ejecuta todo en una sola caja:


```
Bashcd /storage/emulated/0/Download/armon

mkdir -p ~/armon_code

python3 - <<'PY'
import struct
from pathlib import Path

EXE = "ARMON.EXE"
OUT = Path.home() / "armon_code"

with open(EXE, "rb") as f:
    data = f.read()

ne = struct.unpack_from("<I", data, 0x3c)[0]

def W(o):
    return struct.unpack_from("<H", data, ne + o)[0]

seg_table = W(0x22)
shift = W(0x32)
sector_size = 1 << shift

# Segmento 3
seg_no = 3
p = ne + seg_table + (seg_no - 1) * 8

sector, length, flags, minalloc = struct.unpack_from(
    "<HHHH", data, p
)

offset = sector * sector_size
size = 0x10000 if length == 0 else length

segment = data[offset:offset + size]

outfile = OUT / "segmento_03.bin"

with open(outfile, "wb") as f:
    f.write(segment)

print("===== SEGMENTO 3 =====")
print(f"Offset archivo : 0x{offset:X}")
print(f"Tamaño         : 0x{size:X} ({size} bytes)")
print(f"Flags          : 0x{flags:04X}")
print(f"Extraido       : {outfile}")
PY

echo
echo "===== HERRAMIENTAS DE DESENSAMBLADO DISPONIBLES ====="
command -v objdump || true
command -v ndisasm || true
command -v llvm-objdump || true

echo
echo "===== CONTEXTO BINARIO DE LOS MANEJADORES ====="

python3 - <<'PY'
from pathlib import Path

f = Path.home() / "armon_code" / "segmento_03.bin"
data = f.read_bytes()

objetivos = [
    "CalcularClick",
    "ZodiacoChange",
    "DHGoHarmClick",
    "ArmonicoChange",
    "AjustarArmonicas1Click",
    "Sinastra1Click",
    "UseHelioClick",
]

print("Segmento 3:", len(data), "bytes")

for nombre in objetivos:
    b = nombre.encode("latin1")
    pos = data.find(b)

    print()
    print("-----", nombre, "-----")

    if pos < 0:
        print("NO encontrado")
        continue

    print(f"Offset dentro del segmento: 0x{pos:04X}")
    print(f"Offset archivo original : 0x{0x4B2C0 + pos:X}")

    ini = max(0, pos - 32)
    fin = min(len(data), pos + len(b) + 32)

    bloque = data[ini:fin]

    for i in range(0, len(bloque), 16):
        parte = bloque[i:i+16]
        direccion = ini + i
        hexpart = " ".join(f"{x:02X}" for x in parte)
        asci = "".join(chr(x) if 32 <= x < 127 else "." for x in parte)
        print(f"{direccion:04X}: {hexpart:<47}  {asci}")
PY
```


### Qué buscamos ahora


Si `ndisasm` está disponible, después podremos hacer:


```
segmento_03.bin
        ↓
desensamblado 16-bit
        ↓
0x00AC  CalcularClick
0x029A  ZodiacoChange
0x03A8  DHGoHarmClick
0x0460  ArmonicoChange
...
```


Pero primero quiero ver **los bytes reales alrededor de esas cadenas**. Eso nos permitirá determinar si estamos ante:


- referencias a cadenas;
- tablas de métodos;
- punteros FAR;
- datos RTTI;
- o realmente código ejecutable.


Ese análisis es importante porque nos puede llevar a reconstruir la relación:


**`TForm1` → evento Delphi → dirección de procedimiento → código ensamblador → pseudocódigo Pascal.**

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ mkdir -p ~/armon_code
.EXE"
OUT = Path.home() / "armon_code"

with open(EXE, "rb") as f:
    data = f.read()

ne = struct.unpack_from("<I", data, 0x3c)[0]

def W(o):
    return struct.unpack_from("<H", data, ne + o)[0]

seg_table = W(0x22)
shift = W(0x32)
sector_size = 1 << shift

# Segmento 3
seg_no = 3
p = ne + seg_table + (seg_no - 1) * 8

sector, length, flags, minalloc = struct.unpack_from(
    "<HHHH", data, p
)

offset = sector * sector_size
size = 0x10000 if length == 0 else length

segment = data[offset:offset + size]

outfile = OUT / "segmento_03.bin"

with open(outfile, "wb") as f:
    f.write(segment)

print("===== SEGMENTO 3 =====")
print(f"Offset archivo : 0x{offset:X}")
print(f"Tamaño         : 0x{size:X} ({size} bytes)")
print(f"Flags          : 0x{flags:04X}")
print(f"Extraido       : {outfile}")
PY

echo
echo "===== HERRAMIENTAS DE DESENSAMBLADO DISPONIBLES ====="
command -v objdump || true
command -v ndisasm || true
command -v llvm-objdump || true

echo
echo "===== CONTEXTO BINARIO DE LOS MANEJADORES ====="

python3 - <<'PY'
from pathlib import Path

f = Path.home() / "armon_code" / "segmento_03.bin"
data = f.read_bytes()

objetivos = [
    "CalcularClick",
    "ZodiacoChange",
    "DHGoHarmClick",
    "ArmonicoChange",
    "AjustarArmonicas1Click",
    "Sinastra1Click",
    "UseHelioClick",
]

print("Segmento 3:", len(data), "bytes")

for nombre in objetivos:
    b = nombre.encode("latin1")
    pos = data.find(b)

    print()
    print("-----", nombre, "-----")

    if pos < 0:
        print("NO encontrado")
        continue

    print(f"Offset dentro del segmento: 0x{pos:04X}")
    print(f"Offset archivo original : 0x{0x4B2C0 + pos:X}")

    ini = max(0, pos - 32)
    fin = min(len(data), pos + len(b) + 32)

    bloque = data[ini:fin]

    for i in range(0, len(bloque), 16):
        parte = bloque[i:i+16]
        direccion = ini + i
        hexpart = " ".join(f"{x:02X}" for x in parte)
        asci = "".join(chr(x) if 32 <= x < 127 else "." for x in parte)
        print(f"{direccion:04X}: {hexpart:<47}  {asci}")
PY.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> import struct
> from pathlib import Path
>
> EXE = "ARMON.EXE"
> OUT = Path.home() / "armon_code"
>
> with open(EXE, "rb") as f:
>     data = f.read()
>
> ne = struct.unpack_from("<I", data, 0x3c)[0]
>
> def W(o):
>     return struct.unpack_from("<H", data, ne + o)[0]
>
> seg_table = W(0x22)
> shift = W(0x32)
> sector_size = 1 << shift
>
> # Segmento 3
> seg_no = 3
> p = ne + seg_table + (seg_no - 1) * 8
>
> sector, length, flags, minalloc = struct.unpack_from(
>     "<HHHH", data, p
> )
>
> offset = sector * sector_size
> size = 0x10000 if length == 0 else length
>
> segment = data[offset:offset + size]
>
> outfile = OUT / "segmento_03.bin"
>
> with open(outfile, "wb") as f:
>     f.write(segment)
>
> print("===== SEGMENTO 3 =====")
> print(f"Offset archivo : 0x{offset:X}")
> print(f"Tamaño         : 0x{size:X} ({size} bytes)")
> print(f"Flags          : 0x{flags:04X}")
> print(f"Extraido       : {outfile}")
> PY
===== SEGMENTO 3 =====
Offset archivo : 0x4B2C0
Tamaño         : 0x6ED0 (28368 bytes)
Flags          : 0x1D10
Extraido       : /data/data/com.termux/files/home/armon_code/segmento_03.bin
.../Download/armon $
.../Download/armon $ echo

.../Download/armon $ echo "===== HERRAMIENTAS DE DESENSAMBLADO DISPONIBLES ====="
===== HERRAMIENTAS DE DESENSAMBLADO DISPONIBLES =====
.../Download/armon $ command -v objdump || true
/data/data/com.termux/files/usr/bin/objdump
.../Download/armon $ command -v ndisasm || true
.../Download/armon $ command -v llvm-objdump || true
/data/data/com.termux/files/usr/bin/llvm-objdump
.../Download/armon $
.../Download/armon $ echo

.../Download/armon $ echo "===== CONTEXTO BINARIO DE LOS MANEJADORES ====="
===== CONTEXTO BINARIO DE LOS MANEJADORES =====
.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> from pathlib import Path
>
> f = Path.home() / "armon_code" / "segmento_03.bin"
> data = f.read_bytes()
>
> objetivos = [
>     "CalcularClick",
>     "ZodiacoChange",
>     "DHGoHarmClick",
>     "ArmonicoChange",
>     "AjustarArmonicas1Click",
>     "Sinastra1Click",
>     "UseHelioClick",
> ]
>
> print("Segmento 3:", len(data), "bytes")
>
> for nombre in objetivos:
>     b = nombre.encode("latin1")
>     pos = data.find(b)
>
>     print()
>     print("-----", nombre, "-----")
>
>     if pos < 0:
>         print("NO encontrado")
>         continue
>
>     print(f"Offset dentro del segmento: 0x{pos:04X}")
>     print(f"Offset archivo original : 0x{0x4B2C0 + pos:X}")
>
>     ini = max(0, pos - 32)
>     fin = min(len(data), pos + len(b) + 32)
>
>     bloque = data[ini:fin]
>
>     for i in range(0, len(bloque), 16):
>         parte = bloque[i:i+16]
>         direccion = ini + i
>         hexpart = " ".join(f"{x:02X}" for x in parte)
>         asci = "".join(chr(x) if 32 <= x < 127 else "." for x in parte)
>         print(f"{direccion:04X}: {hexpart:<47}  {asci}")
> PY
Segmento 3: 28368 bytes

----- CalcularClick -----
Offset dentro del segmento: 0x00AC
Offset archivo original : 0x4B36C
008C: 54 00 D6 31 7C 00 2C 47 3C 00 61 5F 48 00 80 57  T..1|.,G<.a_H..W
009C: 10 00 06 54 46 6F 72 6D 31 68 00 47 1B BB 00 0D  ...TForm1h.G....
00AC: 43 61 6C 63 75 6C 61 72 43 6C 69 63 6B C3 1D CA  CalcularClick...
00BC: 00 0A 46 6F 72 6D 52 65 73 69 7A 65 FA 1D DC 00  ..FormResize....
00CC: 0D 55 73 65 48 65 6C 69 6F 43 6C 69 63           .UseHelioClic

----- ZodiacoChange -----
Offset dentro del segmento: 0x029A
Offset archivo original : 0x4B55A
027A: 42 43 65 6E 74 55 70 43 6C 69 63 6B B7 2E 97 02  BCentUpClick....
028A: 0A 41 68 6F 72 61 43 6C 69 63 6B 6D 2F A9 02 0D  .AhoraClickm/...
029A: 5A 6F 64 69 61 63 6F 43 68 61 6E 67 65 F8 3C C2  ZodiacoChange.<.
02AA: 02 14 41 72 63 68 69 76 61 72 44 69 62 75 6A 6F  ..ArchivarDibujo
02BA: 31 43 6C 69 63 6B 2D 3E E1 02 1A 50 6C           1Click->...Pl

----- DHGoHarmClick -----
Offset dentro del segmento: 0x03A8
Offset archivo original : 0x4B668
0388: 43 6C 69 63 6B 0B 40 A5 03 11 48 61 72 6D 6F 67  Click.@...Harmog
0398: 72 61 6D 61 73 31 43 6C 69 63 6B E8 40 B7 03 0D  ramas1Click.@...
03A8: 44 48 47 6F 48 61 72 6D 43 6C 69 63 6B A9 41 C8  DHGoHarmClick.A.
03B8: 03 0C 44 48 4D 53 61 76 65 43 6C 69 63 6B 1B 43  ..DHMSaveClick.C
03C8: D9 03 0C 44 48 4D 4F 74 72 6F 43 6C 69           ...DHMOtroCli

----- ArmonicoChange -----
Offset dentro del segmento: 0x0460
Offset archivo original : 0x4B720
0440: 61 52 75 65 64 61 31 43 6C 69 63 6B 1F 64 5D 04  aRueda1Click.d].
0450: 0A 55 73 65 41 52 43 6C 69 63 6B 23 47 70 04 0E  .UseARClick#Gp..
0460: 41 72 6D 6F 6E 69 63 6F 43 68 61 6E 67 65 80 47  ArmonicoChange.G
0470: 87 04 12 53 70 69 6E 42 75 74 74 6F 6E 31 55 70  ...SpinButton1Up
0480: 43 6C 69 63 6B DD 47 A0 04 14 53 70 69 6E        Click.G...Spin

----- AjustarArmonicas1Click -----
Offset dentro del segmento: 0x06BE
Offset archivo original : 0x4B97E
069E: 6B CB 5E BB 06 15 43 61 6C 63 75 6C 61 72 53 69  k.^...CalcularSi
06AE: 65 6D 70 72 65 31 43 6C 69 63 6B 22 5F D6 06 16  empre1Click"_...
06BE: 41 6A 75 73 74 61 72 41 72 6D 6F 6E 69 63 61 73  AjustarArmonicas
06CE: 31 43 6C 69 63 6B 96 5F F0 06 15 41 72 6D 6E 69  1Click._...Armni
06DE: 63 6F 73 41 63 74 69 76 6F 73 31 43 6C 69 63 6B  cosActivos1Click
06EE: E9 58 05 07 10 52                                .X...R

----- Sinastra1Click -----
Offset dentro del segmento: 0x076B
Offset archivo original : 0x4BA2B
074B: 07 19 49 6E 64 69 63 61 64 6F 72 70 61 72 61 47  ..IndicadorparaG
075B: 72 69 73 65 73 31 43 6C 69 63 6B 54 63 7B 07 0E  rises1ClickTc{..
076B: 53 69 6E 61 73 74 72 61 31 43 6C 69 63 6B 93 63  Sinastra1Click.c
077B: 8D 07 0D 4C 75 67 61 72 44 62 6C 43 6C 69 63 6B  ...LugarDblClick
078B: E3 63 B0 07 1E 4F 70 63 69 6F 6E 65 73 70        .c...Opcionesp

----- UseHelioClick -----
Offset dentro del segmento: 0x00CD
Offset archivo original : 0x4B38D
00AD: 61 6C 63 75 6C 61 72 43 6C 69 63 6B C3 1D CA 00  alcularClick....
00BD: 0A 46 6F 72 6D 52 65 73 69 7A 65 FA 1D DC 00 0D  .FormResize.....
00CD: 55 73 65 48 65 6C 69 6F 43 6C 69 63 6B 1A 1E EE  UseHelioClick...
00DD: 00 0D 55 73 65 53 69 64 65 72 43 6C 69 63 6B 55  ..UseSiderClickU
00ED: 24 FD 00 0A 46 6F 72 6D 43 72 65 61 74           $...FormCreat
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Ahora sí tenemos una pista mucho más fuerte. **Me corrijo respecto al paso anterior:** no debemos tratar `CalcularClick` como una simple cadena aislada.


El patrón:


```
0D CalcularClick C3 1D CA 00
0A FormResize   FA 1D DC 00
0D UseHelioClick 1A 1E EE 00
```


muestra que **después del nombre hay cuatro bytes estructurados**. Además, los offsets de esos cuatro bytes avanzan de forma muy coherente:


```
CalcularClick       → 0x1DC3
FormResize          → 0x1DFA
UseHelioClick       → 0x1E1A
UseSiderClick       → 0x1E55
```


Esto es exactamente el tipo de información que necesitamos para recuperar las direcciones de los métodos. Delphi utiliza tablas internas/VMT para resolver nombres de métodos publicados hacia sus direcciones.


La cuestión que debemos resolver ahora es **qué significan exactamente los otros dos bytes** (`CA 00`, `DC 00`, `EE 00`, etc.) y cómo se relacionan con los segmentos NE.


No voy a asumirlo todavía.


Vamos a hacer una prueba mucho más precisa: extraer **toda la tabla que empieza alrededor de `TForm1`**, interpretar cada registro como:


```
[longitud][nombre][4 bytes de dirección]
```


y comparar los valores obtenidos con los 81 segmentos NE.


Ejecuta todo en **una sola caja**:


```
Bashcd /storage/emulated/0/Download/armon

python3 - <<'PY'
import struct

EXE = "ARMON.EXE"

with open(EXE, "rb") as f:
    data = f.read()

# Segmento 3
SEG_OFF = 0x4B2C0
SEG_SIZE = 0x6ED0
seg = data[SEG_OFF:SEG_OFF + SEG_SIZE]

# La tabla comienza justo antes de TForm1.
start = 0x009C

print("============================================================")
print(" TABLA DE METODOS / RTTI EN EL SEGMENTO 3")
print("============================================================")
print()
print(f"Inicio: segmento+0x{start:04X}")
print()

p = start

for i in range(100):

    if p >= len(seg):
        break

    length = seg[p]

    # Fin de datos razonables
    if length == 0:
        print(f"[{p:04X}] FIN")
        break

    if length > 80 or p + 1 + length + 4 > len(seg):
        print(f"[{p:04X}] longitud sospechosa: {length}")
        break

    name_bytes = seg[p + 1:p + 1 + length]

    try:
        name = name_bytes.decode("cp1252")
    except:
        name = repr(name_bytes)

    addr_pos = p + 1 + length
    raw = seg[addr_pos:addr_pos + 4]

    value = struct.unpack_from("<I", seg, addr_pos)[0]
    offset = struct.unpack_from("<H", seg, addr_pos)[0]
    second = struct.unpack_from("<H", seg, addr_pos + 2)[0]

    print(
        f"{p:04X}  "
        f"{length:02X}  "
        f"{name:<45}  "
        f"{raw.hex(' ')}  "
        f"offset=0x{offset:04X}  "
        f"word2=0x{second:04X}"
    )

    p = addr_pos + 4
PY

echo
echo "============================================================"
echo " COMPROBACION DE LOS OFFSETS DE METODO"
echo "============================================================"

python3 - <<'PY'
import struct

EXE = "ARMON.EXE"

with open(EXE, "rb") as f:
    data = f.read()

SEG_OFF = 0x4B2C0
SEG_SIZE = 0x6ED0
seg = data[SEG_OFF:SEG_OFF + SEG_SIZE]

start = 0x009C
p = start

for i in range(100):

    if p >= len(seg):
        break

    n = seg[p]

    if n == 0 or n > 80 or p + 1 + n + 4 > len(seg):
        break

    name = seg[p+1:p+1+n].decode("cp1252", errors="replace")

    a = p + 1 + n

    off = struct.unpack_from("<H", seg, a)[0]
    segword = struct.unpack_from("<H", seg, a+2)[0]

    # Si el primer WORD parece un offset dentro del segmento 3
    if off < SEG_SIZE:

        archivo = SEG_OFF + off

        # Mostrar los primeros bytes del posible procedimiento
        code = data[archivo:archivo+12]

        print(
            f"{name:<45} "
            f"segword=0x{segword:04X} "
            f"offset=0x{off:04X} "
            f"archivo=0x{archivo:06X} "
            f"bytes={code.hex(' ')}"
        )

    p = a + 4
PY
```


### Lo que espero que nos revele


Si encontramos algo como:


```
CalcularClick       ... offset=0x1DC3 ... bytes=55 8B EC ...
FormResize          ... offset=0x1DFA ... bytes=55 8B EC ...
UseHelioClick       ... offset=0x1E1A ... bytes=55 8B EC ...
```


entonces habremos encontrado directamente los **puntos de entrada de los procedimientos Delphi**.


Y si los bytes no comienzan así, no pasa nada: los analizaremos como código 8086/80186 de 16 bits.


El objetivo inmediato ya no es simplemente «desensamblar el segmento 3». Es mucho mejor:


**nombre Delphi → registro RTTI → offset del método → bytes reales del procedimiento.**


Ese sería el primer puente sólido hacia la reconstrucción del código fuente original.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> import struct
>
> EXE = "ARMON.EXE"
>
> with open(EXE, "rb") as f:
>     data = f.read()
>
> # Segmento 3
> SEG_OFF = 0x4B2C0
> SEG_SIZE = 0x6ED0
> seg = data[SEG_OFF:SEG_OFF + SEG_SIZE]
>
> # La tabla comienza justo antes de TForm1.
> start = 0x009C
==>
> print("============================================================")
> print(" TABLA DE METODOS / RTTI EN EL SEGMENTO 3")
> print("============================================================")
> print()
> print(f"Inicio: segmento+0x{start:04X}")
> print()
>
> p = start
>
> for i in range(100):
>
>     if p >= len(seg):
>         break
>
>     length = seg[p]
>
>     # Fin de datos razonables
>     if length == 0:
>         print(f"[{p:04X}] FIN")
>         break
>
>     if length > 80 or p + 1 + length + 4 > len(seg):
>         print(f"[{p:04X}] longitud sospechosa: {length}")
>         break
>
>     name_bytes = seg[p + 1:p + 1 + length]
>
>     try:
>         name = name_bytes.decode("cp1252")
>     except:
>         name = repr(name_bytes)
>
>     addr_pos = p + 1 + length
>     raw = seg[addr_pos:addr_pos + 4]
IZE:

        archivo = SEG_OFF + o>
ff
                                                       ]
ible p>     offset = struct.unpack_from("<H", seg, addr_pos)[0]
>     second = struct.unpack_from("<H", seg, addr_pos + 2)[0]
>
>     print(
      >         f"{p:04X}  "
  p = a>         f"{length:02X}  "
>         f"{name:<45}  "
>         f"{raw.hex(' ')}  "
>         f"offset=0x{offset:04X}  "
>         f"word2=0x{second:04X}"
>     )
>
>     p = addr_pos + 4
> PY
============================================================
 TABLA DE METODOS / RTTI EN EL SEGMENTO 3
============================================================

Inicio: segmento+0x009C

C                               61 6c 63 75  offset=0x6C61  word2=0x7563
[00B1] longitud sospechosa: 108
.../Download/armon $
.../Download/armon $ echo

.../Download/armon $ echo "============================================================"
============================================================
.../Download/armon $ echo " COMPROBACION DE LOS OFFSETS DE METODO"
 COMPROBACION DE LOS OFFSETS DE METODO
.../Download/armon $ echo "============================================================"
============================================================
.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> import struct
>
> EXE = "ARMON.EXE"
>
> with open(EXE, "rb") as f:
>     data = f.read()
>
> SEG_OFF = 0x4B2C0
> SEG_SIZE = 0x6ED0
> seg = data[SEG_OFF:SEG_OFF + SEG_SIZE]
>
> start = 0x009C
> p = start
>
> for i in range(100):
>
>     if p >= len(seg):
>         break
>
>     n = seg[p]
>
>     if n == 0 or n > 80 or p + 1 + n + 4 > len(seg):
>         break
>
>     name = seg[p+1:p+1+n].decode("cp1252", errors="replace")
>
>     a = p + 1 + n
>
>     off = struct.unpack_from("<H", seg, a)[0]
>     segword = struct.unpack_from("<H", seg, a+2)[0]
>
>     # Si el primer WORD parece un offset dentro del segmento 3
>     if off < SEG_SIZE:
>
>         archivo = SEG_OFF + off
>
>         # Mostrar los primeros bytes del posible procedimiento
>         code = data[archivo:archivo+12]
>
>         print(
>             f"{name:<45} "
>             f"segword=0x{segword:04X} "
>             f"offset=0x{off:04X} "
>             f"archivo=0x{archivo:06X} "
>             f"bytes={code.hex(' ')}"
>         )
>
>     p = a + 4
> PY
C                              segword=0x7563 offset=0x6C61 archivo=0x051F21 bytes=06 25 31 35 2e 37 66 55 89 e5 b8 02
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Esto es importante: **la hipótesis anterior de que `0x009C` era el inicio de una tabla de métodos era incorrecta**.


El resultado lo demuestra:


```
0x009C:
10 00 06 54 46 6F 72 6D 31 68 ...
```


El `10` pertenece a una estructura anterior, y en `0x00A0` empieza realmente `TForm1`. Por eso nuestro parser interpretó la `C` de `CalcularClick` como si fuera una longitud.


Pero hemos obtenido algo todavía más interesante: los bytes que siguen a cada nombre tienen una estructura muy regular.


Por ejemplo:


```
CalcularClick
43 61 6C 63 75 6C 61 72 43 6C 69 63 6B
C3 1D CA 00

FormResize
46 6F 72 6D 52 65 73 69 7A 65
FA 1D DC 00

UseHelioClick
55 73 65 48 65 6C 69 6F 43 6C 69 63 6B
1A 1E EE 00
```


Es decir:


```
[nombre del método][4 bytes]
```


Eso **sí merece ser investigado**, pero todavía no podemos afirmar que esos 4 bytes sean directamente `segmento:offset`.


### El siguiente paso


En vez de inventar la estructura de la tabla, vamos a localizar **automáticamente todos los nombres de eventos y sus cuatro bytes posteriores**, y después comparar esos valores con:


1. la tabla de segmentos NE;
2. las direcciones de entrada;
3. posibles offsets válidos dentro de cada segmento;
4. los bytes que hay realmente en esas posiciones.


Además, esta vez el script localizará los nombres por contenido, sin asumir dónde empieza la tabla.


Ejecuta **todo este bloque único**:


```
Bashcd /storage/emulated/0/Download/armon

python3 - <<'PY'
import struct
from pathlib import Path

EXE = "ARMON.EXE"

with open(EXE, "rb") as f:
    data = f.read()

ne = struct.unpack_from("<I", data, 0x3C)[0]

def W(o):
    return struct.unpack_from("<H", data, ne + o)[0]

seg_table = W(0x22)
shift = W(0x32)
sector_size = 1 << shift
seg_count = W(0x1C)

print("============================================================")
print(" ANALISIS DE LOS REGISTROS POSTERIORES A LOS NOMBRES")
print("============================================================")
print()
print(f"NE              : 0x{ne:X}")
print(f"Segmentos       : {seg_count}")
print(f"Tabla segmentos : 0x{ne + seg_table:X}")
print(f"Tamaño sector   : {sector_size}")
print()

# ------------------------------------------------------------
# Tabla NE de segmentos
# ------------------------------------------------------------

segments = []

for n in range(1, seg_count + 1):

    p = ne + seg_table + (n - 1) * 8

    sector, length, flags, minalloc = struct.unpack_from(
        "<HHHH", data, p
    )

    off = sector * sector_size
    size = 0x10000 if length == 0 else length

    segments.append({
        "num": n,
        "off": off,
        "size": size,
        "flags": flags,
        "minalloc": minalloc
    })

# ------------------------------------------------------------
# Eventos que ya sabemos que pertenecen a TForm1
# ------------------------------------------------------------

objetivos = [
    "CalcularClick",
    "FormResize",
    "UseHelioClick",
    "UseSiderClick",
    "FormCreate",
    "ZodiacoChange",
    "AhoraClick",
    "DHGoHarmClick",
    "DHMSaveClick",
    "DHMOtroClick",
    "ArmonicoChange",
    "UseARClick",
    "AjustarArmonicas1Click",
    "ArmnicosActivos1Click",
    "Sinastra1Click",
    "LugarDblClick",
    "OpcionesparaFlorArmonica1Click",
    "CalcularSiempre1Click",
    "NodosPlanetarios1Click",
    "PlanetasqueseDibujan1Click",
    "PlanetasqueseNumeran1Click",
    "PlanetasqueSeAspectan1Click",
    "PlanetasqueEmitenAspectos1Click",
]

# ------------------------------------------------------------
# Función: determinar si un offset pertenece a un segmento
# ------------------------------------------------------------

def localizar_offset(file_offset):

    candidatos = []

    for s in segments:

        if s["off"] <= file_offset < s["off"] + s["size"]:

            dentro = file_offset - s["off"]

            candidatos.append(
                (
                    s["num"],
                    dentro,
                    s["size"],
                    s["flags"]
                )
            )

    return candidatos

# ------------------------------------------------------------
# Buscar todos los nombres
# ------------------------------------------------------------

for nombre in objetivos:

    b = nombre.encode("cp1252")

    posiciones = []

    inicio = 0

    while True:

        pos = data.find(b, inicio)

        if pos < 0:
            break

        posiciones.append(pos)

        inicio = pos + 1

    print()
    print("------------------------------------------------------------")
    print(nombre)
    print("------------------------------------------------------------")

    if not posiciones:
        print("NO encontrado")
        continue

    for pos in posiciones:

        despues = pos + len(b)

        raw = data[despues:despues + 4]

        if len(raw) < 4:
            continue

        v32 = struct.unpack_from("<I", raw)[0]
        lo = struct.unpack_from("<H", raw)[0]
        hi = struct.unpack_from("<H", raw)[0]

        print()
        print(f"Nombre en archivo : 0x{pos:06X}")
        print(f"4 bytes          : {raw.hex(' ')}")
        print(f"DWORD LE         : 0x{v32:08X}")
        print(f"WORD bajo        : 0x{lo:04X}")
        print(f"WORD alto        : 0x{hi:04X}")

        # ¿El propio registro está dentro de un segmento?
        loc = localizar_offset(pos)

        if loc:

            for segnum, dentro, ssize, flags in loc:
                print(
                    f"Nombre está en   : segmento {segnum} "
                    f"+0x{dentro:04X} "
                    f"(flags 0x{flags:04X})"
                )

        # ----------------------------------------------------
        # Probar ambas interpretaciones posibles
        # ----------------------------------------------------

        print("Interpretaciones:")

        # A: hi:lo
        for etiqueta, segnum, off in [
            ("hi:lo", hi, lo),
            ("lo:hi", lo, hi),
        ]:

            if 1 <= segnum <= len(segments):

                s = segments[segnum - 1]

                if off < s["size"]:

                    fileoff = s["off"] + off

                    code = data[fileoff:fileoff + 16]

                    print(
                        f"  {etiqueta:<6} "
                        f"segmento={segnum:<3} "
                        f"offset=0x{off:04X} "
                        f"archivo=0x{fileoff:06X} "
                        f"bytes={code.hex(' ')}"
                    )

                else:

                    print(
                        f"  {etiqueta:<6} "
                        f"segmento={segnum:<3} "
                        f"offset=0x{off:04X} "
                        f"FUERA DEL SEGMENTO"
                    )

            else:

                print(
                    f"  {etiqueta:<6} "
                    f"segmento={segnum} "
                    f"NO EXISTE"
                )

PY
```


### Qué buscamos exactamente


Hay tres posibilidades principales.


**Caso A — encontramos algo como:**


```
segmento=3
offset=0x....
bytes=55 8B ...
```


Entonces tenemos una pista muy fuerte de que acabamos de encontrar código de una rutina Delphi.


**Caso B — aparecen otros segmentos pero offsets válidos:**


Eso sería incluso más interesante, porque significaría que los métodos están distribuidos entre diferentes segmentos NE.


**Caso C — ninguna interpretación produce posiciones coherentes:**


Entonces esos cuatro bytes probablemente **no son una dirección segmentada directa**, y podremos investigar su significado sin destruir la estructura que ya tenemos.


Lo importante es que **ya no vamos a hacer suposiciones sobre el formato TPF0**. Vamos a dejar que las relaciones entre los datos del ejecutable nos indiquen la estructura real.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> import struct
> from pathlib import Path
>
> EXE = "ARMON.EXE"
>
> with open(EXE, "rb") as f:
>     data = f.read()
>
> ne = struct.unpack_from("<I", data, 0x3C)[0]
>
> def W(o):
>     return struct.unpack_from("<H", data, ne + o)[0]
>
> seg_table = W(0x22)
> shift = W(0x32)
> sector_size = 1 << shift
> seg_count = W(0x1C)
>
> print("============================================================")
m1
# -------------------------------------------> print(" ANALISIS DE LOS REGISTROS POSTERIORES A LOS NOMBRES")
> print("============================================================")
> print()
> print(f"NE              : 0x{ne:X}")
> print(f"Segmentos       : {seg_count}")
> print(f"Tabla segmentos : 0x{ne + seg_table:X}")
anetas> print(f"Tamaño sector   : {sector_size}")
> print()
>
> # ------------------------------------------------------------
> # Tabla NE de segmentos
         s["siz> # ------------------------------------------------------------
eturn>
 can> segments = []
>
-----> for n in range(1, seg_count + 1):
------------
# Buscar todo>
s los nom>     p = ne + seg_table + (n - 1) * 8
>
------------------------->     sector, length, flags, minalloc = struct.unpack_from(
>         "<HHHH", data, p
 = []

    inicio = 0

    while True:

    >     )
 data.>
find(b, inici>     off = sector * sector_size
   p>     size = 0x10000 if length == 0 else length
int()
  >
>     segments.append({
>         "num": n,
>         "off": off,
>         "size": size,
>         "flags": flags,
>         "minalloc": minalloc
>     })
>
> # ------------------------------------------------------------
unpack_fr> # Eventos que ya sabemos que pertenecen a TForm1
> # ------------------------------------------------------------
>
    > objetivos = [
>     "CalcularClick",
>     "FormResize",
>     "UseHelioClick",
>     "UseSiderClick",
>     "FormCreate",
>     "ZodiacoChange",
>     "AhoraClick",
>     "DHGoHarmClick",
>     "DHMSaveClick",
>     "DHMOtroClick",
>     "ArmonicoChange",
>     "UseARClick",
>     "AjustarArmonicas1Click",
f>     "ArmnicosActivos1Click",
>     "Sinastra1Click",
>     "LugarDblClick",
>     "OpcionesparaFlorArmonica1Click",
>     "CalcularSiempre1Click",
>     "NodosPlanetarios1Click",
>     "PlanetasqueseDibujan1Click",
erpretaciones>     "PlanetasqueseNumeran1Click",
-->     "PlanetasqueSeAspectan1Click",
nterpretac>     "PlanetasqueEmitenAspectos1Click",
> ]
>
> # ------------------------------------------------------------
nts):
> # Función: determinar si un offset pertenece a un segm
ento
> # ------------------------------------------------------------
>
> def localizar_offset(file_offset):
>
>     candidatos = []
>
>     for s in segments:
>
>         if s["off"] <= file_offset < s["off"] + s["size"]:
>
>             dentro = file_offset - s["off"]
>
>             candidatos.append(
SEGMENTO>                 (
"
   >                     s["num"],
"  {e>                     dentro,
   >                     s["size"],
>                     s["flags"]
>                 )
>             )
>
>     return candidatos
>
> # ------------------------------------------------------------
> # Buscar todos los nombres
> # ------------------------------------------------------------
>
> for nombre in objetivos:
>
>     b = nombre.encode("cp1252")
>
>     posiciones = []
>
>     inicio = 0
>
>     while True:
>
>         pos = data.find(b, inicio)
>
>         if pos < 0:
>             break
>
>         posiciones.append(pos)
>
>         inicio = pos + 1
>
>     print()
>     print("------------------------------------------------------------")
>     print(nombre)
>     print("------------------------------------------------------------")
>
>     if not posiciones:
>         print("NO encontrado")
>         continue
>
>     for pos in posiciones:
>
>         despues = pos + len(b)
>
>         raw = data[despues:despues + 4]
>
>         if len(raw) < 4:
>             continue
>
>         v32 = struct.unpack_from("<I", raw)[0]
>         lo = struct.unpack_from("<H", raw)[0]
>         hi = struct.unpack_from("<H", raw)[0]
>
>         print()
>         print(f"Nombre en archivo : 0x{pos:06X}")
>         print(f"4 bytes          : {raw.hex(' ')}")
>         print(f"DWORD LE         : 0x{v32:08X}")
>         print(f"WORD bajo        : 0x{lo:04X}")
>         print(f"WORD alto        : 0x{hi:04X}")
>
>         # ¿El propio registro está dentro de un segmen
to?
>         loc = localizar_offset(pos)
>
>         if loc:
>
>             for segnum, dentro, ssize, flags in loc:
>                 print(
>                     f"Nombre está en   : segmento {seg
num} "
>                     f"+0x{dentro:04X} "
>                     f"(flags 0x{flags:04X})"
>                 )
>
>         # ----------------------------------------------------
>         # Probar ambas interpretaciones posibles
>         # ----------------------------------------------------
>
>         print("Interpretaciones:")
>
>         # A: hi:lo
>         for etiqueta, segnum, off in [
>             ("hi:lo", hi, lo),
>             ("lo:hi", lo, hi),
>         ]:
>
>             if 1 <= segnum <= len(segments):
>
>                 s = segments[segnum - 1]
>
>                 if off < s["size"]:
>
>                     fileoff = s["off"] + off
>
>                     code = data[fileoff:fileoff + 16]
>
>                     print(
>                         f"  {etiqueta:<6} "
>                         f"segmento={segnum:<3} "
>                         f"offset=0x{off:04X} "
>                         f"archivo=0x{fileoff:06X} "
>                         f"bytes={code.hex(' ')}"
>                     )
>
>                 else:
>
>                     print(
>                         f"  {etiqueta:<6} "
>                         f"segmento={segnum:<3} "
>                         f"offset=0x{off:04X} "
>                         f"FUERA DEL SEGMENTO"
>                     )
>
>             else:
>
>                 print(
>                     f"  {etiqueta:<6} "
>                     f"segmento={segnum} "
>                     f"NO EXISTE"
>                 )
>
> PY
============================================================
 ANALISIS DE LOS REGISTROS POSTERIORES A LOS NOMBRES
============================================================

NE              : 0x100
Segmentos       : 81
Tabla segmentos : 0x140
Tamaño sector   : 64


------------------------------------------------------------
CalcularClick
------------------------------------------------------------

Nombre en archivo : 0x04B36C
4 bytes          : c3 1d ca 00
DWORD LE         : 0x00CA1DC3
WORD bajo        : 0x1DC3
WORD alto        : 0x1DC3
Nombre está en   : segmento 3 +0x00AC (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=7619 NO EXISTE
  lo:hi  segmento=7619 NO EXISTE

Nombre en archivo : 0x2F46AB
4 bytes          : 00 00 07 54
DWORD LE         : 0x54070000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE

------------------------------------------------------------
FormResize
------------------------------------------------------------

Nombre en archivo : 0x03CB11
4 bytes          : ab 9c ec 00
DWORD LE         : 0x00EC9CAB
WORD bajo        : 0x9CAB
WORD alto        : 0x9CAB
Nombre está en   : segmento 2 +0x00D1 (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=40107 NO EXISTE
  lo:hi  segmento=40107 NO EXISTE

Nombre en archivo : 0x04B37E
4 bytes          : fa 1d dc 00
DWORD LE         : 0x00DC1DFA
WORD bajo        : 0x1DFA
WORD alto        : 0x1DFA
Nombre está en   : segmento 3 +0x00BE (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=7674 NO EXISTE
  lo:hi  segmento=7674 NO EXISTE

Nombre en archivo : 0x052AA3
4 bytes          : 9a 24 7f 01
DWORD LE         : 0x017F249A
WORD bajo        : 0x249A
WORD alto        : 0x249A
Nombre está en   : segmento 4 +0x0163 (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=9370 NO EXISTE
  lo:hi  segmento=9370 NO EXISTE

Nombre en archivo : 0x28E8C1
4 bytes          : 0e 15 e5 00
DWORD LE         : 0x00E5150E
WORD bajo        : 0x150E
WORD alto        : 0x150E
Nombre está en   : segmento 57 +0x00C1 (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=5390 NO EXISTE
  lo:hi  segmento=5390 NO EXISTE

Nombre en archivo : 0x2EE48E
4 bytes          : 0a 54 65 78
DWORD LE         : 0x7865540A
WORD bajo        : 0x540A
WORD alto        : 0x540A
Interpretaciones:
  hi:lo  segmento=21514 NO EXISTE
  lo:hi  segmento=21514 NO EXISTE

Nombre en archivo : 0x2EEAC5
4 bytes          : 08 4f 6e 43
DWORD LE         : 0x436E4F08
WORD bajo        : 0x4F08
WORD alto        : 0x4F08
Interpretaciones:
  hi:lo  segmento=20232 NO EXISTE
  lo:hi  segmento=20232 NO EXISTE

Nombre en archivo : 0x2F87C1
4 bytes          : 0a 54 65 78
DWORD LE         : 0x7865540A
WORD bajo        : 0x540A
WORD alto        : 0x540A
Interpretaciones:
  hi:lo  segmento=21514 NO EXISTE
  lo:hi  segmento=21514 NO EXISTE

Nombre en archivo : 0x2FE193
4 bytes          : 0a 54 65 78
DWORD LE         : 0x7865540A
WORD bajo        : 0x540A
WORD alto        : 0x540A
Interpretaciones:
  hi:lo  segmento=21514 NO EXISTE
  lo:hi  segmento=21514 NO EXISTE

------------------------------------------------------------
UseHelioClick
------------------------------------------------------------

Nombre en archivo : 0x04B38D
4 bytes          : 1a 1e ee 00
DWORD LE         : 0x00EE1E1A
WORD bajo        : 0x1E1A
WORD alto        : 0x1E1A
Nombre está en   : segmento 3 +0x00CD (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=7706 NO EXISTE
  lo:hi  segmento=7706 NO EXISTE

Nombre en archivo : 0x2F455A
4 bytes          : 00 00 09 54
DWORD LE         : 0x54090000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE

------------------------------------------------------------
UseSiderClick
------------------------------------------------------------

Nombre en archivo : 0x04B39F
4 bytes          : 55 24 fd 00
DWORD LE         : 0x00FD2455
WORD bajo        : 0x2455
WORD alto        : 0x2455
Nombre está en   : segmento 3 +0x00DF (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=9301 NO EXISTE
  lo:hi  segmento=9301 NO EXISTE

Nombre en archivo : 0x2F45F9
4 bytes          : 00 00 07 54
DWORD LE         : 0x54070000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE

------------------------------------------------------------
FormCreate
------------------------------------------------------------

Nombre en archivo : 0x039E7C
4 bytes          : 07 00 9b 25
DWORD LE         : 0x259B0007
WORD bajo        : 0x0007
WORD alto        : 0x0007
Nombre está en   : segmento 1 +0x253C (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=7   offset=0x0007 archivo=0x068D87 bytes=00 b2 c2 00 00 00 3f 00 00 80 3f 35 c2 68 21 a2
  lo:hi  segmento=7   offset=0x0007 archivo=0x068D87 bytes=00 b2 c2 00 00 00 3f 00 00 80 3f 35 c2 68 21 a2

Nombre en archivo : 0x03CB20
4 bytes          : ee 9c 89 01
DWORD LE         : 0x01899CEE
WORD bajo        : 0x9CEE
WORD alto        : 0x9CEE
Nombre está en   : segmento 2 +0x00E0 (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=40174 NO EXISTE
  lo:hi  segmento=40174 NO EXISTE

Nombre en archivo : 0x04B3B1
4 bytes          : 53 28 0b 01
DWORD LE         : 0x010B2853
WORD bajo        : 0x2853
WORD alto        : 0x2853
Nombre está en   : segmento 3 +0x00F1 (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=10323 NO EXISTE
  lo:hi  segmento=10323 NO EXISTE

Nombre en archivo : 0x0529EE
4 bytes          : 0b 1d d3 00
DWORD LE         : 0x00D31D0B
WORD bajo        : 0x1D0B
WORD alto        : 0x1D0B
Nombre está en   : segmento 4 +0x00AE (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=7435 NO EXISTE
  lo:hi  segmento=7435 NO EXISTE

Nombre en archivo : 0x1528AC
4 bytes          : 76 3a 84 38
DWORD LE         : 0x38843A76
WORD bajo        : 0x3A76
WORD alto        : 0x3A76
Nombre está en   : segmento 25 +0x386C (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=14966 NO EXISTE
  lo:hi  segmento=14966 NO EXISTE

Nombre en archivo : 0x2EE479
4 bytes          : 08 4f 6e 52
DWORD LE         : 0x526E4F08
WORD bajo        : 0x4F08
WORD alto        : 0x4F08
Interpretaciones:
  hi:lo  segmento=20232 NO EXISTE
  lo:hi  segmento=20232 NO EXISTE

Nombre en archivo : 0x2EEADA
4 bytes          : 0a 54 65 78
DWORD LE         : 0x7865540A
WORD bajo        : 0x540A
WORD alto        : 0x540A
Interpretaciones:
  hi:lo  segmento=21514 NO EXISTE
  lo:hi  segmento=21514 NO EXISTE

Nombre en archivo : 0x2F8799
4 bytes          : 07 4f 6e 4b
DWORD LE         : 0x4B6E4F07
WORD bajo        : 0x4F07
WORD alto        : 0x4F07
Interpretaciones:
  hi:lo  segmento=20231 NO EXISTE
  lo:hi  segmento=20231 NO EXISTE

Nombre en archivo : 0x2FAF76
4 bytes          : 0a 54 65 78
DWORD LE         : 0x7865540A
WORD bajo        : 0x540A
WORD alto        : 0x540A
Interpretaciones:
  hi:lo  segmento=21514 NO EXISTE
  lo:hi  segmento=21514 NO EXISTE

Nombre en archivo : 0x2FEE61
4 bytes          : 0a 54 65 78
DWORD LE         : 0x7865540A
WORD bajo        : 0x540A
WORD alto        : 0x540A
Interpretaciones:
  hi:lo  segmento=21514 NO EXISTE
  lo:hi  segmento=21514 NO EXISTE

------------------------------------------------------------
ZodiacoChange
------------------------------------------------------------

Nombre en archivo : 0x04B55A
4 bytes          : f8 3c c2 02
DWORD LE         : 0x02C23CF8
WORD bajo        : 0x3CF8
WORD alto        : 0x3CF8
Nombre está en   : segmento 3 +0x029A (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=15608 NO EXISTE
  lo:hi  segmento=15608 NO EXISTE

Nombre en archivo : 0x2F4A33
4 bytes          : 06 4f 6e 45
DWORD LE         : 0x456E4F06
WORD bajo        : 0x4F06
WORD alto        : 0x4F06
Interpretaciones:
  hi:lo  segmento=20230 NO EXISTE
  lo:hi  segmento=20230 NO EXISTE

Nombre en archivo : 0x2F4A49
4 bytes          : 00 00 09 54
DWORD LE         : 0x54090000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE

Nombre en archivo : 0x2F4B20
4 bytes          : 00 00 09 54
DWORD LE         : 0x54090000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE

------------------------------------------------------------
AhoraClick
------------------------------------------------------------

Nombre en archivo : 0x04B54B
4 bytes          : 6d 2f a9 02
DWORD LE         : 0x02A92F6D
WORD bajo        : 0x2F6D
WORD alto        : 0x2F6D
Nombre está en   : segmento 3 +0x028B (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=12141 NO EXISTE
  lo:hi  segmento=12141 NO EXISTE

Nombre en archivo : 0x2F4746
4 bytes          : 00 00 05 54
DWORD LE         : 0x54050000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE

------------------------------------------------------------
DHGoHarmClick
------------------------------------------------------------

Nombre en archivo : 0x04B668
4 bytes          : a9 41 c8 03
DWORD LE         : 0x03C841A9
WORD bajo        : 0x41A9
WORD alto        : 0x41A9
Nombre está en   : segmento 3 +0x03A8 (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=16809 NO EXISTE
  lo:hi  segmento=16809 NO EXISTE

Nombre en archivo : 0x2EEBD9
4 bytes          : 00 00 0b 54
DWORD LE         : 0x540B0000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE

------------------------------------------------------------
DHMSaveClick
------------------------------------------------------------

Nombre en archivo : 0x04B67A
4 bytes          : 1b 43 d9 03
DWORD LE         : 0x03D9431B
WORD bajo        : 0x431B
WORD alto        : 0x431B
Nombre está en   : segmento 3 +0x03BA (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=17179 NO EXISTE
  lo:hi  segmento=17179 NO EXISTE

Nombre en archivo : 0x2EEFA7
4 bytes          : 00 00 05 54
DWORD LE         : 0x54050000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE

------------------------------------------------------------
DHMOtroClick
------------------------------------------------------------

Nombre en archivo : 0x04B68B
4 bytes          : 6f 44 e9 03
DWORD LE         : 0x03E9446F
WORD bajo        : 0x446F
WORD alto        : 0x446F
Nombre está en   : segmento 3 +0x03CB (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=17519 NO EXISTE
  lo:hi  segmento=17519 NO EXISTE

Nombre en archivo : 0x2EEF08
4 bytes          : 00 00 07 54
DWORD LE         : 0x54070000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE

------------------------------------------------------------
ArmonicoChange
------------------------------------------------------------

Nombre en archivo : 0x04B720
4 bytes          : 80 47 87 04
DWORD LE         : 0x04874780
WORD bajo        : 0x4780
WORD alto        : 0x4780
Nombre está en   : segmento 3 +0x0460 (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=18304 NO EXISTE
  lo:hi  segmento=18304 NO EXISTE

Nombre en archivo : 0x2F4C19
4 bytes          : 00 00 0b 54
DWORD LE         : 0x540B0000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE

------------------------------------------------------------
UseARClick
------------------------------------------------------------

Nombre en archivo : 0x04B711
4 bytes          : 23 47 70 04
DWORD LE         : 0x04704723
WORD bajo        : 0x4723
WORD alto        : 0x4723
Nombre está en   : segmento 3 +0x0451 (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=18211 NO EXISTE
  lo:hi  segmento=18211 NO EXISTE

Nombre en archivo : 0x2F4B90
4 bytes          : 00 00 05 54
DWORD LE         : 0x54050000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE

------------------------------------------------------------
AjustarArmonicas1Click
------------------------------------------------------------

Nombre en archivo : 0x04B97E
4 bytes          : 96 5f f0 06
DWORD LE         : 0x06F05F96
WORD bajo        : 0x5F96
WORD alto        : 0x5F96
Nombre está en   : segmento 3 +0x06BE (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=24470 NO EXISTE
  lo:hi  segmento=24470 NO EXISTE

Nombre en archivo : 0x2F7E22
4 bytes          : 00 00 09 54
DWORD LE         : 0x54090000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE

------------------------------------------------------------
ArmnicosActivos1Click
------------------------------------------------------------

Nombre en archivo : 0x04B999
4 bytes          : e9 58 05 07
DWORD LE         : 0x070558E9
WORD bajo        : 0x58E9
WORD alto        : 0x58E9
Nombre está en   : segmento 3 +0x06D9 (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=22761 NO EXISTE
  lo:hi  segmento=22761 NO EXISTE

Nombre en archivo : 0x2F7DBF
4 bytes          : 00 00 09 54
DWORD LE         : 0x54090000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE

------------------------------------------------------------
Sinastra1Click
------------------------------------------------------------

Nombre en archivo : 0x04BA2B
4 bytes          : 93 63 8d 07
DWORD LE         : 0x078D6393
WORD bajo        : 0x6393
WORD alto        : 0x6393
Nombre está en   : segmento 3 +0x076B (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=25491 NO EXISTE
  lo:hi  segmento=25491 NO EXISTE

Nombre en archivo : 0x2F6DC1
4 bytes          : 00 00 07 54
DWORD LE         : 0x54070000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE

Nombre en archivo : 0x2F82FD
4 bytes          : 00 00 09 54
DWORD LE         : 0x54090000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE

------------------------------------------------------------
LugarDblClick
------------------------------------------------------------

Nombre en archivo : 0x04BA3E
4 bytes          : e3 63 b0 07
DWORD LE         : 0x07B063E3
WORD bajo        : 0x63E3
WORD alto        : 0x63E3
Nombre está en   : segmento 3 +0x077E (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=25571 NO EXISTE
  lo:hi  segmento=25571 NO EXISTE

Nombre en archivo : 0x2F4894
4 bytes          : 07 4f 6e 4b
DWORD LE         : 0x4B6E4F07
WORD bajo        : 0x4F07
WORD alto        : 0x4F07
Interpretaciones:
  hi:lo  segmento=20231 NO EXISTE
  lo:hi  segmento=20231 NO EXISTE

------------------------------------------------------------
OpcionesparaFlorArmonica1Click
------------------------------------------------------------

Nombre en archivo : 0x04BA50
4 bytes          : 9a 64 c3 07
DWORD LE         : 0x07C3649A
WORD bajo        : 0x649A
WORD alto        : 0x649A
Nombre está en   : segmento 3 +0x0790 (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=25754 NO EXISTE
  lo:hi  segmento=25754 NO EXISTE

Nombre en archivo : 0x2F7E97
4 bytes          : 00 00 09 54
DWORD LE         : 0x54090000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE

------------------------------------------------------------
CalcularSiempre1Click
------------------------------------------------------------

Nombre en archivo : 0x04B964
4 bytes          : 22 5f d6 06
DWORD LE         : 0x06D65F22
WORD bajo        : 0x5F22
WORD alto        : 0x5F22
Nombre está en   : segmento 3 +0x06A4 (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=24354 NO EXISTE
  lo:hi  segmento=24354 NO EXISTE

Nombre en archivo : 0x2F7A8F
4 bytes          : 00 00 09 54
DWORD LE         : 0x54090000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE

------------------------------------------------------------
NodosPlanetarios1Click
------------------------------------------------------------

Nombre en archivo : 0x04B9E0
4 bytes          : e0 62 4a 07
DWORD LE         : 0x074A62E0
WORD bajo        : 0x62E0
WORD alto        : 0x62E0
Nombre está en   : segmento 3 +0x0720 (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=25312 NO EXISTE
  lo:hi  segmento=25312 NO EXISTE

Nombre en archivo : 0x2F7B02
4 bytes          : 00 00 09 54
DWORD LE         : 0x54090000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE

------------------------------------------------------------
PlanetasqueseDibujan1Click
------------------------------------------------------------

Nombre en archivo : 0x04B585
4 bytes          : 9e 3e 01 03
DWORD LE         : 0x03013E9E
WORD bajo        : 0x3E9E
WORD alto        : 0x3E9E
Nombre está en   : segmento 3 +0x02C5 (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=16030 NO EXISTE
  lo:hi  segmento=16030 NO EXISTE

Nombre en archivo : 0x2F7B66
4 bytes          : 00 00 09 54
DWORD LE         : 0x54090000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE

------------------------------------------------------------
PlanetasqueseNumeran1Click
------------------------------------------------------------

Nombre en archivo : 0x04B7B8
4 bytes          : d6 6c 2b 05
DWORD LE         : 0x052B6CD6
WORD bajo        : 0x6CD6
WORD alto        : 0x6CD6
Nombre está en   : segmento 3 +0x04F8 (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=27862 NO EXISTE
  lo:hi  segmento=27862 NO EXISTE

Nombre en archivo : 0x2F7BCE
4 bytes          : 00 00 09 54
DWORD LE         : 0x54090000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE

------------------------------------------------------------
PlanetasqueSeAspectan1Click
------------------------------------------------------------
NO encontrado

------------------------------------------------------------
PlanetasqueEmitenAspectos1Click
------------------------------------------------------------

Nombre en archivo : 0x04B77B
4 bytes          : 10 4a f5 04
DWORD LE         : 0x04F54A10
WORD bajo        : 0x4A10
WORD alto        : 0x4A10
Nombre está en   : segmento 3 +0x04BB (flags 0x1D10)
Interpretaciones:
  hi:lo  segmento=18960 NO EXISTE
  lo:hi  segmento=18960 NO EXISTE

Nombre en archivo : 0x2F7CB0
4 bytes          : 00 00 09 54
DWORD LE         : 0x54090000
WORD bajo        : 0x0000
WORD alto        : 0x0000
Interpretaciones:
  hi:lo  segmento=0 NO EXISTE
  lo:hi  segmento=0 NO EXISTE
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

La corrección cambia bastante la interpretación.


En realidad, para:


```
CalcularClick → C3 1D CA 00
FormResize    → FA 1D DC 00
UseHelioClick → 1A 1E EE 00
```


los valores son:


```
WORD 1 = 0x1DC3   WORD 2 = 0x00CA
WORD 1 = 0x1DFA   WORD 2 = 0x00DC
WORD 1 = 0x1E1A   WORD 2 = 0x00EE
```


Y **0x1DC3, 0x1DFA, 0x1E1A... parecen offsets internos muy plausibles**, porque están dentro del segmento 3 de 0x6ED0 bytes.


Así que ahora debemos comprobar directamente qué hay en:


```
segmento 3 + 0x1DC3
segmento 3 + 0x1DFA
segmento 3 + 0x1E1A
...
```


Si allí encontramos instrucciones 8086 coherentes, habremos dado un salto enorme: **nombre del evento → posición real de su código**.


Ejecuta este bloque único; además corrige definitivamente la lectura de los dos `WORD`:


```
Bashcd /storage/emulated/0/Download/armon

python3 - <<'PY'
import struct

EXE = "ARMON.EXE"

with open(EXE, "rb") as f:
    data = f.read()

SEG_OFF = 0x4B2C0
SEG_SIZE = 0x6ED0

seg = data[SEG_OFF:SEG_OFF + SEG_SIZE]

objetivos = [
    "CalcularClick",
    "FormResize",
    "UseHelioClick",
    "UseSiderClick",
    "FormCreate",
    "ZodiacoChange",
    "AhoraClick",
    "DHGoHarmClick",
    "DHMSaveClick",
    "DHMOtroClick",
    "ArmonicoChange",
    "UseARClick",
    "AjustarArmonicas1Click",
    "ArmnicosActivos1Click",
    "Sinastra1Click",
    "LugarDblClick",
    "OpcionesparaFlorArmonica1Click",
    "CalcularSiempre1Click",
    "NodosPlanetarios1Click",
    "PlanetasqueseDibujan1Click",
    "PlanetasqueseNumeran1Click",
    "PlanetasqueEmitenAspectos1Click",
]

print("============================================================")
print(" POSIBLES DIRECCIONES DE LOS METODOS DELPHI")
print("============================================================")
print()
print(f"Segmento 3: archivo 0x{SEG_OFF:X}")
print(f"Tamaño    : 0x{SEG_SIZE:X}")
print()

for nombre in objetivos:

    b = nombre.encode("cp1252")
    pos = seg.find(b)

    if pos < 0:
        print(f"{nombre:<45} NO ENCONTRADO")
        continue

    a = pos + len(b)

    if a + 4 > len(seg):
        continue

    # IMPORTANTE:
    # ahora sí se leen correctamente los DOS WORD
    offset = struct.unpack_from("<H", seg, a)[0]
    segundo = struct.unpack_from("<H", seg, a + 2)[0]

    archivo = SEG_OFF + offset

    print()
    print(f"{nombre}")
    print(f"  nombre en segmento : 0x{pos:04X}")
    print(f"  archivo nombre     : 0x{SEG_OFF + pos:06X}")
    print(f"  WORD 1              : 0x{offset:04X}")
    print(f"  WORD 2              : 0x{segundo:04X}")

    if offset >= SEG_SIZE:
        print("  -> WORD 1 está fuera del segmento 3")
        continue

    print(f"  posible código      : segmento 3 + 0x{offset:04X}")
    print(f"  offset archivo      : 0x{archivo:06X}")

    code = seg[offset:offset + 32]

    print("  bytes:")

    for i in range(0, len(code), 16):

        parte = code[i:i+16]

        print(
            f"    {offset+i:04X}: "
            + " ".join(f"{x:02X}" for x in parte)
        )

print()
print("============================================================")
print(" FIN")
print("============================================================")
PY
```


### Qué vamos a comprobar


Por ejemplo, `CalcularClick` tiene:


```
C3 1D CA 00
```


y por tanto:


```
offset = 0x1DC3
segundo = 0x00CA
```


No debemos interpretar todavía `0x00CA`. Lo importante primero es comprobar **qué contiene `segmento 3 + 0x1DC3`**.


Si aparecen secuencias del tipo:


```
55 8B EC
...
8B ...
FF ...
E8 ...
```


o cualquier secuencia de instrucciones 8086 coherente, podremos empezar a desensamblar **el procedimiento real de `CalcularClick`**.


Y hay otra observación importante: los offsets están aumentando de forma muy lógica:


```
CalcularClick       1DC3
FormResize          1DFA
UseHelioClick       1E1A
UseSiderClick       1E55
...
```


Eso sugiere que estamos probablemente ante **una tabla de métodos consecutivos o referencias a métodos**, no ante valores aleatorios.


Este resultado es mucho más prometedor que la interpretación anterior.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> import struct
>
> EXE = "ARMON.EXE"
>
> with open(EXE, "rb") as f:
>     data = f.read()
>
> SEG_OFF = 0x4B2C0
> SEG_SIZE = 0x6ED0
>
> seg = data[SEG_OFF:SEG_OFF + SEG_SIZE]
>
> objetivos = [
>     "CalcularClick",
>     "FormResize",
>     "UseHelioClick",
>     "UseSiderClick",
>     "FormCreate",
>     "ZodiacoChange",
>     "AhoraClick",
>     "DHGoHarmClick",
>     "DHMSaveClick",
>     "DHMOtroClick",
>     "ArmonicoChange",
>     "UseARClick",
>     "AjustarArmonicas1Click",
>     "ArmnicosActivos1Click",
>     "Sinastra1Click",
>     "LugarDblClick",
>     "OpcionesparaFlorArmonica1Click",
>     "CalcularSiempre1Click",
>     "NodosPlanetarios1Click",
>     "PlanetasqueseDibujan1Click",
>     "PlanetasqueseNumeran1Click",
>     "PlanetasqueEmitenAspectos1Click",
> ]
>
> print("============================================================")
> print(" POSIBLES DIRECCIONES DE LOS METODOS DELPHI")
> print("============================================================")
> print()
> print(f"Segmento 3: archivo 0x{SEG_OFF:X}")
> print(f"Tamaño    : 0x{SEG_SIZE:X}")
> print()
>
> for nombre in objetivos:
>
>     b = nombre.encode("cp1252")
>     pos = seg.find(b)
>
>     if pos < 0:
>         print(f"{nombre:<45} NO ENCONTRADO")
>         continue
>
>     a = pos + len(b)
>
>     if a + 4 > len(seg):
>         continue
>
>     # IMPORTANTE:
>     # ahora sí se leen correctamente los DOS WORD
>     offset = struct.unpack_from("<H", seg, a)[0]
>     segundo = struct.unpack_from("<H", seg, a + 2)[0]
>
>     archivo = SEG_OFF + offset
>
>     print()
>     print(f"{nombre}")
>     print(f"  nombre en segmento : 0x{pos:04X}")
>     print(f"  archivo nombre     : 0x{SEG_OFF + pos:06X}")
>     print(f"  WORD 1              : 0x{offset:04X}")
>     print(f"  WORD 2              : 0x{segundo:04X}")
>
>     if offset >= SEG_SIZE:
>         print("  -> WORD 1 está fuera del segmento 3")
>         continue
>
>     print(f"  posible código      : segmento 3 + 0x{of
fset:04X}")
>     print(f"  offset archivo      : 0x{archivo:06X}")
>
>     code = seg[offset:offset + 32]
>
>     print("  bytes:")
>
>     for i in range(0, len(code), 16):
>
>         parte = code[i:i+16]
>
>         print(
>             f"    {offset+i:04X}: "
>             + " ".join(f"{x:02X}" for x in parte)
>         )
>
> print()
> print("============================================================")
> print(" FIN")
> print("============================================================")
> PY
============================================================
 POSIBLES DIRECCIONES DE LOS METODOS DELPHI
============================================================

Segmento 3: archivo 0x4B2C0
Tamaño    : 0x6ED0


CalcularClick
  nombre en segmento : 0x00AC
  archivo nombre     : 0x04B36C
  WORD 1              : 0x1DC3
  WORD 2              : 0x00CA
  posible código      : segmento 3 + 0x1DC3
  offset archivo      : 0x04D083
  bytes:
    1DC3: 55 89 E5 B8 08 00 9A 44 04 02 1E 83 EC 08 C4 7E
    1DD3: 06 06 57 9A F4 18 EB 1D 99 A3 5C 51 89 16 5E 51

FormResize
  nombre en segmento : 0x00BE
  archivo nombre     : 0x04B37E
  WORD 1              : 0x1DFA
  WORD 2              : 0x00DC
  posible código      : segmento 3 + 0x1DFA
  offset archivo      : 0x04D0BA
  bytes:
    1DFA: 55 89 E5 31 C0 9A 44 04 22 1E BF F9 1D 0E 57 BF
    1E0A: 9A 5A 1E 57 9A 8A 20 DC 24 E8 AD FB C9 CA 08 00

UseHelioClick
  nombre en segmento : 0x00CD
  archivo nombre     : 0x04B38D
  WORD 1              : 0x1E1A
  WORD 2              : 0x00EE
  posible código      : segmento 3 + 0x1E1A
  offset archivo      : 0x04D0DA
  bytes:
    1E1A: 55 89 E5 31 C0 9A 44 04 CB 1E 80 3E DE 25 00 74
    1E2A: 03 E9 8C 00 80 3E 9C 5A 00 B0 00 75 01 40 A2 9C

UseSiderClick
  nombre en segmento : 0x00DF
  archivo nombre     : 0x04B39F
  WORD 1              : 0x2455
  WORD 2              : 0x00FD
  posible código      : segmento 3 + 0x2455
  offset archivo      : 0x04D715
  bytes:
    2455: 55 89 E5 B8 46 02 9A 44 04 8B 24 81 EC 46 02 9A
    2465: 18 21 D9 31 C4 7E 06 26 8B 85 E8 01 26 8B 95 EA

FormCreate
  nombre en segmento : 0x00F1
  archivo nombre     : 0x04B3B1
  WORD 1              : 0x2853
  WORD 2              : 0x010B
  posible código      : segmento 3 + 0x2853
  offset archivo      : 0x04DB13
  bytes:
    2853: 55 89 E5 B8 06 01 9A 44 04 95 28 81 EC 06 01 8D
    2863: BE FA FE 16 57 C4 7E 06 26 C4 BD F8 01 06 57 9A

ZodiacoChange
  nombre en segmento : 0x029A
  archivo nombre     : 0x04B55A
  WORD 1              : 0x3CF8
  WORD 2              : 0x02C2
  posible código      : segmento 3 + 0x3CF8
  offset archivo      : 0x04EFB8
  bytes:
    3CF8: 55 89 E5 B8 0C 00 9A 44 04 32 3D 83 EC 0C BF B8
    3D08: 3C 0E 57 C4 7E 06 26 C4 BD A4 01 06 57 9A 67 25

AhoraClick
  nombre en segmento : 0x028B
  archivo nombre     : 0x04B54B
  WORD 1              : 0x2F6D
  WORD 2              : 0x02A9
  posible código      : segmento 3 + 0x2F6D
  offset archivo      : 0x04E22D
  bytes:
    2F6D: 55 89 E5 B8 52 02 9A 44 04 A3 2F 81 EC 52 02 8B
    2F7D: 46 0A 8B 56 0C C4 7E 06 26 3B 95 4E 02 75 1B 26

DHGoHarmClick
  nombre en segmento : 0x03A8
  archivo nombre     : 0x04B668
  WORD 1              : 0x41A9
  WORD 2              : 0x03C8
  posible código      : segmento 3 + 0x41A9
  offset archivo      : 0x04F469
  bytes:
    41A9: 55 89 E5 B8 00 01 9A 44 04 E4 41 81 EC 00 01 BF
    41B9: 83 41 0E 57 C4 7E 06 26 C4 BD A4 01 06 57 9A 67

DHMSaveClick
  nombre en segmento : 0x03BA
  archivo nombre     : 0x04B67A
  WORD 1              : 0x431B
  WORD 2              : 0x03D9
  posible código      : segmento 3 + 0x431B
  offset archivo      : 0x04F5DB
  bytes:
    431B: 55 89 E5 B8 00 01 9A 44 04 56 43 81 EC 00 01 BF
    432B: F5 42 0E 57 C4 7E 06 26 C4 BD E8 01 06 57 9A 67

DHMOtroClick
  nombre en segmento : 0x03CB
  archivo nombre     : 0x04B68B
  WORD 1              : 0x446F
  WORD 2              : 0x03E9
  posible código      : segmento 3 + 0x446F
  offset archivo      : 0x04F72F
  bytes:
    446F: 55 89 E5 31 C0 9A 44 04 9B 44 C4 7E 06 26 FF B5
    447F: D6 01 26 FF B5 D4 01 26 C4 BD D4 01 26 8B 85 F2

ArmonicoChange
  nombre en segmento : 0x0460
  archivo nombre     : 0x04B720
  WORD 1              : 0x4780
  WORD 2              : 0x0487
  posible código      : segmento 3 + 0x4780
  offset archivo      : 0x04FA40
  bytes:
    4780: 55 89 E5 B8 00 01 9A 44 04 E6 47 81 EC 00 01 8D
    4790: BE 00 FF 16 57 BF 75 47 0E 57 9B DD 06 96 54 9B

UseARClick
  nombre en segmento : 0x0451
  archivo nombre     : 0x04B711
  WORD 1              : 0x4723
  WORD 2              : 0x0470
  posible código      : segmento 3 + 0x4723
  offset archivo      : 0x04F9E3
  bytes:
    4723: 55 89 E5 B8 0A 01 9A 44 04 89 47 81 EC 0A 01 8D
    4733: BE F6 FE 16 57 C4 7E 06 26 C4 BD 5C 02 06 57 9A

AjustarArmonicas1Click
  nombre en segmento : 0x06BE
  archivo nombre     : 0x04B97E
  WORD 1              : 0x5F96
  WORD 2              : 0x06F0
  posible código      : segmento 3 + 0x5F96
  offset archivo      : 0x051256
  bytes:
    5F96: 55 89 E5 B8 6C 01 9A 44 04 C9 5F 81 EC 6C 01 8D
    5FA6: BE 94 FE 16 57 BF 5E 5F 0E 57 BF 7C 5F 0E 57 BF

ArmnicosActivos1Click
  nombre en segmento : 0x06D9
  archivo nombre     : 0x04B999
  WORD 1              : 0x58E9
  WORD 2              : 0x0705
  posible código      : segmento 3 + 0x58E9
  offset archivo      : 0x050BA9
  bytes:
    58E9: 55 89 E5 B8 02 02 9A 44 04 24 59 81 EC 02 02 BF
    58F9: 5C 47 1E 57 C4 7E 06 26 C4 BD E8 01 06 57 9A F9

Sinastra1Click
  nombre en segmento : 0x076B
  archivo nombre     : 0x04BA2B
  WORD 1              : 0x6393
  WORD 2              : 0x078D
  posible código      : segmento 3 + 0x6393
  offset archivo      : 0x051653
  bytes:
    6393: 55 89 E5 31 C0 9A 44 04 EC 63 C4 3E 64 5B 06 57
    63A3: 9A 6B 27 FF FF C9 CA 08 00 1E 4F 70 63 69 6F 6E

LugarDblClick
  nombre en segmento : 0x077E
  archivo nombre     : 0x04BA3E
  WORD 1              : 0x63E3
  WORD 2              : 0x07B0
  posible código      : segmento 3 + 0x63E3
  offset archivo      : 0x0516A3
  bytes:
    63E3: 55 89 E5 B8 00 01 9A 44 04 19 64 81 EC 00 01 8D
    63F3: BE 00 FF 16 57 BF AC 63 0E 57 BF CB 63 0E 57 C4

OpcionesparaFlorArmonica1Click
  nombre en segmento : 0x0790
  archivo nombre     : 0x04BA50
  WORD 1              : 0x649A
  WORD 2              : 0x07C3
  posible código      : segmento 3 + 0x649A
  offset archivo      : 0x05175A
  bytes:
    649A: 55 89 E5 31 C0 9A 44 04 1D 65 80 3E DE 25 00 74
    64AA: 02 EB 64 C6 06 DE 25 01 C4 7E 06 26 C4 BD E8 02

CalcularSiempre1Click
  nombre en segmento : 0x06A4
  archivo nombre     : 0x04B964
  WORD 1              : 0x5F22
  WORD 2              : 0x06D6
  posible código      : segmento 3 + 0x5F22
  offset archivo      : 0x0511E2
  bytes:
    5F22: 55 89 E5 B8 00 01 9A 44 04 55 5F 81 EC 00 01 8D
    5F32: BE 00 FF 16 57 BF F9 5E 0E 57 BF 15 5F 0E 57 BF

NodosPlanetarios1Click
  nombre en segmento : 0x0720
  archivo nombre     : 0x04B9E0
  WORD 1              : 0x62E0
  WORD 2              : 0x074A
  posible código      : segmento 3 + 0x62E0
  offset archivo      : 0x0515A0
  bytes:
    62E0: 55 89 E5 B8 00 01 9A 44 04 13 63 81 EC 00 01 8D
    62F0: BE 00 FF 16 57 BF B1 62 0E 57 BF DA 62 0E 57 BF

PlanetasqueseDibujan1Click
  nombre en segmento : 0x02C5
  archivo nombre     : 0x04B585
  WORD 1              : 0x3E9E
  WORD 2              : 0x0301
  posible código      : segmento 3 + 0x3E9E
  offset archivo      : 0x04F15E
  bytes:
    3E9E: 55 89 E5 B8 00 01 9A 44 04 D1 3E 81 EC 00 01 8D
    3EAE: BE 00 FF 16 57 BF 6C 3E 0E 57 BF 8A 3E 0E 57 BF

PlanetasqueseNumeran1Click
  nombre en segmento : 0x04F8
  archivo nombre     : 0x04B7B8
  WORD 1              : 0x6CD6
  WORD 2              : 0x052B
  posible código      : segmento 3 + 0x6CD6
  offset archivo      : 0x051F96
  bytes:
    6CD6: 55 89 E5 B8 32 01 9A 44 04 FF 6C 81 EC 32 01 8D
    6CE6: BE CE FE 16 57 BF D6 54 1E 57 9A 95 34 FF FF 8D

PlanetasqueEmitenAspectos1Click
  nombre en segmento : 0x04BB
  archivo nombre     : 0x04B77B
  WORD 1              : 0x4A10
  WORD 2              : 0x04F5
  posible código      : segmento 3 + 0x4A10
  offset archivo      : 0x04FCD0
  bytes:
    4A10: 55 89 E5 B8 00 01 9A 44 04 35 4A 81 EC 00 01 BF
    4A20: EC 49 0E 57 BF FF 49 0E 57 8D BE 00 FF 16 57 68

============================================================
 FIN
============================================================
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Este resultado es **decisivo**. Ya podemos afirmar algo que antes solo era una hipótesis:


> **Los `WORD 1` son offsets reales de entrada de los procedimientos Delphi dentro del segmento 3.**


La prueba es extremadamente fuerte: **todos los offsets encontrados comienzan con el mismo prólogo de procedimiento 16-bit**:


```
55 89 E5
```


que corresponde a:


```
asmpush bp
mov  bp,sp
```


Por ejemplo:


```
CalcularClick       → 0x1DC3 → 55 89 E5
FormResize          → 0x1DFA → 55 89 E5
UseHelioClick       → 0x1E1A → 55 89 E5
UseSiderClick       → 0x2455 → 55 89 E5
FormCreate          → 0x2853 → 55 89 E5
ZodiacoChange       → 0x3CF8 → 55 89 E5
DHGoHarmClick       → 0x41A9 → 55 89 E5
ArmonicoChange      → 0x4780 → 55 89 E5
AjustarArmonicas... → 0x5F96 → 55 89 E5
Sinastra1Click      → 0x6393 → 55 89 E5
```


Esto ya no parece una coincidencia.


### Lo que acabamos de recuperar


Tenemos una correspondencia directa:


```
Nombre Delphi
     ↓
registro interno
     ↓
WORD 1
     ↓
offset dentro del segmento 3
     ↓
código máquina del procedimiento
```


Por ejemplo:


```
CalcularClick
      ↓
0x1DC3
      ↓
55 89 E5 B8 08 00 ...
      ↓
procedimiento Delphi
```


Y además los procedimientos están **perfectamente separados**. Por ejemplo:


```
CalcularClick       0x1DC3
FormResize          0x1DFA
UseHelioClick       0x1E1A
```


Por tanto podemos empezar a reconstruir el código original.


### Hay otro detalle muy interesante


El segundo `WORD` probablemente **no es parte de la dirección del procedimiento**.


Por ejemplo:


```
CalcularClick → 0x1DC3 / 0x00CA
FormResize    → 0x1DFA / 0x00DC
UseHelioClick → 0x1E1A / 0x00EE
```


El primer WORD apunta claramente al código; el segundo pertenece a otra estructura asociada. **No debemos interpretarlo todavía.**


---


## Ahora podemos hacer algo mucho más útil


En lugar de seguir analizando las cadenas, podemos extraer automáticamente cada procedimiento y desensamblarlo como **8086/80186 de 16 bits**, utilizando como límites las direcciones de los siguientes procedimientos.


Eso nos permitirá obtener algo conceptualmente así:


```
pascalprocedure TForm1.CalcularClick(...);
begin
    ...
end;
```


reconstruido desde:


```
asmpush bp
mov bp,sp
...
call ...
...
leave
ret
```


No recuperaremos mágicamente los nombres originales de las variables, pero sí podremos reconstruir la lógica.


**El siguiente paso debe ser desensamblar los procedimientos reales.**


Ejecuta este único bloque:


```
Bashcd /storage/emulated/0/Download/armon

python3 - <<'PY'
import struct
import subprocess
from pathlib import Path

EXE = "ARMON.EXE"
SEG_OFF = 0x4B2C0
SEG_SIZE = 0x6ED0

with open(EXE, "rb") as f:
    data = f.read()

seg = data[SEG_OFF:SEG_OFF + SEG_SIZE]

# Nombre -> offset real del procedimiento
metodos = [
    ("CalcularClick",                 0x1DC3),
    ("FormResize",                    0x1DFA),
    ("UseHelioClick",                 0x1E1A),
    ("UseSiderClick",                 0x2455),
    ("FormCreate",                    0x2853),
    ("ZodiacoChange",                 0x3CF8),
    ("AhoraClick",                    0x2F6D),
    ("DHGoHarmClick",                 0x41A9),
    ("DHMSaveClick",                  0x431B),
    ("DHMOtroClick",                  0x446F),
    ("UseARClick",                    0x4723),
    ("ArmonicoChange",                0x4780),
    ("PlanetasqueseDibujan1Click",    0x3E9E),
    ("PlanetasqueEmitenAspectos1Click",0x4A10),
    ("ArmnicosActivos1Click",         0x58E9),
    ("CalcularSiempre1Click",         0x5F22),
    ("AjustarArmonicas1Click",        0x5F96),
    ("NodosPlanetarios1Click",        0x62E0),
    ("Sinastra1Click",                0x6393),
    ("LugarDblClick",                 0x63E3),
    ("OpcionesparaFlorArmonica1Click",0x649A),
    ("PlanetasqueseNumeran1Click",    0x6CD6),
]

OUT = Path.home() / "armon_code" / "procedimientos"
OUT.mkdir(parents=True, exist_ok=True)

# Ordenar por dirección
metodos = sorted(metodos, key=lambda x: x[1])

# Crear archivos individuales
for i, (nombre, inicio) in enumerate(metodos):

    if i + 1 < len(metodos):
        fin = metodos[i + 1][1]
    else:
        fin = SEG_SIZE

    if fin <= inicio:
        continue

    bloque = seg[inicio:fin]

    archivo_bin = OUT / f"{inicio:04X}_{nombre}.bin"

    with open(archivo_bin, "wb") as f:
        f.write(bloque)

    print(
        f"{nombre:<38} "
        f"0x{inicio:04X} - 0x{fin:04X} "
        f"({len(bloque)} bytes)"
    )

print()
print("Archivos creados en:")
print(OUT)

# ------------------------------------------------------------
# Intentar desensamblar con GNU objdump
# ------------------------------------------------------------

objdump = "/data/data/com.termux/files/usr/bin/objdump"

if not Path(objdump).exists():
    objdump = "objdump"

print()
print("============================================================")
print(" DESENSAMBLADO 8086")
print("============================================================")

for i, (nombre, inicio) in enumerate(metodos):

    if i + 1 < len(metodos):
        fin = metodos[i + 1][1]
    else:
        fin = SEG_SIZE

    bloque = seg[inicio:fin]

    if not bloque:
        continue

    binfile = OUT / f"{inicio:04X}_{nombre}.bin"
    asmfile = OUT / f"{inicio:04X}_{nombre}.asm"

    cmd = [
        objdump,
        "-D",
        "-b", "binary",
        "-m", "i8086",
        "--adjust-vma", f"0x{inicio:X}",
        str(binfile)
    ]

    try:

        result = subprocess.run(
            cmd,
            capture_output=True,
            text=True
        )

        with open(asmfile, "w") as f:
            f.write(result.stdout)
            if result.stderr:
                f.write("\n\nSTDERR:\n")
                f.write(result.stderr)

        print(f"{nombre:<38} -> {asmfile}")

    except Exception as e:
        print(f"{nombre:<38} ERROR: {e}")

print()
print("============================================================")
print(" PRIMERAS LINEAS DE CALCULARCLICK")
print("============================================================")

archivo = OUT / "1DC3_CalcularClick.asm"

if archivo.exists():

    with open(archivo, "r", errors="replace") as f:
        for i, linea in enumerate(f):

            print(linea.rstrip())

            if i >= 80:
                break

else:
    print("No se pudo generar el desensamblado.")
```


La salida de **`CalcularClick`** es ahora la pieza que más nos interesa.


A partir de ella podremos empezar a traducir instrucciones 8086 a una representación comprensible y, progresivamente, reconstruir algo cercano al **código Pascal/Delphi original**.

## Usuario

Esta correcto el codigo me dejo barado en el prompt >

## Asistente

No. **Quedó incompleto al pegarse en Termux**, por eso apareció el prompt `>`: Bash estaba esperando que terminara una construcción, normalmente una comilla, paréntesis o heredoc.


Además, en mi bloque anterior hubo un problema de formato que pudo provocar eso. **No ejecutes nada mientras estés en `>`**.


Primero cancela el comando pendiente con `Ctrl+C`.


Luego usa este bloque corregido y más sencillo. Mantiene exactamente el objetivo: extraer los procedimientos y generar sus desensamblados.


```
BashCtrl+C
cd /storage/emulated/0/Download/armon

python3 - <<'PY'
import struct
import subprocess
from pathlib import Path

EXE = "ARMON.EXE"
SEG_OFF = 0x4B2C0
SEG_SIZE = 0x6ED0

with open(EXE, "rb") as f:
    data = f.read()

seg = data[SEG_OFF:SEG_OFF + SEG_SIZE]

metodos = [
    ("CalcularClick", 0x1DC3),
    ("FormResize", 0x1DFA),
    ("UseHelioClick", 0x1E1A),
    ("UseSiderClick", 0x2455),
    ("FormCreate", 0x2853),
    ("ZodiacoChange", 0x3CF8),
    ("AhoraClick", 0x2F6D),
    ("DHGoHarmClick", 0x41A9),
    ("DHMSaveClick", 0x431B),
    ("DHMOtroClick", 0x446F),
    ("UseARClick", 0x4723),
    ("ArmonicoChange", 0x4780),
    ("PlanetasqueseDibujan1Click", 0x3E9E),
    ("PlanetasqueEmitenAspectos1Click", 0x4A10),
    ("ArmnicosActivos1Click", 0x58E9),
    ("CalcularSiempre1Click", 0x5F22),
    ("AjustarArmonicas1Click", 0x5F96),
    ("NodosPlanetarios1Click", 0x62E0),
    ("Sinastra1Click", 0x6393),
    ("LugarDblClick", 0x63E3),
    ("OpcionesparaFlorArmonica1Click", 0x649A),
    ("PlanetasqueseNumeran1Click", 0x6CD6),
]

metodos.sort(key=lambda x: x[1])

out = Path.home() / "armon_code" / "procedimientos"
out.mkdir(parents=True, exist_ok=True)

for i, (nombre, inicio) in enumerate(metodos):

    if i + 1 < len(metodos):
        fin = metodos[i + 1][1]
    else:
        fin = SEG_SIZE

    bloque = seg[inicio:fin]

    binfile = out / ("%04X_%s.bin" % (inicio, nombre))
    asmfile = out / ("%04X_%s.asm" % (inicio, nombre))

    with open(binfile, "wb") as f:
        f.write(bloque)

    cmd = [
        "objdump",
        "-D",
        "-b", "binary",
        "-m", "i8086",
        "--adjust-vma", "0x%X" % inicio,
        str(binfile)
    ]

    r = subprocess.run(
        cmd,
        stdout=subprocess.PIPE,
        stderr=subprocess.STDOUT,
        text=True
    )

    with open(asmfile, "w") as f:
        f.write(r.stdout)

    print("%-38s 0x%04X - 0x%04X  %d bytes" %
          (nombre, inicio, fin, len(bloque)))

print()
print("Archivos:")
print(out)

print()
print("============================================================")
print(" CALCULARCLICK")
print("============================================================")

asm = out / "1DC3_CalcularClick.asm"

if asm.exists():
    with open(asm, "r", errors="replace") as f:
        for i, linea in enumerate(f):
            print(linea.rstrip())
            if i >= 100:
                break
else:
    print("No se genero el archivo de desensamblado.")
PY
```


### Qué significa el `>`


Cuando Termux muestra:


```
>
```


**no es un error de Python**. Es Bash diciendo:


> «Creo que todavía no terminaste el comando anterior».


En este caso, `Ctrl+C` devuelve el prompt normal:


```
.../Download/armon $
```


y entonces puedes pegar el bloque corregido.


Y sí: **el descubrimiento anterior sigue siendo válido**. Los offsets `0x1DC3`, `0x1DFA`, `0x1E1A`, etc. producen el prólogo inequívoco:


```
55 89 E5
```


en todos esos casos. Esa es una evidencia muy fuerte de que estamos localizando correctamente los procedimientos compilados de Delphi.

## Usuario

.../Download/armon $ Ctrl+C
= "ARMON.EXE"
SEG_OFF = 0x4B2C0
SEG_SIZE = 0x6ED0

with open(EXE, "rb") as f:
    data = f.read()

seg = data[SEG_OFF:SEG_OFF + SEG_SIZE]

metodos = [
    ("CalcularClick", 0x1DC3),
    ("FormResize", 0x1DFA),
    ("UseHelioClick", 0x1E1A),
    ("UseSiderClick", 0x2455),
    ("FormCreate", 0x2853),
    ("ZodiacoChange", 0x3CF8),
    ("AhoraClick", 0x2F6D),
    ("DHGoHarmClick", 0x41A9),
    ("DHMSaveClick", 0x431B),
    ("DHMOtroClick", 0x446F),
    ("UseARClick", 0x4723),
    ("ArmonicoChange", 0x4780),
    ("PlanetasqueseDibujan1Click", 0x3E9E),
    ("PlanetasqueEmitenAspectos1Click", 0x4A10),
    ("ArmnicosActivos1Click", 0x58E9),
    ("CalcularSiempre1Click", 0x5F22),
    ("AjustarArmonicas1Click", 0x5F96),
    ("NodosPlanetarios1Click", 0x62E0),
    ("Sinastra1Click", 0x6393),
    ("LugarDblClick", 0x63E3),
    ("OpcionesparaFlorArmonica1Click", 0x649A),
    ("PlanetasqueseNumeran1Click", 0x6CD6),
]

metodos.sort(key=lambda x: x[1])

out = Path.home() / "armon_code" / "procedimientos"
out.mkdir(parents=True, exist_ok=True)

for i, (nombre, inicio) in enumerate(metodos):

    if i + 1 < len(metodos):
        fin = metodos[i + 1][1]
    else:
        fin = SEG_SIZE

    bloque = seg[inicio:fin]

    binfile = out / ("%04X_%s.bin" % (inicio, nombre))
    asmfile = out / ("%04X_%s.asm" % (inicio, nombre))

    with open(binfile, "wb") as f:
        f.write(bloque)

    cmd = [
        "objdump",
        "-D",
        "-b", "binary",
        "-m", "i8086",
        "--adjust-vma", "0x%X" % inicio,
        str(binfile)
    ]

    r = subprocess.run(
        cmd,
        stdout=subprocess.PIPE,
        stderr=subprocess.STDOUT,
        text=True
    )

    with open(asmfile, "w") as f:
        f.write(r.stdout)

    print("%-38s 0x%04X - 0x%04X  %d bytes" %
          (nombre, inicio, fin, len(bloque)))

print()
print("Archivos:")
print(out)

print()
print("============================================================")
print(" CALCULARCLICK")
print("============================================================")

asm = out / "1DC3_CalcularClick.asm"

if asm.exists():
    with open(asm, "r", errors="replace") as f:
        for i, linea in enumerate(f):
            print(linea.rstrip())
            if i >= 100:
                break
else:
    print("No se genero el archivo de desensamblado.")
PYCtrl+C: command not found
.../Download/armon $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> import struct
> import subprocess
> from pathlib import Path
>
> EXE = "ARMON.EXE"
> SEG_OFF = 0x4B2C0
> SEG_SIZE = 0x6ED0
>
> with open(EXE, "rb") as f:
>     data = f.read()
>
> seg = data[SEG_OFF:SEG_OFF + SEG_SIZE]
>
> metodos = [
>     ("CalcularClick", 0x1DC3),
>     ("FormResize", 0x1DFA),
>     ("UseHelioClick", 0x1E1A),
>     ("UseSiderClick", 0x2455),
>     ("FormCreate", 0x2853),
>     ("ZodiacoChange", 0x3CF8),
>     ("AhoraClick", 0x2F6D),
>     ("DHGoHarmClick", 0x41A9),
>     ("DHMSaveClick", 0x431B),
>     ("DHMOtroClick", 0x446F),
>     ("UseARClick", 0x4723),
>     ("ArmonicoChange", 0x4780),
>     ("PlanetasqueseDibujan1Click", 0x3E9E),
>     ("PlanetasqueEmitenAspectos1Click", 0x4A10),
>     ("ArmnicosActivos1Click", 0x58E9),
>     ("CalcularSiempre1Click", 0x5F22),
>     ("AjustarArmonicas1Click", 0x5F96),
>     ("NodosPlanetarios1Click", 0x62E0),
>     ("Sinastra1Click", 0x6393),
>     ("LugarDblClick", 0x63E3),
>     ("OpcionesparaFlorArmonica1Click", 0x649A),
>     ("PlanetasqueseNumeran1Click", 0x6CD6),
> ]
>
> metodos.sort(key=lambda x: x[1])
>
> out = Path.home() / "armon_code" / "procedimientos"
> out.mkdir(parents=True, exist_ok=True)
>
> for i, (nombre, inicio) in enumerate(metodos):
>
>     if i + 1 < len(metodos):
>         fin = metodos[i + 1][1]
>     else:
>         fin = SEG_SIZE
>
>     bloque = seg[inicio:fin]
>                                                      )
>     asmfile = out / ("%04X_%s.asm" % (inicio, nombre))
>     with open(binfile, "wb") as f:
>         f.write(bloque)
>
>     cmd = [
>         "objdump",
>         "-D",
>         "-b", "binary",
>         "-m", "i8086",
>         "--adjust-vma", "0x%X" % inicio,
>         str(binfile)
>     ]
>
>     r = subprocess.run(
>         cmd,
>         stdout=subprocess.PIPE,
>         stderr=subprocess.STDOUT,
>         text=True
>     )
>
>     with open(asmfile, "w") as f:
>         f.write(r.stdout)
>
>     print("%-38s 0x%04X - 0x%04X  %d bytes" %
>           (nombre, inicio, fin, len(bloque)))
>
> print()
> print("Archivos:")
> print(out)
>
> print()
> print("============================================================")
> print(" CALCULARCLICK")
> print("============================================================")
>
> asm = out / "1DC3_CalcularClick.asm"
>
> if asm.exists():
>     with open(asm, "r", errors="replace") as f:
>         for i, linea in enumerate(f):
>             print(linea.rstrip())
>             if i >= 100:
>                 break
> else:                                                )
> PY  print("No se genero el archivo de desensamblado.")
CalcularClick                          0x1DC3 - 0x1DFA  55 bytes
FormResize                             0x1DFA - 0x1E1A  32 bytes
UseHelioClick                          0x1E1A - 0x2455  1595 bytes
UseSiderClick                          0x2455 - 0x2853  1022 bytes
FormCreate                             0x2853 - 0x2F6D  1818 bytes
AhoraClick                             0x2F6D - 0x3CF8  3467 bytes
ZodiacoChange                          0x3CF8 - 0x3E9E  422 bytes
PlanetasqueseDibujan1Click             0x3E9E - 0x41A9  779 bytes
DHGoHarmClick                          0x41A9 - 0x431B  370 bytes
DHMSaveClick                           0x431B - 0x446F  340 bytes
DHMOtroClick                           0x446F - 0x4723  692 bytes
UseARClick                             0x4723 - 0x4780  93 bytes
ArmonicoChange                         0x4780 - 0x4A10  656 bytes
PlanetasqueEmitenAspectos1Click        0x4A10 - 0x58E9  3801 bytes
ArmnicosActivos1Click                  0x58E9 - 0x5F22  1593 bytes
CalcularSiempre1Click                  0x5F22 - 0x5F96  116 bytes
AjustarArmonicas1Click                 0x5F96 - 0x62E0  842 bytes
NodosPlanetarios1Click                 0x62E0 - 0x6393  179 bytes
Sinastra1Click                         0x6393 - 0x63E3  80 bytes
LugarDblClick                          0x63E3 - 0x649A  183 bytes
OpcionesparaFlorArmonica1Click         0x649A - 0x6CD6  2108 bytes
PlanetasqueseNumeran1Click             0x6CD6 - 0x6ED0  506 bytes

Archivos:
/data/data/com.termux/files/home/armon_code/procedimientos

============================================================
 CALCULARCLICK
============================================================
objdump: error: unknown argument '-b'
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Sí. **Ahora sí ejecutaste correctamente el script**. El problema restante no es tu código de Python: es que el `objdump` de Termux que tienes instalado **no acepta la opción `-b`**.


Lo importante es que ya conseguimos esto:


- Los 22 procedimientos fueron extraídos.
- Sus límites quedaron definidos.
- Los `.bin` fueron creados correctamente.
- Solo falló la etapa de desensamblado.


Y hay un detalle importante: **no debemos modificar ni volver a calcular esos offsets**. Ya están confirmados por el prólogo `55 89 E5`.


Vamos a comprobar qué sintaxis soporta exactamente tu `objdump`, sin tocar los archivos generados. Ejecuta solamente este bloque:


```
Bashcd /storage/emulated/0/Download/armon

echo "=== OBJdump ==="
objdump --version | head -3

echo
echo "=== AYUDA ==="
objdump --help | head -60

echo
echo "=== ARCHIVO EXTRAIDO ==="
ls -lh ~/armon_code/procedimientos/1DC3_CalcularClick.bin

echo
echo "=== PRIMEROS BYTES ==="
xxd -g 1 -l 64 ~/armon_code/procedimientos/1DC3_CalcularClick.bin
```


Con esa salida podremos adaptar **solo el comando de desensamblado** al `objdump` real de tu Termux.


Además, el primer procedimiento mide:


```
CalcularClick = 55 bytes
```


porque va exactamente de:


```
0x1DC3 → 0x1DFA
```


Así que ya tenemos el primer procedimiento Delphi aislado con precisión.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ echo "=== OBJdump ==="
=== OBJdump ===
bjdump --version | head -3

echo
echo "=== AYUDA ==="
objdump --help | head -60

echo
echo "=== ARCHIVO EXTRAIDO ==="
ls -lh ~/armon_code/procedimientos/1DC3_CalcularClick.bin

echo
echo "=== PRIMEROS BYTES ==="
xxd -g 1 -l 64 ~/armon_code/procedimientos/1DC3_CalcularClick.bin.../Download/armon $ objdump --version | head -3
LLVM (http://llvm.org/):
  LLVM version 21.1.8
  Optimized build.
.../Download/armon $
.../Download/armon $ echo

.../Download/armon $ echo "=== AYUDA ==="
=== AYUDA ===
.../Download/armon $ objdump --help | head -60
OVERVIEW: llvm object file dumper

USAGE: objdump [options] <input object files>

OPTIONS:
  --adjust-vma=offset     Increase the displayed address by the specified offset
  --all-headers           Display all available header information, relocation entries and the symbol table
  --arch-name=<value>     Target arch to disassemble for, see --version for available targets
  --archive-headers       Display archive header information
  -a                      Alias for --archive-headers
  --build-id=<hex>        Build ID to look up. Once found, added as an input file
  -C                      Alias for --demangle
  --debug-file-directory=<dir>
                          Path to directory where to look for debug files
  --debug-vars-indent=<value>
                          Distance to indent the source-level variable display, relative to the start of the disassembly
  --debug-vars=<value>    Print the locations (in registers or memory) of source-level variables alongside disassembly. Supported formats: ascii, unicode (default)
  --debuginfod            Use debuginfod to find debug files
  --demangle              Demangle symbol names
  --disassemble-all       Disassemble all sections found in the input files
  --disassemble-symbols=<value>
                          List of symbols to disassemble. Accept demangled names when --demangle is specified, otherwise accept mangled names
  --disassemble-zeroes    Do not skip blocks of zeroes when disassembling
  --disassembler-color=mode
                          Enable or disable disassembler color output. Valid options are "on", "off" and "terminal" (default)
  --disassembler-options=options
                          Pass target specific disassembler options
  --disassemble           Disassemble all executable sections found in the input files
  --dwarf=<value>         Dump the specified DWARF debug sections. The only supported value is 'frames'
  --dynamic-reloc         Display the dynamic relocation entries in the file
  --dynamic-syms          Display the contents of the dynamic symbol table
  -D                      Alias for --disassemble-all
  -d                      Alias for --disassemble
  --fault-map-section     Display the content of the fault map section
  --file-headers          Display the contents of the overall file header
  --full-contents         Display the content of each section
  -f                      Alias for --file-headers
  --headers               Alias for --section-headers
  --help                  Display available options (--help-hidden for more)
  -h                      Alias for --section-headers
  -j <value>              Alias for --section
  --line-numbers          When disassembling, display source line numbers. Implies --disassemble
  -l                      Alias for --line-numbers
  --macho                 Use MachO specific object file parser
  --mattr=a1,+a2,-a3,...  Target specific attributes (--mattr=help for details)
  --mcpu=cpu-name         Target a specific cpu type (--mcpu=help for details)
  -mllvm <arg>            Specify an argument to forward to LLVM's CommandLine library
  -M <value>              Alias for --disassembler-options=
  -m                      Alias for --macho
  --no-addresses          Alias for --no-leading-addr
  --no-debuginfod         Don't use debuginfod to find debug files
  --no-leading-addr       When disassembling, do not print leading addresses for instructions or inline relocations
  --no-print-imm-hex      Do not use hex format for immediate values
  --no-show-raw-insn      When disassembling instructions, do not print the instruction bytes.
  --offloading            Display the content of the offloading section
  --prefix-strip=prefix   Strip out initial directories from absolute paths. No effect without --prefix
  --prefix=prefix         Add prefix to absolute paths
  --pretty-pgo-analysis-map
                          Display PGO analysis values with formatting rather than raw numbers
  --print-imm-hex         Use hex format for immediate values (default)
.../Download/armon $
.../Download/armon $ echo

.../Download/armon $ echo "=== ARCHIVO EXTRAIDO ==="
=== ARCHIVO EXTRAIDO ===
.../Download/armon $ ls -lh ~/armon_code/procedimientos/1DC3_CalcularClick.bin
-rw-------. 1 u0_a197 u0_a197 55 Sep 26 22:36 /data/data/com.termux/files/home/armon_code/procedimientos/1DC3_CalcularClick.bin
.../Download/armon $
.../Download/armon $ echo

.../Download/armon $ echo "=== PRIMEROS BYTES ==="
=== PRIMEROS BYTES ===
.../Download/armon $ xxd -g 1 -l 64 ~/armon_code/procedimientos/1DC3_CalcularClick.bin
00000000: 55 89 e5 b8 08 00 9a 44 04 02 1e 83 ec 08 c4 7e  U......D.......~
00000010: 06 06 57 9a f4 18 eb 1d 99 a3 5c 51 89 16 5e 51  ..W.......\Q..^Q
00000020: c4 7e 06 06 57 9a a9 18 ad 1f 99 a3 58 51 89 16  .~..W.......XQ..
00000030: 5a 51 c9 ca 08 00 00                             ZQ.....
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Perfecto. Ahora tenemos la información necesaria y, además, **el `CalcularClick` de 55 bytes está completo**.


Hay una corrección importante respecto a lo que te dije antes: tu `objdump` es **LLVM 21.1.8**, no GNU `objdump`. Por eso `-b binary` no funciona.


Pero podemos evitar depender de `objdump` por completo. Como ya tenemos los bytes y sabemos que son código **8086**, podemos utilizar directamente el desensamblador de LLVM (`llvm-objdump`) y especificarle la arquitectura.


Primero probemos **un solo procedimiento**, para no generar más archivos hasta confirmar que el desensamblado es correcto.


Ejecuta este único bloque:


```
Bashcd /storage/emulated/0/Download/armon

echo "=== LLVM OBJDUMP ==="
llvm-objdump --version | head -3

echo
echo "=== ARQUITECTURAS DISPONIBLES ==="
llvm-objdump --version | grep -i "registered targets" -A2

echo
echo "=== DESENSAMBLADO DE CALCULARCLICK ==="

llvm-objdump \
  --arch-name=i386 \
  --adjust-vma=0x1DC3 \
  --disassemble-all \
  ~/armon_code/procedimientos/1DC3_CalcularClick.bin
```


### Pero hay algo todavía más interesante


Los bytes:


```
55 89 E5
```


son:


```
asmpush bp
mov  bp, sp
```


y luego:


```
B8 08 00
```


es:


```
asmmov ax, 0008h
```


Después aparece:


```
9A 44 04 02 1E
```


que tiene la forma de una instrucción **`CALL FAR`**:


```
asmcall 1E02:0444
```


Esto es exactamente el tipo de código que esperaríamos de un programa Delphi 1 para Windows 3.x: procedimientos 16-bit segmentados realizando llamadas FAR entre segmentos.


Y al final:


```
C9 CA 08 00
```


corresponde a:


```
asmleave
ret 0008h
```


Eso es particularmente valioso porque nos confirma que **55 bytes constituyen realmente un procedimiento completo**, no simplemente un fragmento arbitrario.


Así que ya estamos pasando de:


**"encontramos cadenas de nombres Delphi"**


a:


**"tenemos los procedimientos compilados correspondientes a esos nombres y podemos empezar a reconstruir su lógica".**


El resultado de ese primer `llvm-objdump` nos permitirá verificar exactamente cómo LLVM interpreta estas instrucciones 16-bit antes de procesar los 22 procedimientos.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ echo "=== LLVM OBJDUMP ==="
=== LLVM OBJDUMP ===
.../Download/armon $ llvm-objdump --version | head -3
LLVM (http://llvm.org/):
  LLVM version 21.1.8
  Optimized build.
.../Download/armon $
.../Download/armon $ echo

.../Download/armon $ echo "=== ARQUITECTURAS DISPONIBLES ==="
=== ARQUITECTURAS DISPONIBLES ===
.../Download/armon $ llvm-objdump --version | grep -i "registered targets" -A2
  Registered Targets:
    aarch64     - AArch64 (little endian)
    aarch64_32  - AArch64 (little endian ILP32)
.../Download/armon $
.../Download/armon $ echo

.../Download/armon $ echo "=== DESENSAMBLADO DE CALCULARCLICK ==="
=== DESENSAMBLADO DE CALCULARCLICK ===
.../Download/armon $
.../Download/armon $ llvm-objdump \
>   --arch-name=i386 \
>   --adjust-vma=0x1DC3 \
>   --disassemble-all \
>   ~/armon_code/procedimientos/1DC3_CalcularClick.bin
llvm-objdump: error: '/data/data/com.termux/files/home/armon_code/procedimientos/1DC3_CalcularClick.bin': The file was not recognized as a valid object file
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Sí. El diagnóstico ya está claro: **LLVM tampoco acepta directamente ese `.bin` como objeto**, porque el archivo es solamente código crudo, no un ELF/COFF/Mach-O.


No necesitamos convertirlo ni modificar el EXE. Para este caso es mejor usar un desensamblador de bytes crudos. Antes de instalar nada, comprobemos si Termux ya tiene alguna herramienta adecuada (`ndisasm`, `radare2`, `rizin`, etc.).


Ejecuta este único bloque:


```
Bashcd /storage/emulated/0/Download/armon

echo "=== DESENSAMBLADORES DISPONIBLES ==="

for x in ndisasm r2 rizin objdump llvm-objdump llvm-mc; do
    if command -v "$x" >/dev/null 2>&1; then
        echo "$x -> $(command -v "$x")"
    else
        echo "$x -> NO"
    fi
done

echo
echo "=== LLVM-MC ==="

if command -v llvm-mc >/dev/null 2>&1; then
    llvm-mc --version | head -8
fi

echo
echo "=== PAQUETES TERMUX RELACIONADOS ==="

pkg list-installed 2>/dev/null | grep -Ei 'binutils|llvm|radare|rizin|nasm|yasm' || true
```


Con esto **no vamos a tocar ARMÓN.EXE ni los procedimientos que ya extrajimos**.


Y una corrección importante de mi respuesta anterior: aunque `55 89 E5` confirma fuertemente el prólogo `push bp / mov bp,sp`, todavía **no debemos afirmar que `C9 CA 08 00` sea necesariamente `leave / ret 0008h`** hasta desensamblar correctamente el bloque como código 16-bit. La evidencia es muy prometedora, pero vamos a verificarla con un desensamblador apropiado antes de reconstruir el Pascal.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ echo "=== DESENSAMBLADORES DISPONIBLES ==="
=== DESENSAMBLADORES DISPONIBLES ===
.../Download/armon $
.../Download/armon $ for x in ndisasm r2 rizin objdump llvm-objdump llvm-mc; do
>     if command -v "$x" >/dev/null 2>&1; then
>         echo "$x -> $(command -v "$x")"
>     else
>         echo "$x -> NO"
>     fi
> done
truendisasm -> NO
r2 -> NO
rizin -> NO
objdump -> /data/data/com.termux/files/usr/bin/objdump
llvm-objdump -> /data/data/com.termux/files/usr/bin/llvm-objdump
llvm-mc -> /data/data/com.termux/files/usr/bin/llvm-mc
.../Download/armon $
.../Download/armon $ echo

.../Download/armon $ echo "=== LLVM-MC ==="
=== LLVM-MC ===
.../Download/armon $
.../Download/armon $ if command -v llvm-mc >/dev/null 2>&1; then
>     llvm-mc --version | head -8
> fi
LLVM (http://llvm.org/):
  LLVM version 21.1.8
  Optimized build.

  Registered Targets:
    aarch64     - AArch64 (little endian)
    aarch64_32  - AArch64 (little endian ILP32)
    aarch64_be  - AArch64 (big endian)
.../Download/armon $
.../Download/armon $ echo

.../Download/armon $ echo "=== PAQUETES TERMUX RELACIONADOS ==="
=== PAQUETES TERMUX RELACIONADOS ===
.../Download/armon $
.../Download/armon $ pkg list-installed 2>/dev/null | grep -Ei 'binutils|llvm|radare|rizin|nasm|yasm' || true
binutils/stable,now 2.47 aarch64 [installed]
libllvm/stable,now 21.1.8-3 aarch64 [installed,automatic]
llvm-mingw-w64-libcompiler-rt/stable,now 20251216 all [installed,automatic]
llvm-mingw-w64-tools/stable,now 14.0.0+dev aarch64 [installed,automatic]
llvm-mingw-w64-ucrt/stable,now 20251216 all [installed,automatic]
llvm-mingw-w64/stable,now 20251216 all [installed]
llvm-tools/stable,now 21.1.8-3 aarch64 [installed,automatic]
llvm/stable,now 21.1.8-3 aarch64 [installed,automatic]
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Perfecto. Ya encontramos la solución correcta: **`llvm-mc` sí puede desensamblar una cadena de bytes hexadecimal**, y LLVM dispone explícitamente del modo `16bit-mode` para i8086.


Esto es mejor que intentar convertir los `.bin` en objetos.


Vamos a probar primero **solamente `CalcularClick`**. El siguiente comando convierte sus 55 bytes a hexadecimal y se los entrega directamente a `llvm-mc` en modo 16 bits.


```
Bashcd /storage/emulated/0/Download/armon

echo "============================================================"
echo " DESENSAMBLADO 16-BIT DE CALCULARCLICK"
echo "============================================================"

python3 - <<'PY' | llvm-mc --disassemble --triple=i386-unknown-none-code16 --x86-asm-syntax=intel
from pathlib import Path

f = Path.home() / "armon_code/procedimientos/1DC3_CalcularClick.bin"

data = f.read_bytes()

print(" ".join("0x%02X" % b for b in data))
PY
```


### Qué estamos haciendo


Los bytes originales:


```
55 89 E5 B8 08 00 9A 44 04 02 1E ...
```


entran directamente al desensamblador.


El `triple`:


```
i386-unknown-none-code16
```


es importante porque LLVM reconoce `CODE16` como modo de 16 bits y activa `16bit-mode`.


Y usamos sintaxis Intel para que la lectura sea más natural:


```
asmpush bp
mov  bp, sp
...
```


**No vamos a generar todavía los 22 desensamblados.** Primero debemos comprobar que `llvm-mc` interpreta correctamente este procedimiento de 55 bytes. Si la salida es coherente, entonces automatizamos todos los procedimientos y podremos comenzar la reconstrucción de `CalcularClick` en Pascal/Delphi.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ echo "============================================================"
============================================================
.../Download/armon $ echo " DESENSAMBLADO 16-BIT DE CALCULARCLICK"
 DESENSAMBLADO 16-BIT DE CALCULARCLICK
.../Download/armon $ echo "============================================================"
============================================================
.../Download/armon $
.../Download/armon $ python3 - <<'PY' | llvm-mc --disassemble --triple=i386-unknown-none-code16 --x86-asm-syntax=intel
> from pathlib import Path
>
> f = Path.home() / "armon_code/procedimientos/1DC3_CalcularClick.bin"
>
> data = f.read_bytes()
>
> print(" ".join("0x%02X" % b for b in data))
> PY
        pushw   %bp
        movw    %sp, %bp
        movw    $8, %ax
        lcallw  $7682, $1092                    # imm = 0x1E02
                                        # imm = 0x444
        subw    $8, %sp
        lesw    6(%bp), %di
        pushw   %es
        pushw   %di
        lcallw  $7659, $6388                    # imm = 0x1DEB
                                        # imm = 0x18F4
        cltd
        movw    %ax, 20828
        movw    %dx, 20830
        lesw    6(%bp), %di
        pushw   %es
        pushw   %di
        lcallw  $8109, $6313                    # imm = 0x1FAD
                                        # imm = 0x18A9
        cltd
        movw    %ax, 20824
        movw    %dx, 20826
        leave
        lretw   $8
<stdin>:1:271: warning: invalid instruction encoding
0x55 0x89 0xE5 0xB8 0x08 0x00 0x9A 0x44 0x04 0x02 0x1E 0x83 0xEC 0x08 0xC4 0x7E 0x06 0x06 0x57 0x9A 0xF4 0x18 0xEB 0x1D 0x99 0xA3 0x5C 0x51 0x89 0x16 0x5E 0x51 0xC4 0x7E 0x06 0x06 0x57 0x9A 0xA9 0x18 0xAD 0x1F 0x99 0xA3 0x58 0x51 0x89 0x16 0x5A 0x51 0xC9 0xCA 0x08 0x00 0x00
                                                                                                                                                                                                                                                                              ^
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Sí. **Ahora tenemos el desensamblado correcto.** La advertencia final se debe a que el archivo contiene **un byte `00` sobrante después del `lretw $8`**; no afecta al procedimiento.


Y el resultado es muy bueno: `CalcularClick` está completamente delimitado y tiene una estructura muy clara.


### `CalcularClick` reconstruido


```
asmpush bp
mov  bp, sp
mov  ax, 8

call far 1E02:0444

sub  sp, 8

les  di, [bp+6]
push es
push di

call far 18F4:1DEB

cwd
mov  [515C], ax
mov  [515E], dx

les  di, [bp+6]
push es
push di

call far 18A9:1FA9

cwd
mov  [5158], ax
mov  [515A], dx

leave
ret 8
```


Hay una observación importante: LLVM muestra los operandos del `lcallw` en el orden **segmento:offset**, pero debemos conservar los bytes originales para reconstruir exactamente las direcciones FAR.


Por ejemplo:


```
9A 44 04 02 1E
```


se interpreta como:


```
offset = 0x0444
segment = 0x1E02
```


por tanto:


```
CALL FAR 1E02:0444
```


### Lo más interesante


El procedimiento tiene una forma típica de Delphi:


```
entrada
  ↓
push bp
mov bp,sp
  ↓
llamada a otra rutina
  ↓
trabajo con parámetro [BP+6]
  ↓
dos llamadas FAR
  ↓
guardar resultados
  ↓
leave
ret 8
```


Y `ret 8` nos dice que el procedimiento recibe **8 bytes de parámetros en la pila**.


Además, aparecen dos resultados de 32 bits:


```
DX:AX → [515C:515E]
DX:AX → [5158:515A]
```


porque:


```
asmcwd
mov [515C], ax
mov [515E], dx
```


y:


```
asmcwd
mov [5158], ax
mov [515A], dx
```


Eso empieza a permitirnos reconstruir **la semántica**, no solo las instrucciones.


### Pero hay una corrección importante


No debemos llamar todavía a esas direcciones:


```
1E02:0444
1DEB:18F4
18A9:1FA9
```


"funciones Delphi" con nombre conocido. Son **llamadas FAR a otras rutinas**, pero todavía tenemos que localizar sus destinos en los segmentos NE y determinar qué rutina contienen.


Eso es precisamente lo siguiente que conviene hacer.


**Ya podemos pasar de analizar nombres a reconstruir el grafo de llamadas del programa:** `CalcularClick` → sus tres rutinas FAR → las rutinas que ellas llaman. Eso puede llevarnos mucho más cerca de recuperar la lógica original de ARMÓN.

## Usuario

Que sigue

## Asistente

Lo siguiente es **resolver las llamadas FAR de `CalcularClick` y comprobar a qué segmentos/rutinas del NE apuntan**. Ese es el paso que convierte el desensamblado aislado en una reconstrucción del programa.


Tenemos estas tres llamadas:


```
1E02:0444
1DEB:18F4
18A9:1FA9
```


Pero hay una particularidad: en un ejecutable **NE de Windows 3.1**, el `segment` de una llamada FAR no necesariamente es el número de segmento NE directamente. Primero debemos mapearlo correctamente mediante la tabla de segmentos/relocaciones.


### Paso inmediato


Vamos a hacer tres cosas automáticamente:


1. Leer la tabla de segmentos NE.
2. Buscar dónde aparecen esas referencias FAR.
3. Determinar qué segmento físico corresponde a cada llamada.
4. Extraer el código destino.
5. Desensamblarlo también en modo 16-bit.


Ejecuta **todo este bloque único**:


```
Bashcd /storage/emulated/0/Download/armon

echo "============================================================"
echo " RESOLVIENDO LLAMADAS FAR DE CALCULARCLICK"
echo "============================================================"

python3 - <<'PY'
from pathlib import Path
import struct

exe = Path("/storage/emulated/0/Download/armon/ARMON.EXE")
data = exe.read_bytes()

NE = 0x100

# Cabecera NE
seg_count = struct.unpack_from("<H", data, NE + 0x1C)[0]
seg_table_rel = struct.unpack_from("<H", data, NE + 0x22)[0]
seg_table = NE + seg_table_rel
shift = struct.unpack_from("<H", data, NE + 0x32)[0]

print(f"Segmentos NE : {seg_count}")
print(f"Tabla        : 0x{seg_table:05X}")
print(f"Sector shift : {shift}")
print()

# Tabla de segmentos
segments = {}

for n in range(1, seg_count + 1):
    p = seg_table + (n - 1) * 8

    off_units, length, flags, minalloc = struct.unpack_from(
        "<HHHH", data, p
    )

    file_off = off_units << shift

    # Longitud 0 significa 64K
    real_len = length if length else 0x10000

    segments[n] = {
        "file_off": file_off,
        "length": real_len,
        "flags": flags,
        "minalloc": minalloc,
    }

print("TABLA DE SEGMENTOS")
print("------------------")

for n, s in segments.items():
    print(
        f"{n:02d}: "
        f"archivo=0x{s['file_off']:06X} "
        f"tam=0x{s['length']:04X} "
        f"flags=0x{s['flags']:04X}"
    )

print()
print("============================================================")
print(" BUSCANDO REFERENCIAS FAR")
print("============================================================")

# Bytes de las tres llamadas:
# 9A offset_lo offset_hi segment_lo segment_hi

calls = [
    ("CALL 1", 0x0444, 0x1E02),
    ("CALL 2", 0x18F4, 0x1DEB),
    ("CALL 3", 0x18A9, 0x1FA9),
]

for name, off, seg in calls:

    print()
    print(f"{name}: {seg:04X}:{off:04X}")

    # Intentamos interpretar seg como número de segmento NE
    if seg in segments:
        s = segments[seg]

        print(
            f"  -> segmento NE {seg}"
        )
        print(
            f"  -> archivo 0x{s['file_off']:06X}"
        )
        print(
            f"  -> longitud 0x{s['length']:04X}"
        )

        if off < s["length"]:
            file_pos = s["file_off"] + off
            print(
                f"  -> destino archivo 0x{file_pos:06X}"
            )

            chunk = data[file_pos:file_pos + 64]

            print("  -> bytes:")
            for i in range(0, len(chunk), 16):
                print(
                    f"     {file_pos+i:06X}: "
                    + " ".join(f"{b:02X}" for b in chunk[i:i+16])
                )

        else:
            print("  -> OFFSET FUERA DEL SEGMENTO")

    else:
        print(
            f"  -> 0x{seg:04X} NO coincide directamente "
            f"con un número de segmento NE"
        )

print()
print("============================================================")
print(" FIN")
print("============================================================")
PY
```


### ¿Por qué este paso?


Porque si conseguimos resolver, por ejemplo:


```
CALL FAR 1E02:0444
        ↓
segmento NE  ...
        ↓
offset ...
        ↓
rutina Delphi
```


podremos empezar a construir algo como:


```
CalcularClick
 ├── rutina A
 │    ├── rutina C
 │    └── rutina D
 │
 ├── rutina B
 │    └── rutina E
 │
 └── rutina F
```


Ese **grafo de llamadas** es mucho más valioso que simplemente desensamblar miles de bytes.


Y una vez identificado el código, podremos intentar traducirlo progresivamente de:


**ensamblador → operaciones → variables → lógica → pseudocódigo Pascal/Delphi**.


No intentaría todavía generar un `.pas` completo: primero debemos resolver correctamente las referencias FAR.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
x100

# Cabecera NE
seg_count = struct.unpack_from("<H", data, NE + 0x1C)[0]
seg_table_rel = struct.unpack_from("<H", data, NE + 0x22)[0]
seg_table = NE + seg_table_rel
shift = struct.unpack_from("<H", data, NE + 0x32)[0]

print(f"Segmentos NE : {seg_count}")
print(f"Tabla        : 0x{seg_table:05X}")
print(f"Sector shift : {shift}")
print()

# Tabla de segmentos
segments = {}

for n in range(1, seg_count + 1):
    p = seg_table + (n - 1) * 8

    off_units, length, flags, minalloc = struct.unpack_from(
        "<HHHH", data, p
    )

    file_off = off_units << shift

    # Longitud 0 significa 64K
    real_len = length if length else 0x10000

    segments[n] = {
        "file_off": file_off,
        "length": real_len,
        "flags": flags,
        "minalloc": minalloc,
    }

print("TABLA DE SEGMENTOS")
print("------------------")

for n, s in segments.items():
    print(
        f"{n:02d}: "
        f"archivo=0x{s['file_off']:06X} "
        f"tam=0x{s['length']:04X} "
        f"flags=0x{s['flags']:04X}"
    )

print()
print("============================================================")
print(" BUSCANDO REFERENCIAS FAR")
print("============================================================")

# Bytes de las tres llamadas:
# 9A offset_lo offset_hi segment_lo segment_hi

calls = [
    ("CALL 1", 0x0444, 0x1E02),
    ("CALL 2", 0x18F4, 0x1DEB),
    ("CALL 3", 0x18A9, 0x1FA9),
]

for name, off, seg in calls:

    print()
    print(f"{name}: {seg:04X}:{off:04X}")

    # Intentamos interpretar seg como número de segmento NE
    if seg in segments:
        s = segments[seg]

        print(
            f"  -> segmento NE {seg}"
        )
        print(
            f"  -> archivo 0x{s['file_off']:06X}"
        )
        print(
            f"  -> longitud 0x{s['length']:04X}"
        )

        if off < s["length"]:
            file_pos = s["file_off"] + off
            print(
                f"  -> destino archivo 0x{file_pos:06X}"
            )

            chunk = data[file.../Download/armon $
  .../Download/armon $ echo "============================================================"
============================================================
.../Download/armon $ echo " RESOLVIENDO LLAMADAS FAR DE CALCULARCLICK"
 RESOLVIENDO LLAMADAS FAR DE CALCULARCLICK
.../Download/armon $ echo "============================================================"
============================================================
.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> from pathlib import Path
> import struct
>
> exe = Path("/storage/emulated/0/Download/armon/ARMON.EXE")
      for i in range(0, len(ch> data = exe.read_bytes()
>
> NE = 0x100
>
> # Cabecera NE
> seg_count = struct.unpack_from("<H", data, NE + 0x1C)[0]
> seg_table_rel = struct.unpack_from("<H", data, NE + 0x22)[0]
> seg_table = NE + seg_table_rel
> shift = struct.unpack_from("<H", data, NE + 0x32)[0]
>
> print(f"Segmentos NE : {seg_count}")
> print(f"Tabla        : 0x{seg_table:05X}")
> print(f"Sector shift : {shift}")
> print()
>
> # Tabla de segmentos
> segments = {}
>
> for n in range(1, seg_count + 1):
>     p = seg_table + (n - 1) * 8
>
>     off_units, length, flags, minalloc = struct.unpack_from(
>         "<HHHH", data, p
>     )
>
>     file_off = off_units << shift
>
>     # Longitud 0 significa 64K
>     real_len = length if length else 0x10000
>
>     segments[n] = {
>         "file_off": file_off,
>         "length": real_len,
>         "flags": flags,
>         "minalloc": minalloc,
>     }
>
> print("TABLA DE SEGMENTOS")
> print("------------------")
>
> for n, s in segments.items():
>     print(
>         f"{n:02d}: "
>         f"archivo=0x{s['file_off']:06X} "
>         f"tam=0x{s['length']:04X} "
>         f"flags=0x{s['flags']:04X}"
>     )
>
> print()
> print("============================================================")
> print(" BUSCANDO REFERENCIAS FAR")
> print("============================================================")
>
> # Bytes de las tres llamadas:
> # 9A offset_lo offset_hi segment_lo segment_hi
>
> calls = [
>     ("CALL 1", 0x0444, 0x1E02),
>     ("CALL 2", 0x18F4, 0x1DEB),
>     ("CALL 3", 0x18A9, 0x1FA9),
> ]
>
> for name, off, seg in calls:
>
>     print()
>     print(f"{name}: {seg:04X}:{off:04X}")
>
>     # Intentamos interpretar seg como número de segmen
to NE
>     if seg in segments:
>         s = segments[seg]
>
>         print(
>             f"  -> segmento NE {seg}"
>         )
>         print(
>             f"  -> archivo 0x{s['file_off']:06X}"
>         )
>         print(
>             f"  -> longitud 0x{s['length']:04X}"
>         )
>
>         if off < s["length"]:
>             file_pos = s["file_off"] + off
>             print(
>                 f"  -> destino archivo 0x{file_pos:06X}"
>             )
>
>             chunk = data[file_pos:file_pos + 64]
>
>             print("  -> bytes:")
>             for i in range(0, len(chunk), 16):
>                 print(
>                     f"     {file_pos+i:06X}: "
>                     + " ".join(f"{b:02X}" for b in chunk[i:i+16])
>                 )
>
>         else:
>             print("  -> OFFSET FUERA DEL SEGMENTO")
>
>     else:
>         print(
>             f"  -> 0x{seg:04X} NO coincide directamente "
>             f"con un número de segmento NE"
>         )
>
> print()
> print("============================================================")
> print(" FIN")
> print("============================================================")
> PY
Segmentos NE : 81
Tabla        : 0x00140
Sector shift : 6

TABLA DE SEGMENTOS
------------------
01: archivo=0x037940 tam=0x3FF6 flags=0x1D10
02: archivo=0x03CA40 tam=0x9D2D flags=0x1D10
03: archivo=0x04B2C0 tam=0x6ED0 flags=0x1D10
04: archivo=0x052940 tam=0x3F50 flags=0x1D10
05: archivo=0x056A80 tam=0xAF25 flags=0x1D10
06: archivo=0x063BC0 tam=0x3F82 flags=0x1D10
07: archivo=0x068D80 tam=0x62D6 flags=0x1D10
08: archivo=0x070E00 tam=0x80ED flags=0x1D10
09: archivo=0x07C400 tam=0xF0E4 flags=0x1D10
10: archivo=0x090B40 tam=0xF03D flags=0x1D10
11: archivo=0x0A6240 tam=0x4DF7 flags=0x1D10
12: archivo=0x0AD180 tam=0x99C4 flags=0x1D10
13: archivo=0x0BA640 tam=0xBF4D flags=0x1D10
14: archivo=0x0CB740 tam=0x3EA7 flags=0x1D10
15: archivo=0x0D0FC0 tam=0xD32A flags=0x1D10
16: archivo=0x0E3880 tam=0x78F0 flags=0x1D10
17: archivo=0x0ED500 tam=0x7983 flags=0x1D10
18: archivo=0x0F7E40 tam=0x86D4 flags=0x1D10
19: archivo=0x104240 tam=0xB1FE flags=0x1D10
20: archivo=0x112D40 tam=0x8808 flags=0x1D10
21: archivo=0x120D80 tam=0x4305 flags=0x1D10
22: archivo=0x126A00 tam=0xEA0E flags=0x1D10
23: archivo=0x139480 tam=0x9B80 flags=0x1D10
24: archivo=0x145E00 tam=0x7414 flags=0x1D10
25: archivo=0x14F040 tam=0x3D29 flags=0x1D10
26: archivo=0x153A40 tam=0xCDC9 flags=0x1D10
27: archivo=0x165400 tam=0x4DB5 flags=0x1D10
28: archivo=0x16BC80 tam=0xEDC4 flags=0x1D10
29: archivo=0x180140 tam=0xC991 flags=0x1D10
30: archivo=0x1916C0 tam=0xD12F flags=0x1D10
31: archivo=0x1A44C0 tam=0x4469 flags=0x1D10
32: archivo=0x1A96C0 tam=0x3BC5 flags=0x1D10
33: archivo=0x1AEE80 tam=0x3B3B flags=0x1D10
34: archivo=0x1B2EC0 tam=0x7EBC flags=0x1D10
35: archivo=0x1BD840 tam=0xC336 flags=0x1D10
36: archivo=0x1CDB80 tam=0x3BDD flags=0x1D10
37: archivo=0x1D28C0 tam=0xE0C1 flags=0x1D10
38: archivo=0x1E7B00 tam=0x41F8 flags=0x1D10
39: archivo=0x1EEF00 tam=0xC820 flags=0x1D10
40: archivo=0x1FFB80 tam=0x53E6 flags=0x1D10
41: archivo=0x206B80 tam=0x5C50 flags=0x1D10
42: archivo=0x20E380 tam=0x3EF9 flags=0x1D10
43: archivo=0x213300 tam=0x4874 flags=0x1D10
44: archivo=0x219B40 tam=0x3F08 flags=0x1D10
45: archivo=0x21F980 tam=0x7B16 flags=0x1D10
46: archivo=0x228FC0 tam=0xF600 flags=0x1D10
47: archivo=0x23E040 tam=0x5A44 flags=0x1D10
48: archivo=0x245C40 tam=0xB3F1 flags=0x1D10
49: archivo=0x2563C0 tam=0x79D0 flags=0x1D10
50: archivo=0x25FB00 tam=0x3C8E flags=0x1D10
51: archivo=0x264640 tam=0x3AB4 flags=0x1D10
52: archivo=0x268640 tam=0x4147 flags=0x1D10
53: archivo=0x26CCC0 tam=0x3DB8 flags=0x1D10
54: archivo=0x271FC0 tam=0x45D0 flags=0x1D10
55: archivo=0x2784C0 tam=0xCFF6 flags=0x1D10
56: archivo=0x288F80 tam=0x3D77 flags=0x1D10
57: archivo=0x28E800 tam=0x3B9B flags=0x1D10
58: archivo=0x292BC0 tam=0xBED1 flags=0x1D10
59: archivo=0x2A5B00 tam=0x3FF4 flags=0x1D10
60: archivo=0x2AAEC0 tam=0x8732 flags=0x1D10
61: archivo=0x2BD900 tam=0x3FD5 flags=0x1D10
62: archivo=0x2C3180 tam=0x3D11 flags=0x1D10
63: archivo=0x2C7340 tam=0x30D2 flags=0x1D10
64: archivo=0x2CB180 tam=0x2670 flags=0x1D10
65: archivo=0x2D0080 tam=0x390C flags=0x1D10
66: archivo=0x2D6F80 tam=0x43D6 flags=0x1D10
67: archivo=0x2DC340 tam=0x376F flags=0x1D10
68: archivo=0x2E1A80 tam=0x9AD0 flags=0x1D10
69: archivo=0x2EB6C0 tam=0x2876 flags=0x1D10
70: archivo=0x000900 tam=0x3879 flags=0x1D50
71: archivo=0x0043C0 tam=0x2FC1 flags=0x1D50
72: archivo=0x007400 tam=0x2540 flags=0x1D50
73: archivo=0x009A00 tam=0x8716 flags=0x1D50
74: archivo=0x012280 tam=0x722E flags=0x1D50
75: archivo=0x019740 tam=0x22CC flags=0x1D50
76: archivo=0x01BAC0 tam=0x6EB3 flags=0x1D50
77: archivo=0x022C00 tam=0x7AAA flags=0x1D50
78: archivo=0x02A9C0 tam=0x51C4 flags=0x1D50
79: archivo=0x02FC80 tam=0x2AC9 flags=0x0D50
80: archivo=0x032A00 tam=0x261A flags=0x0D50
81: archivo=0x035700 tam=0x21E4 flags=0x0D59

============================================================
 BUSCANDO REFERENCIAS FAR
============================================================

CALL 1: 1E02:0444
  -> 0x1E02 NO coincide directamente con un número de segmento NE

CALL 2: 1DEB:18F4
  -> 0x1DEB NO coincide directamente con un número de segmento NE

CALL 3: 1FA9:18A9
  -> 0x1FA9 NO coincide directamente con un número de segmento NE

============================================================
 FIN
============================================================
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

El script funcionó correctamente. El resultado nos muestra exactamente dónde estaba la trampa.


En un ejecutable **NE**, una instrucción como:


```
9A 44 04 02 1E
```


no debe interpretarse simplemente como:


```
segmento NE = 0x1E02
offset = 0x0444
```


El `0x1E02` es un **valor que el linker dejó para ser corregido mediante una relocación**. Windows lo transforma en el selector/segmento real cuando carga el módulo.


Por eso:


```
1E02
1DEB
1FA9
```


no aparecen entre los segmentos 1–81.


### El siguiente paso correcto


Debemos analizar la **tabla de relocaciones del segmento 3**, porque ahí están precisamente las instrucciones de `CalcularClick`.


Sabemos:


```
Segmento 3
inicio archivo = 0x04B2C0
tamaño código  = 0x6ED0
fin            = 0x052190
```


y el segmento 4 comienza en:


```
0x052940
```


Por tanto existe una zona intermedia que debemos inspeccionar:


```
0x052190 → 0x05293F
```


Es muy probable que contenga las relocaciones asociadas al segmento 3.


**No debemos seguir intentando convertir `1E02`, `1DEB` y `1FA9` directamente a números de segmento.**


Lo que sigue es:


```
CalcularClick
      │
      ├── CALL FAR 1E02:0444
      │        ↓
      │   tabla de relocación
      │        ↓
      │   segmento destino real
      │
      ├── CALL FAR 1DEB:18F4
      │        ↓
      │   tabla de relocación
      │
      └── CALL FAR 1FA9:18A9
               ↓
          tabla de relocación
```


Y una vez resueltos esos tres destinos podremos desensamblar **las rutinas realmente llamadas por `CalcularClick`**.


### Hagamos ahora únicamente la inspección de esa zona


Así evitamos modificar nada del ejecutable:


```
Bashcd /storage/emulated/0/Download/armon

echo "============================================================"
echo " ZONA POSTERIOR AL SEGMENTO 3"
echo "============================================================"

python3 - <<'PY'
from pathlib import Path

data = Path("/storage/emulated/0/Download/armon/ARMON.EXE").read_bytes()

inicio = 0x052190
fin    = 0x052940

print(f"Inicio : 0x{inicio:06X}")
print(f"Fin    : 0x{fin:06X}")
print(f"Tamaño : 0x{fin-inicio:X} ({fin-inicio} bytes)")
print()

for p in range(inicio, fin, 16):
    b = data[p:min(p+16, fin)]
    print(f"{p:06X}: " + " ".join(f"{x:02X}" for x in b))
PY

echo
echo "============================================================"
echo " FIN"
echo "============================================================"
```


**No necesitamos todavía desensamblar esa zona.** Primero quiero ver su estructura hexadecimal, porque así podremos identificar exactamente dónde empieza la tabla de relocaciones del segmento 3 y después escribir un parser específico para ella.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ echo "============================================================"
============================================================
.../Download/armon $ echo " ZONA POSTERIOR AL SEGMENTO 3"
 ZONA POSTERIOR AL SEGMENTO 3
.../Download/armon $ echo "============================================================"
============================================================
.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> from pathlib import Path
>
> data = Path("/storage/emulated/0/Download/armon/ARMON.EXE").read_bytes()
>
> inicio = 0x052190
> fin    = 0x052940
>
> print(f"Inicio : 0x{inicio:06X}")
> print(f"Fin    : 0x{fin:06X}")
> print(f"Tamaño : 0x{fin-inicio:X} ({fin-inicio} bytes)
")
> print()
>
> for p in range(inicio, fin, 16):
>     b = data[p:min(p+16, fin)]
>     print(f"{p:06X}: " + " ".join(f"{x:02X}" for x in b))
> PY
Inicio : 0x052190
Fin    : 0x052940
Tamaño : 0x7B0 (1968 bytes)

052190: F4 00 02 00 90 00 4D 00 00 00 02 00 78 00 4C 00
0521A0: 00 00 02 00 34 00 4E 00 00 00 02 00 1C 00 50 00
0521B0: 00 00 02 00 A9 00 03 00 00 00 02 00 E2 0F 48 00
0521C0: 00 00 02 00 EA 0F 4B 00 00 00 02 00 EE 0F 46 00
0521D0: 00 00 02 00 FE 0F 49 00 00 00 02 00 06 10 44 00
0521E0: 00 00 02 00 12 10 43 00 00 00 02 00 90 10 3A 00
0521F0: 00 00 02 00 8E 13 4F 00 00 00 02 00 7E 14 05 00
052200: 00 00 02 00 C3 14 3E 00 00 00 05 07 D9 14 05 00
052210: 00 00 05 07 DE 14 05 00 00 00 05 07 E6 14 04 00
052220: 00 00 05 07 F0 14 02 00 00 00 05 07 F4 14 06 00
052230: 00 00 02 00 40 15 2D 00 00 00 02 00 36 16 18 00
052240: 00 00 02 00 BE 16 4A 00 00 00 05 07 AA 17 05 00
052250: 00 00 05 07 AF 17 03 00 00 00 05 07 B5 17 05 00
052260: 00 00 05 07 C4 17 05 00 00 00 05 07 C9 17 03 00
052270: 00 00 05 07 CF 17 05 00 00 00 05 07 E6 17 05 00
052280: 00 00 05 07 EB 17 03 00 00 00 05 07 F1 17 05 00
052290: 00 00 02 00 B0 18 42 00 00 00 05 07 26 19 05 00
0522A0: 00 00 05 07 2B 19 03 00 00 00 05 07 31 19 05 00
0522B0: 00 00 02 00 25 1C 32 00 00 00 02 00 2F 1C 3D 00
0522C0: 00 00 05 07 4E 1F 05 00 00 00 05 07 52 1F 05 00
0522D0: 00 00 05 07 57 1F 06 00 00 00 05 07 70 1F 05 00
0522E0: 00 00 05 07 75 1F 05 00 00 00 02 00 67 24 2F 00
0522F0: 00 00 02 00 5C 25 31 00 00 00 02 00 D1 25 2A 00
052300: 00 00 02 00 F7 26 33 00 00 00 05 07 25 28 03 00
052310: 00 00 05 07 2B 28 05 00 00 00 05 07 30 28 06 00
052320: 00 00 02 00 4D 28 01 00 00 00 02 00 7F 28 3F 00
052330: 00 00 05 07 4A 2A 05 00 00 00 05 07 4F 2A 06 00
052340: 00 00 02 00 84 2A 36 00 00 00 05 07 86 2A 05 00
052350: 00 00 05 07 A6 2A 03 00 00 00 05 07 AC 2A 05 00
052360: 00 00 05 07 B1 2A 05 00 00 00 05 07 B4 2A 05 00
052370: 00 00 05 07 B9 2A 06 00 00 00 05 07 C2 2A 05 00
052380: 00 00 05 07 C6 2A 03 00 00 00 05 07 CC 2A 05 00
052390: 00 00 05 07 D1 2A 05 00 00 00 05 07 D6 2A 06 00
0523A0: 00 00 05 07 DF 2A 05 00 00 00 05 07 E3 2A 03 00
0523B0: 00 00 05 07 E9 2A 05 00 00 00 05 07 EE 2A 05 00
0523C0: 00 00 05 07 F3 2A 06 00 00 00 05 07 FC 2A 05 00
0523D0: 00 00 05 07 00 2B 03 00 00 00 05 07 06 2B 05 00
0523E0: 00 00 05 07 0B 2B 05 00 00 00 05 07 10 2B 06 00
0523F0: 00 00 05 07 19 2B 05 00 00 00 05 07 1E 2B 05 00
052400: 00 00 05 07 22 2B 05 00 00 00 05 07 27 2B 06 00
052410: 00 00 05 07 33 2B 05 00 00 00 05 07 A8 2B 05 00
052420: 00 00 05 07 AD 2B 06 00 00 00 05 07 B5 2B 05 00
052430: 00 00 05 07 F0 2B 05 00 00 00 05 07 F5 2B 06 00
052440: 00 00 05 07 54 2D 05 00 00 00 05 07 5C 2D 02 00
052450: 00 00 05 07 60 2D 06 00 00 00 05 07 92 2D 02 00
052460: 00 00 05 07 96 2D 06 00 00 00 02 00 1B 31 04 00
052470: 00 00 03 01 29 31 06 00 45 00 02 00 14 32 3B 00
052480: 00 00 05 07 8B 33 03 00 00 00 05 07 91 33 05 00
052490: 00 00 05 07 96 33 06 00 00 00 05 07 A8 33 05 00
0524A0: 00 00 05 07 B1 33 02 00 00 00 05 07 B5 33 06 00
0524B0: 00 00 05 07 C7 33 05 00 00 00 05 07 D1 33 02 00
0524C0: 00 00 05 07 D5 33 06 00 00 00 03 01 30 34 07 00
0524D0: 68 00 05 07 EF 37 03 00 00 00 05 07 F5 37 05 00
0524E0: 00 00 05 07 FA 37 06 00 00 00 05 07 FC 37 03 00
0524F0: 00 00 05 07 02 38 05 00 00 00 05 07 07 38 06 00
052500: 00 00 05 07 1F 38 03 00 00 00 05 07 25 38 05 00
052510: 00 00 05 07 2A 38 06 00 00 00 05 07 2C 38 03 00
052520: 00 00 05 07 32 38 05 00 00 00 05 07 37 38 06 00
052530: 00 00 05 07 4F 38 03 00 00 00 05 07 55 38 05 00
052540: 00 00 05 07 5A 38 06 00 00 00 05 07 5C 38 03 00
052550: 00 00 05 07 62 38 05 00 00 00 05 07 67 38 06 00
052560: 00 00 05 07 7F 38 03 00 00 00 05 07 85 38 05 00
052570: 00 00 05 07 8A 38 06 00 00 00 05 07 8C 38 03 00
052580: 00 00 05 07 92 38 05 00 00 00 05 07 97 38 06 00
052590: 00 00 05 07 AF 38 03 00 00 00 05 07 B5 38 05 00
0525A0: 00 00 05 07 BA 38 06 00 00 00 05 07 BC 38 03 00
0525B0: 00 00 05 07 C2 38 05 00 00 00 05 07 C7 38 06 00
0525C0: 00 00 05 07 E4 38 05 00 00 00 05 07 E9 38 03 00
0525D0: 00 00 05 07 EF 38 04 00 00 00 05 07 F3 38 03 00
0525E0: 00 00 05 07 F9 38 05 00 00 00 05 07 0B 39 05 00
0525F0: 00 00 05 07 10 39 03 00 00 00 05 07 19 39 04 00
052600: 00 00 05 07 1D 39 03 00 00 00 05 07 23 39 05 00
052610: 00 00 05 07 7C 3B 05 00 00 00 05 07 81 3B 03 00
052620: 00 00 05 07 87 3B 05 00 00 00 05 07 A4 3B 05 00
052630: 00 00 05 07 A9 3B 03 00 00 00 05 07 AF 3B 05 00
052640: 00 00 02 00 13 45 2C 00 00 00 05 07 5A 47 05 00
052650: 00 00 05 07 5E 47 06 00 00 00 05 07 66 47 05 00
052660: 00 00 05 07 6A 47 05 00 00 00 05 07 6F 47 06 00
052670: 00 00 05 07 9A 47 05 00 00 00 05 07 9F 47 03 00
052680: 00 00 05 07 AA 47 02 00 00 00 05 07 AE 47 06 00
052690: 00 00 05 07 EC 47 05 00 00 00 05 07 F1 47 03 00
0526A0: 00 00 05 07 F7 47 05 00 00 00 05 07 FB 47 06 00
0526B0: 00 00 05 07 0E 48 05 00 00 00 05 07 13 48 03 00
0526C0: 00 00 05 07 1E 48 02 00 00 00 05 07 22 48 06 00
0526D0: 00 00 05 07 5B 48 05 00 00 00 05 07 60 48 03 00
0526E0: 00 00 05 07 66 48 05 00 00 00 05 07 6B 48 06 00
0526F0: 00 00 05 07 BA 48 05 00 00 00 05 07 BF 48 03 00
052700: 00 00 05 07 C5 48 05 00 00 00 05 07 C9 48 06 00
052710: 00 00 05 07 D1 48 05 00 00 00 05 07 D6 48 03 00
052720: 00 00 05 07 DC 48 05 00 00 00 05 07 E1 48 06 00
052730: 00 00 05 07 59 49 05 00 00 00 05 07 5E 49 06 00
052740: 00 00 05 07 08 4B 05 00 00 00 05 07 0D 4B 06 00
052750: 00 00 02 00 6D 57 06 00 00 00 02 00 7F 5A 34 00
052760: 00 00 05 07 43 5D 05 00 00 00 05 07 48 5D 06 00
052770: 00 00 05 07 B4 66 05 00 00 00 05 07 B9 66 03 00
052780: 00 00 05 07 C4 66 02 00 00 00 05 07 C8 66 06 00
052790: 00 00 05 07 0A 67 05 00 00 00 05 07 0F 67 03 00
0527A0: 00 00 05 07 1A 67 02 00 00 00 05 07 1E 67 06 00
0527B0: 00 00 05 07 8A 67 05 00 00 00 05 07 8E 67 03 00
0527C0: 00 00 05 07 99 67 02 00 00 00 05 07 9D 67 06 00
0527D0: 00 00 05 07 0E 68 05 00 00 00 05 07 12 68 03 00
0527E0: 00 00 05 07 1D 68 02 00 00 00 05 07 21 68 06 00
0527F0: 00 00 05 07 92 68 05 00 00 00 05 07 96 68 03 00
052800: 00 00 05 07 A1 68 02 00 00 00 05 07 A5 68 06 00
052810: 00 00 05 07 16 69 05 00 00 00 05 07 1A 69 03 00
052820: 00 00 05 07 25 69 02 00 00 00 05 07 29 69 06 00
052830: 00 00 05 07 33 6B 03 00 00 00 05 07 39 6B 05 00
052840: 00 00 05 07 3E 6B 05 00 00 00 05 07 43 6B 06 00
052850: 00 00 05 07 98 6B 05 00 00 00 05 07 9D 6B 03 00
052860: 00 00 05 07 A3 6B 05 00 00 00 05 07 A7 6B 06 00
052870: 00 00 05 07 AF 6B 03 00 00 00 05 07 B5 6B 05 00
052880: 00 00 05 07 BA 6B 05 00 00 00 05 07 BF 6B 06 00
052890: 00 00 05 07 0C 6C 03 00 00 00 05 07 12 6C 05 00
0528A0: 00 00 05 07 17 6C 05 00 00 00 05 07 1C 6C 05 00
0528B0: 00 00 05 07 21 6C 06 00 00 00 05 07 76 6C 05 00
0528C0: 00 00 05 07 7B 6C 03 00 00 00 05 07 81 6C 05 00
0528D0: 00 00 05 07 85 6C 06 00 00 00 05 07 8D 6C 03 00
0528E0: 00 00 05 07 93 6C 05 00 00 00 05 07 98 6C 05 00
0528F0: 00 00 05 07 9D 6C 05 00 00 00 05 07 A2 6C 06 00
052900: 00 00 05 07 0C 6E 05 00 00 00 05 07 11 6E 06 00
052910: 00 00 05 07 65 6E 05 00 00 00 05 07 69 6E 03 00
052920: 00 00 05 07 74 6E 02 00 00 00 05 07 78 6E 06 00
052930: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
.../Download/armon $
.../Download/armon $ echo

.../Download/armon $ echo "============================================================"
============================================================
.../Download/armon $ echo " FIN"
 FIN
.../Download/armon $ echo "============================================================"
============================================================
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Esto ya revela una pieza muy importante: **sí encontramos la tabla de relocaciones del segmento 3**, pero su estructura no coincide directamente con la interpretación estándar que intentamos aplicar.


La evidencia es muy fuerte:


- Segmento 3 termina en `0x052190`.
- En `0x052190` aparece `F4 00` → **244 registros**.
- `244 × 8 = 1952 = 0x7A0`.
- Sumando los 2 bytes del contador: `0x7A2`.
- La secuencia termina prácticamente exactamente en `0x052932`.
- Desde `0x052932` hasta `0x05293F` hay ceros/padding.
- El segmento 4 empieza en `0x052940`.


Es decir, **no estamos viendo datos aleatorios**: es la tabla de relocaciones.


La documentación del formato NE confirma que, cuando un segmento tiene relocaciones, estas siguen inmediatamente a los datos del segmento y comienzan con un contador de entradas.


### Pero aparece una anomalía


El primer registro es:


```
90 00 4D 00 00 00 02 00
```


y los registros siguientes contienen patrones como:


```
00 00 02 00 78 00 4C 00
00 00 02 00 34 00 4E 00
00 00 05 07 D9 14 05 00
```


Si aplicamos literalmente la estructura estándar, algunos campos no tendrían valores válidos. **No quiero forzar una interpretación incorrecta.**


Hay otra posibilidad mucho más interesante: **Borland/Delphi está utilizando una variante de la información de relocación que debemos interpretar considerando cómo el linker Borland genera las referencias.**


### El siguiente paso


Antes de intentar resolver `1E02`, `1DEB` y `1FA9`, vamos a identificar automáticamente **qué registros de esta tabla corresponden a los tres `CALL FAR` de `CalcularClick`**.


Sabemos exactamente dónde están las palabras de segmento dentro del procedimiento:


```
CALL 1 → segmento en 0x1DCC
CALL 2 → segmento en 0x1DDE
CALL 3 → segmento en 0x1DF0
```


Así que podemos buscar esos offsets dentro de los 244 registros y averiguar cómo está codificada realmente la tabla.


Ejecuta este único bloque:


```
Bashcd /storage/emulated/0/Download/armon

python3 - <<'PY'
from pathlib import Path
import struct

data = Path("/storage/emulated/0/Download/armon/ARMON.EXE").read_bytes()

# Segmento 3
seg_start = 0x04B2C0
seg_size  = 0x6ED0
reloc     = seg_start + seg_size

count = struct.unpack_from("<H", data, reloc)[0]

print("============================================================")
print(" ANÁLISIS DE RELOCACIONES DEL SEGMENTO 3")
print("============================================================")
print(f"Tabla        : 0x{reloc:06X}")
print(f"Entradas     : {count}")
print(f"Tamaño tabla : 0x{2 + count*8:X}")
print()

# Offsets dentro del segmento donde están las palabras de SEGMENTO
objetivos = {
    "CALL 1": 0x1DCC,
    "CALL 2": 0x1DDE,
    "CALL 3": 0x1DF0,
}

print("OFFSETS QUE BUSCAMOS")
print("--------------------")

for nombre, off in objetivos.items():
    print(f"{nombre}: segmento 3 + 0x{off:04X}")

print()
print("============================================================")
print(" REGISTROS QUE CONTIENEN ESOS OFFSETS")
print("============================================================")

base = reloc + 2

encontrados = set()

for i in range(count):
    p = base + i * 8
    raw = data[p:p+8]

    # Buscar cada objetivo como WORD little-endian
    words = [
        struct.unpack_from("<H", raw, 0)[0],
        struct.unpack_from("<H", raw, 2)[0],
        struct.unpack_from("<H", raw, 4)[0],
        struct.unpack_from("<H", raw, 6)[0],
    ]

    for nombre, objetivo in objetivos.items():
        if objetivo in words:
            print()
            print(f"{nombre}")
            print(f"  registro : {i}")
            print(f"  archivo  : 0x{p:06X}")
            print(f"  bytes    : {' '.join(f'{x:02X}' for x in raw)}")
            print(f"  WORDs    : " + " ".join(f"0x{x:04X}" for x in words))
            print(f"  objetivo : 0x{objetivo:04X}")
            encontrados.add(nombre)

print()
print("============================================================")
print(" BÚSQUEDA DE LOS VALORES DE SEGMENTO 1E02 / 1DEB / 1FA9")
print("============================================================")

segmentos = {
    "CALL 1": 0x1E02,
    "CALL 2": 0x1DEB,
    "CALL 3": 0x1FA9,
}

for nombre, valor in segmentos.items():

    print()
    print(f"{nombre}: 0x{valor:04X}")

    encontrados_valor = 0

    for i in range(count):
        p = base + i * 8
        raw = data[p:p+8]

        if valor in [
            struct.unpack_from("<H", raw, 0)[0],
            struct.unpack_from("<H", raw, 2)[0],
            struct.unpack_from("<H", raw, 4)[0],
            struct.unpack_from("<H", raw, 6)[0],
        ]:
            print(
                f"  registro {i:3d} "
                f"archivo 0x{p:06X} : "
                + " ".join(f"{x:02X}" for x in raw)
            )
            encontrados_valor += 1

    print(f"  coincidencias: {encontrados_valor}")

print()
print("============================================================")
print(" RESUMEN")
print("============================================================")

for nombre in objetivos:
    print(
        f"{nombre}: "
        + ("ENCONTRADO" if nombre in encontrados else "NO ENCONTRADO")
    )
PY
```


### Qué buscamos exactamente


Si aparece algo como:


```
CALL 1
  registro : ...
  bytes    : ...
```


podremos comparar la posición con el formato real que está utilizando este ejecutable.


**Este es el punto correcto para continuar.** Ya tenemos:


**formulario Delphi → nombre del evento → procedimiento → instrucciones 16-bit → CALL FAR → relocación NE → rutina destino.**


El objetivo inmediato es resolver ese último enlace.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> from pathlib import Path
> import struct
>
> data = Path("/storage/emulated/0/Download/armon/ARMON.EXE").read_bytes()
>
> # Segmento 3
> seg_start = 0x04B2C0
> seg_size  = 0x6ED0
> reloc     = seg_start + seg_size
>
> count = struct.unpack_from("<H", data, reloc)[0]
>
> print("============================================================")
> print(" ANÁLISIS DE RELOCACIONES DEL SEGMENTO 3")
> print("============================================================")
> print(f"Tabla        : 0x{reloc:06X}")
> print(f"Entradas     : {count}")
> print(f"Tamaño tabla : 0x{2 + count*8:X}")
> print()
>
> # Offsets dentro del segmento donde están las palabras
 de SEGMENTO
> objetivos = {
>     "CALL 1": 0x1DCC,
>     "CALL 2": 0x1DDE,
>     "CALL 3": 0x1DF0,
> }

       >
> print("OFFSETS QUE BUSCAMOS")
> print("--------------------")
>
> for nombre, off in objetivos.items():
>     print(f"{nombre}: segmento 3 + 0x{off:04X}")
>
> print()
> print("============================================================")
> print(" REGISTROS QUE CONTIENEN ESOS OFFSETS")
> print("============================================================")
>
> base = reloc + 2
>
> encontrados = set()
>
> for i in range(count):
>     p = base + i * 8
>     raw = data[p:p+8]
>
>     # Buscar cada objetivo como WORD little-endian
>     words = [
>         struct.unpack_from("<H", raw, 0)[0],
>         struct.unpack_from("<H", raw, 2)[0],
>         struct.unpack_from("<H", raw, 4)[0],
>         struct.unpack_from("<H", raw, 6)[0],
>     ]
>
>     for nombre, objetivo in objetivos.items():
>         if objetivo in words:
>             print()
>             print(f"{nombre}")
  struct.unpack_from("<>             print(f"  registro : {i}")
>             print(f"  archivo  : 0x{p:06X}")
rom("<H>             print(f"  bytes    : {' '.join(f'{x:02X}' for x in raw)}")
>             print(f"  WORDs    : " + " ".join(f"0x{x:04X}" for x in words))
 in raw)>             print(f"  objetivo : 0x{objetivo:04X}")
>             encontrados.add(nombre)
>
> print()
> print("============================================================")
"==> print(" BÚSQUEDA DE LOS VALORES DE SEGMENTO 1E02 / 1DE
B / 1FA9")
> print("============================================================")
bre i>
n en> segmentos = {
>     "CALL 1": 0x1E02,
>     "CALL 2": 0x1DEB,
>     "CALL 3": 0x1FA9,
> }
>
> for nombre, valor in segmentos.items():
>
>     print()
>     print(f"{nombre}: 0x{valor:04X}")
>
>     encontrados_valor = 0
>
>     for i in range(count):
>         p = base + i * 8
>         raw = data[p:p+8]
>
>         if valor in [
>             struct.unpack_from("<H", raw, 0)[0],
>             struct.unpack_from("<H", raw, 2)[0],
>             struct.unpack_from("<H", raw, 4)[0],
>             struct.unpack_from("<H", raw, 6)[0],
>         ]:
>             print(
>                 f"  registro {i:3d} "
>                 f"archivo 0x{p:06X} : "
>                 + " ".join(f"{x:02X}" for x in raw)
>             )
>             encontrados_valor += 1
>
>     print(f"  coincidencias: {encontrados_valor}")
>
> print()
> print("============================================================")
> print(" RESUMEN")
> print("============================================================")
>
> for nombre in objetivos:
>     print(
>         f"{nombre}: "
>         + ("ENCONTRADO" if nombre in encontrados else "NO ENCONTRADO")
>     )
> PY
============================================================
 ANÁLISIS DE RELOCACIONES DEL SEGMENTO 3
============================================================
Tabla        : 0x052190
Entradas     : 244
Tamaño tabla : 0x7A2

OFFSETS QUE BUSCAMOS
--------------------
CALL 1: segmento 3 + 0x1DCC
CALL 2: segmento 3 + 0x1DDE
CALL 3: segmento 3 + 0x1DF0

============================================================
 REGISTROS QUE CONTIENEN ESOS OFFSETS
============================================================

============================================================
 BÚSQUEDA DE LOS VALORES DE SEGMENTO 1E02 / 1DEB / 1FA9
============================================================

CALL 1: 0x1E02
  coincidencias: 0

CALL 2: 0x1DEB
  coincidencias: 0

CALL 3: 0x1FA9
  coincidencias: 0

============================================================
 RESUMEN
============================================================
CALL 1: NO ENCONTRADO
CALL 2: NO ENCONTRADO
CALL 3: NO ENCONTRADO
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Aquí aparece la clave. **El resultado anterior no significa que las llamadas no tengan relocaciones; significa que interpretamos el orden de los campos de cada registro incorrectamente.**


Los registros que vemos tienen esta forma real:


```
[2 bytes] [1 byte] [1 byte] [2 bytes] [2 bytes]
 offset     tipo      flags    target1    target2
```


Por ejemplo, los primeros 8 bytes después del contador:


```
4D 00 00 00 02 00 78 00
```


se pueden leer como:


```
offset = 0x004D
tipo   = 0x00
flags  = 0x00
target1= 0x0002
target2= 0x0078
```


Todos esos valores tienen sentido para una relocación NE. La documentación del formato describe precisamente los campos de origen, tipo de referencia y destino; además, el offset de origen inicia una cadena de referencias dentro del segmento.


Por eso nuestro script anterior buscó `0x1DCC` en los cuatro WORD de una estructura que habíamos interpretado mal y no encontró nada.


### Ahora debemos hacer algo más preciso


No basta con buscar `0x1DCC` literalmente.


En NE, el campo `offset` es **el primer elemento de una cadena de relocaciones**. El valor almacenado en esa posición del segmento indica la siguiente posición de la cadena, terminando en `FFFF`.


Así que ahora podemos reconstruir correctamente:


```
relocación
   ↓
offset inicial
   ↓
contenido de segmento[offset]
   ↓
siguiente offset
   ↓
...
   ↓
FFFF
```


Y además distinguir:


- `tipo 02` → referencia al segmento
- `tipo 03` → puntero FAR
- `tipo 05` → offset
- `flags 00` → referencia interna
- `flags 01` → ordinal importado
- `flags 02` → nombre importado.


### Este es el siguiente comando


Vamos a **parsear los 244 registros correctamente** y buscar específicamente las cadenas que afectan a las tres llamadas de `CalcularClick`.


```
Bashcd /storage/emulated/0/Download/armon

python3 - <<'PY'
from pathlib import Path
import struct

data = Path("/storage/emulated/0/Download/armon/ARMON.EXE").read_bytes()

SEG_START = 0x04B2C0
SEG_SIZE  = 0x6ED0
RELOC     = SEG_START + SEG_SIZE

count = struct.unpack_from("<H", data, RELOC)[0]
base = RELOC + 2

print("============================================================")
print(" PARSER CORRECTO DE RELOCACIONES NE — SEGMENTO 3")
print("============================================================")
print(f"Tabla    : 0x{RELOC:06X}")
print(f"Entradas : {count}")
print()

# Formato observado:
#
# WORD offset
# BYTE source_type
# BYTE flags
# WORD target1
# WORD target2

relocs = []

for i in range(count):
    p = base + i * 8

    offset, source_type, flags, target1, target2 = struct.unpack_from(
        "<HBBHH", data, p
    )

    relocs.append({
        "index": i,
        "file": p,
        "offset": offset,
        "source_type": source_type,
        "flags": flags,
        "target1": target1,
        "target2": target2,
    })

print("PRIMERAS RELOCACIONES")
print("----------------------")

for r in relocs[:20]:
    print(
        f"{r['index']:3d}: "
        f"off=0x{r['offset']:04X} "
        f"type={r['source_type']:02X} "
        f"flags={r['flags']:02X} "
        f"target1=0x{r['target1']:04X} "
        f"target2=0x{r['target2']:04X}"
    )

print()
print("============================================================")
print(" RELOCACIONES CERCANAS A CALCULARCLICK")
print("============================================================")

objetivos = {
    "CALL 1 segmento": 0x1DCC,
    "CALL 2 segmento": 0x1DDE,
    "CALL 3 segmento": 0x1DF0,
    "CALL 1 offset":   0x1DCB,
    "CALL 2 offset":   0x1DDD,
    "CALL 3 offset":   0x1DEF,
}

for nombre, objetivo in objetivos.items():

    print()
    print(f"{nombre}: 0x{objetivo:04X}")

    hallado = False

    for r in relocs:

        # El registro puede comenzar exactamente en el punto
        # de la referencia o estar muy próximo a él.
        if r["offset"] == objetivo:
            print(
                f"  *** COINCIDE *** "
                f"registro={r['index']} "
                f"archivo=0x{r['file']:06X}"
            )
            print(
                f"      offset=0x{r['offset']:04X} "
                f"type={r['source_type']:02X} "
                f"flags={r['flags']:02X} "
                f"target1=0x{r['target1']:04X} "
                f"target2=0x{r['target2']:04X}"
            )
            hallado = True

    if not hallado:
        print("  no hay registro que comience exactamente aquí")

print()
print("============================================================")
print(" RELOCACIONES EN LA ZONA 0x1D80–0x1E20")
print("============================================================")

for r in relocs:

    if 0x1D80 <= r["offset"] <= 0x1E20:

        print(
            f"registro {r['index']:3d} | "
            f"off=0x{r['offset']:04X} | "
            f"type={r['source_type']:02X} | "
            f"flags={r['flags']:02X} | "
            f"target1=0x{r['target1']:04X} | "
            f"target2=0x{r['target2']:04X}"
        )

print()
print("============================================================")
print(" FIN")
print("============================================================")
PY
```


Este resultado es mucho más importante que el anterior.


Si aparecen entradas exactamente en `0x1DCC`, `0x1DDE` o `0x1DF0`, ya tendremos la **referencia NE formal que Windows utiliza para corregir esas tres llamadas**.


Y si no aparecen exactamente allí, no pasa nada: el siguiente paso será seguir la **cadena de relocación** desde los offsets que sí aparezcan. Ese es el mecanismo correcto del formato NE.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> from pathlib import Path
> import struct
>
> data = Path("/storage/emulated/0/Download/armon/ARMON.EXE").read_bytes()
e": source_type,
        "flags": flags,
        "target1": target1,
        "target2": target2,
    })

print("PRIMERAS RELOCACIONES")
print>
> SEG_START = 0x04B2C0
> SEG_SIZE  = 0x6ED0
rint(
        f"{r['index']:3d}: "
        f"off=0x{r['offset']:04X} "
        f"type={r['source_type']:02X} "
  > RELOC     = SEG_START + SEG_SIZE
f"target1=0x{r['target1']:04X} "
        f"target2=0x{r['target2']:04X}"
    )

print()
print("============================================================")
print(" RELOCACIONES CERCANAS A CALCULARCLICK")
print>
==============> count = struct.unpack_from("<H", data, RELOC)[0]
to": 0x1DCC,
 > base = RELOC + 2
>
x1DDE,
    "CALL 3 segmento": 0> print("============================================================")
tems():

    print()
    print(f"{> print(" PARSER CORRECTO DE RELOCACIONES NE — SEGMENTO 3")
enzar exactamente en el punto
        # de > print("============================================================")
> print(f"Tabla    : 0x{RELOC:06X}")
> print(f"Entradas : {count}")
> print()
>
> # Formato observado:
> #
> # WORD offset
> # BYTE source_type
> # BYTE flags
> # WORD target1
> # WORD target2
>
> relocs = []
>
> for i in range(count):
>     p = base + i * 8
>
>     offset, source_type, flags, target1, target2 = struct.unpack_from(
>         "<HBBHH", data, p
>     )
>
>     relocs.append({
>         "index": i,
>         "file": p,
>         "offset": offset,
>         "source_type": source_type,
>         "flags": flags,
>         "target1": target1,
>         "target2": target2,
>     })
o {r>
 | "
  > print("PRIMERAS RELOCACIONES")
ff=0x{r['o> print("----------------------")
X} | "
     >
> for r in relocs[:20]:
>     print(
>         f"{r['index']:3d}: "
>         f"off=0x{r['offset']:04X} "
>         f"type={r['source_type']:02X} "
4X}"
 >         f"flags={r['flags']:02X} "
==========>         f"target1=0x{r['target1']:04X} "
==========>         f"target2=0x{r['target2']:04X}"
==========>     )
>
> print()
==> print("============================================================")
> print(" RELOCACIONES CERCANAS A CALCULARCLICK")
> print("============================================================")
>
> objetivos = {
>     "CALL 1 segmento": 0x1DCC,
>     "CALL 2 segmento": 0x1DDE,
>     "CALL 3 segmento": 0x1DF0,
>     "CALL 1 offset":   0x1DCB,
>     "CALL 2 offset":   0x1DDD,
>     "CALL 3 offset":   0x1DEF,
> }
>
> for nombre, objetivo in objetivos.items():
>
>     print()
>     print(f"{nombre}: 0x{objetivo:04X}")
>
>     hallado = False
>
>     for r in relocs:
>
>         # El registro puede comenzar exactamente en el punto
>         # de la referencia o estar muy próximo aél.
>         if r["offset"] == objetivo:
>             print(
>                 f"  *** COINCIDE *** "
>                 f"registro={r['index']} "
>                 f"archivo=0x{r['file']:06X}"
>             )
>             print(
>                 f"      offset=0x{r['offset']:04X} "
>                 f"type={r['source_type']:02X} "
>                 f"flags={r['flags']:02X} "
>                 f"target1=0x{r['target1']:04X} "
>                 f"target2=0x{r['target2']:04X}"
>             )
>             hallado = True
>
>     if not hallado:
>         print("  no hay registro que comience exactamente aquí")
>
> print()
> print("============================================================")
> print(" RELOCACIONES EN LA ZONA 0x1D80–0x1E20")
> print("============================================================")
>
> for r in relocs:
>
>     if 0x1D80 <= r["offset"] <= 0x1E20:
>
>         print(
>             f"registro {r['index']:3d} | "
>             f"off=0x{r['offset']:04X} | "
>             f"type={r['source_type']:02X} | "
>             f"flags={r['flags']:02X} | "
>             f"target1=0x{r['target1']:04X} | "
>             f"target2=0x{r['target2']:04X}"
>         )
>
> print()
> print("============================================================")
> print(" FIN")
> print("============================================================")
> PY
============================================================
 PARSER CORRECTO DE RELOCACIONES NE — SEGMENTO 3
============================================================
Tabla    : 0x052190
Entradas : 244

PRIMERAS RELOCACIONES
----------------------
  0: off=0x0002 type=90 flags=00 target1=0x004D target2=0x0000
  1: off=0x0002 type=78 flags=00 target1=0x004C target2=0x0000
  2: off=0x0002 type=34 flags=00 target1=0x004E target2=0x0000
  3: off=0x0002 type=1C flags=00 target1=0x0050 target2=0x0000
  4: off=0x0002 type=A9 flags=00 target1=0x0003 target2=0x0000
  5: off=0x0002 type=E2 flags=0F target1=0x0048 target2=0x0000
  6: off=0x0002 type=EA flags=0F target1=0x004B target2=0x0000
  7: off=0x0002 type=EE flags=0F target1=0x0046 target2=0x0000
  8: off=0x0002 type=FE flags=0F target1=0x0049 target2=0x0000
  9: off=0x0002 type=06 flags=10 target1=0x0044 target2=0x0000
 10: off=0x0002 type=12 flags=10 target1=0x0043 target2=0x0000
 11: off=0x0002 type=90 flags=10 target1=0x003A target2=0x0000
 12: off=0x0002 type=8E flags=13 target1=0x004F target2=0x0000
 13: off=0x0002 type=7E flags=14 target1=0x0005 target2=0x0000
 14: off=0x0002 type=C3 flags=14 target1=0x003E target2=0x0000
 15: off=0x0705 type=D9 flags=14 target1=0x0005 target2=0x0000
 16: off=0x0705 type=DE flags=14 target1=0x0005 target2=0x0000
 17: off=0x0705 type=E6 flags=14 target1=0x0004 target2=0x0000
 18: off=0x0705 type=F0 flags=14 target1=0x0002 target2=0x0000
 19: off=0x0705 type=F4 flags=14 target1=0x0006 target2=0x0000

============================================================
 RELOCACIONES CERCANAS A CALCULARCLICK
============================================================

CALL 1 segmento: 0x1DCC
  no hay registro que comience exactamente aquí

CALL 2 segmento: 0x1DDE
  no hay registro que comience exactamente aquí

CALL 3 segmento: 0x1DF0
  no hay registro que comience exactamente aquí

CALL 1 offset: 0x1DCB
  no hay registro que comience exactamente aquí

CALL 2 offset: 0x1DDD
  no hay registro que comience exactamente aquí

CALL 3 offset: 0x1DEF
  no hay registro que comience exactamente aquí

============================================================
 RELOCACIONES EN LA ZONA 0x1D80–0x1E20
============================================================

============================================================
 FIN
============================================================
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Exactamente. **Ahora tenemos la estructura correcta confirmada por la documentación histórica de Microsoft**: cada relocación es:


```
BYTE  tipo
BYTE  flags
WORD  offset dentro del segmento
4 BYTES destino
```


No:


```
WORD offset
BYTE tipo
...
```


Eso explica completamente el resultado anterior.


Además, hay una pista muy importante: para un `CALL FAR` como:


```
9A 44 04 02 1E
```


la referencia es de tipo **segmento + offset**, por lo que debemos buscar una relocación cuyo `tipo` sea `03` y cuyo `offset` corresponda al inicio del operando de la llamada, es decir, `0x1DCB`. La documentación indica que `03h` representa una dirección segmento+offset.


Vamos a comprobarlo directamente y, si aparece, resolveremos el segmento y offset reales.


```
Bashcd /storage/emulated/0/Download/armon

python3 - <<'PY'
from pathlib import Path
import struct

data = Path("/storage/emulated/0/Download/armon/ARMON.EXE").read_bytes()

SEG_START = 0x04B2C0
SEG_SIZE  = 0x6ED0
RELOC     = SEG_START + SEG_SIZE

count = struct.unpack_from("<H", data, RELOC)[0]
base = RELOC + 2

print("============================================================")
print(" RELOCACIONES NE — FORMATO CORRECTO")
print("============================================================")
print(f"Tabla    : 0x{RELOC:06X}")
print(f"Entradas : {count}")
print()

relocs = []

for i in range(count):
    p = base + i * 8

    source_type = data[p]
    flags       = data[p + 1]
    source_off  = struct.unpack_from("<H", data, p + 2)[0]

    target = data[p + 4:p + 8]

    relocs.append({
        "index": i,
        "file": p,
        "type": source_type,
        "flags": flags,
        "source": source_off,
        "target": target,
    })

print("PRIMERAS 20 RELOCACIONES")
print("------------------------")

for r in relocs[:20]:
    print(
        f"{r['index']:3d}: "
        f"type={r['type']:02X} "
        f"flags={r['flags']:02X} "
        f"source=0x{r['source']:04X} "
        f"target="
        f"{r['target'][0]:02X} "
        f"{r['target'][1]:02X} "
        f"{r['target'][2]:02X} "
        f"{r['target'][3]:02X}"
    )

print()
print("============================================================")
print(" BUSCANDO LAS TRES LLAMADAS FAR")
print("============================================================")

# CALL FAR:
#
# 9A oo oo ss ss
#
# El operando empieza en:
#
# CALL 1 -> 0x1DCB
# CALL 2 -> 0x1DDD
# CALL 3 -> 0x1DEF
#
# Una referencia FAR completa usa source type 03.

calls = {
    "CALL 1": 0x1DCB,
    "CALL 2": 0x1DDD,
    "CALL 3": 0x1DEF,
}

for nombre, objetivo in calls.items():

    print()
    print(f"{nombre}  source = 0x{objetivo:04X}")

    encontrados = []

    for r in relocs:
        if r["source"] == objetivo:
            encontrados.append(r)

    if not encontrados:
        print("  NO ENCONTRADO")
        continue

    for r in encontrados:

        print(
            f"  registro {r['index']}"
        )

        print(
            f"  archivo : 0x{r['file']:06X}"
        )

        print(
            f"  type    : 0x{r['type']:02X}"
        )

        print(
            f"  flags   : 0x{r['flags']:02X}"
        )

        print(
            f"  source  : 0x{r['source']:04X}"
        )

        print(
            "  target  : "
            + " ".join(f"{x:02X}" for x in r["target"])
        )

        if r["type"] == 0x03:
            print("  *** ES REFERENCIA FAR ***")

            if r["flags"] & 0x03 == 0x00:

                seg = r["target"][0]
                off = struct.unpack_from("<H", r["target"], 2)[0]

                print(
                    f"  DESTINO INTERNO: segmento={seg} "
                    f"offset=0x{off:04X}"
                )

        elif r["type"] == 0x02:

            seg = r["target"][0]

            print(
                f"  REFERENCIA A SEGMENTO: {seg}"
            )

print()
print("============================================================")
print(" TODAS LAS RELOCACIONES FAR INTERNAS")
print("============================================================")

for r in relocs:

    if r["type"] == 0x03 and (r["flags"] & 0x03) == 0:

        seg = r["target"][0]
        off = struct.unpack_from("<H", r["target"], 2)[0]

        print(
            f"source=0x{r['source']:04X} "
            f"-> segmento={seg:02X} "
            f"offset=0x{off:04X}"
        )

print()
print("============================================================")
print(" FIN")
print("============================================================")
PY
```


### Qué espero que encontremos


Para cada llamada podría aparecer algo conceptualmente así:


```
CALL 1 source = 0x1DCB
  *** ES REFERENCIA FAR ***
  DESTINO INTERNO: segmento=XX offset=0xYYYY
```


Entonces finalmente tendremos:


```
CalcularClick
    ↓
CALL FAR
    ↓
segmento XX : YYYY
    ↓
bytes reales de la rutina
    ↓
desensamblado
```


**Este es el paso decisivo.** A partir de aquí ya no estaremos intentando adivinar qué significan `1E02`, `1DEB` y `1FA9`: el propio formato NE nos dará el destino de cada llamada.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> from pathlib import Path
> import struct
>
> data = Path("/storage/emulated/0/Download/armon/ARMON.EXE").read_bytes()
>
> SEG_START = 0x04B2C0
> SEG_SIZE  = 0x6ED0
> RELOC     = SEG_START + SEG_SIZE
>
> count = struct.unpack_from("<H", data, RELOC)[0]
> base = RELOC + 2
>
> print("============================================================")
> print(" RELOCACIONES NE — FORMATO CORRECTO")
> print("============================================================")
> print(f"Tabla    : 0x{RELOC:06X}")
> print(f"Entradas : {count}")
> print()
>
> relocs = []
>
> for i in range(count):
>     p = base + i * 8
>
>     source_type = data[p]
>     flags       = data[p + 1]
>     source_off  = struct.unpack_from("<H", data, p + 2)[0]
>
>     target = data[p + 4:p + 8]
>
>     relocs.append({
>         "index": i,
>         "file": p,
>         "type": source_type,
>         "flags": flags,
>         "source": source_off,
>         "target": target,
>     })
>
> print("PRIMERAS 20 RELOCACIONES")
> print("------------------------")
>
> for r in relocs[:20]:
>     print(
>         f"{r['index']:3d}: "
>         f"type={r['type']:02X} "
>         f"flags={r['flags']:02X} "
>         f"source=0x{r['source']:04X} "
>         f"target="
>         f"{r['target'][0]:02X} "
>         f"{r['target'][1]:02X} "
>         f"{r['target'][2]:02X} "
>         f"{r['target'][3]:02X}"
>     )
>
> print()
> print("============================================================")
> print(" BUSCANDO LAS TRES LLAMADAS FAR")
> print("============================================================")
>
> # CALL FAR:
> #
> # 9A oo oo ss ss
> #
m(> # El operando empieza en:
> #
, > # CALL 1 -> 0x1DCB
> # CALL 2 -> 0x1DDD
> # CALL 3 -> 0x1DEF
> #
> # Una referencia FAR completa usa source type 03.
>
> calls = {
>     "CALL 1": 0x1DCB,
>     "CALL 2": 0x1DDD,
>     "CALL 3": 0x1DEF,
> }
>
> for nombre, objetivo in calls.items():
>
>     print()
>     print(f"{nombre}  source = 0x{objetivo:04X}")
>
>     encontrados = []
>
>     for r in relocs:
>         if r["source"] == objetivo:
>             encontrados.append(r)
>
>     if not encontrados:
>         print("  NO ENCONTRADO")
>         continue
>
>     for r in encontrados:
>
[0]

                print(
                    f"  DESTINO INTERNO: segmento={seg} "
                    f"offset=0x{off:04X}"
                )

        elif r["type"] == 0x02:

            seg = r["target"][0]

            print(
                f"  REFERENCIA A SEGMENTO: {seg}"
            )

print()
print("============================================================")
print(" TODAS LAS RELOCACIONES FAR INTERNAS")
print("============================================================")

for r in relocs:

    if r["type"] == 0x03 and (r["flags"] & 0x03) == 0:

        seg = r["target"][0]
        off = struct.unpack_from("<H", r["target"], 2)[0]

        print(
            f"source=0x{r['source']:04X} "
            f"-> segmento={seg:02X} "
            f"offset=0x{off:04X}"
        )

print()
print("============================================================")
print(" FIN")
print("============================================================")
PY>         print(
>             f"  registro {r['index']}"
>         )
>
>         print(
>             f"  archivo : 0x{r['file']:06X}"
>         )
>
>         print(
>             f"  type    : 0x{r['type']:02X}"
>         )
>
>         print(
>             f"  flags   : 0x{r['flags']:02X}"
>         )
>
>         print(
>             f"  source  : 0x{r['source']:04X}"
>         )
>
>         print(
>             "  target  : "
>             + " ".join(f"{x:02X}" for x in r["target"])
>         )
>
>         if r["type"] == 0x03:
>             print("  *** ES REFERENCIA FAR ***")
>
>             if r["flags"] & 0x03 == 0x00:
>
>                 seg = r["target"][0]
>                 off = struct.unpack_from("<H", r["target"], 2)[0]
>
>                 print(
>                     f"  DESTINO INTERNO: segmento={seg} "
>                     f"offset=0x{off:04X}"
>                 )
>
>         elif r["type"] == 0x02:
>
>             seg = r["target"][0]
>
>             print(
>                 f"  REFERENCIA A SEGMENTO: {seg}"
>             )
>
> print()
> print("============================================================")
> print(" TODAS LAS RELOCACIONES FAR INTERNAS")
> print("============================================================")
>
> for r in relocs:
>                                                      :
>     if r["type"] == 0x03 and (r["flags"] & 0x03) == 0:
>         seg = r["target"][0]
>         off = struct.unpack_from("<H", r["target"], 2)[0]
>
>         print(
>             f"source=0x{r['source']:04X} "
>             f"-> segmento={seg:02X} "
>             f"offset=0x{off:04X}"
>         )
>
> print()
> print("============================================================")
> print(" FIN")
> print("============================================================")
> PY
============================================================
 RELOCACIONES NE — FORMATO CORRECTO
============================================================
Tabla    : 0x052190
Entradas : 244

PRIMERAS 20 RELOCACIONES
------------------------
  0: type=02 flags=00 source=0x0090 target=4D 00 00 00
  1: type=02 flags=00 source=0x0078 target=4C 00 00 00
  2: type=02 flags=00 source=0x0034 target=4E 00 00 00
  3: type=02 flags=00 source=0x001C target=50 00 00 00
  4: type=02 flags=00 source=0x00A9 target=03 00 00 00
  5: type=02 flags=00 source=0x0FE2 target=48 00 00 00
  6: type=02 flags=00 source=0x0FEA target=4B 00 00 00
  7: type=02 flags=00 source=0x0FEE target=46 00 00 00
  8: type=02 flags=00 source=0x0FFE target=49 00 00 00
  9: type=02 flags=00 source=0x1006 target=44 00 00 00
 10: type=02 flags=00 source=0x1012 target=43 00 00 00
 11: type=02 flags=00 source=0x1090 target=3A 00 00 00
 12: type=02 flags=00 source=0x138E target=4F 00 00 00
 13: type=02 flags=00 source=0x147E target=05 00 00 00
 14: type=02 flags=00 source=0x14C3 target=3E 00 00 00
 15: type=05 flags=07 source=0x14D9 target=05 00 00 00
 16: type=05 flags=07 source=0x14DE target=05 00 00 00
 17: type=05 flags=07 source=0x14E6 target=04 00 00 00
 18: type=05 flags=07 source=0x14F0 target=02 00 00 00
 19: type=05 flags=07 source=0x14F4 target=06 00 00 00

============================================================
 BUSCANDO LAS TRES LLAMADAS FAR
============================================================

CALL 1  source = 0x1DCB
  NO ENCONTRADO

CALL 2  source = 0x1DDD
  NO ENCONTRADO

CALL 3  source = 0x1DEF
  NO ENCONTRADO

============================================================
 TODAS LAS RELOCACIONES FAR INTERNAS
============================================================

============================================================
 FIN
============================================================
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Esto confirma la hipótesis importante: **el formato de cada registro ya está correctamente interpretado**, pero todavía estamos buscando la referencia en el campo equivocado.


La clave es esta:


> `source=0x0090` no significa necesariamente que la relocación corresponda solamente a `0x0090`. Es el **inicio de una cadena de referencias**.


En NE, el valor almacenado en cada posición de la cadena apunta a la siguiente posición de la misma cadena. Por eso nuestro `0x1DCC` puede **no aparecer como `source` de ningún registro**, pero sí estar dentro de una cadena iniciada, por ejemplo, en `0x0090`.


Y esto encaja perfectamente con lo que vimos en `CalcularClick`:


```
CALL FAR
9A 44 04 02 1E
         └─────┘
         0x1E02
```


Ese `0x1E02` puede ser precisamente el **siguiente enlace de la cadena de relocación**, no el segmento definitivo.


Vamos a seguir las cadenas automáticamente. Esta vez buscaremos si `0x1DCB`, `0x1DCC`, etc. aparecen **en cualquier punto de una cadena**, y entonces obtendremos el destino real de la relocación.


```
Bashcd /storage/emulated/0/Download/armon

python3 - <<'PY'
from pathlib import Path
import struct

data = Path("/storage/emulated/0/Download/armon/ARMON.EXE").read_bytes()

SEG_START = 0x04B2C0
SEG_SIZE  = 0x6ED0
RELOC     = SEG_START + SEG_SIZE

count = struct.unpack_from("<H", data, RELOC)[0]
base = RELOC + 2

print("============================================================")
print(" NE — SEGUIMIENTO DE CADENAS DE RELOCACION")
print("============================================================")
print(f"Segmento 3 : archivo 0x{SEG_START:06X}")
print(f"Tamaño     : 0x{SEG_SIZE:04X}")
print(f"Tabla      : 0x{RELOC:06X}")
print(f"Entradas   : {count}")
print()

# ------------------------------------------------------------
# Leer relocaciones
# ------------------------------------------------------------

relocs = []

for i in range(count):

    p = base + i * 8

    source_type = data[p]
    flags       = data[p + 1]
    source      = struct.unpack_from("<H", data, p + 2)[0]

    # Los cuatro bytes del destino.
    target = data[p + 4:p + 8]

    relocs.append({
        "index": i,
        "type": source_type,
        "flags": flags,
        "source": source,
        "target": target,
    })


# ------------------------------------------------------------
# Leer WORD dentro del segmento
# ------------------------------------------------------------

def word(off):

    if off < 0 or off + 2 > SEG_SIZE:
        return None

    return struct.unpack_from(
        "<H",
        data,
        SEG_START + off
    )[0]


# ------------------------------------------------------------
# Interpretar destino
# ------------------------------------------------------------

def destino(r):

    typ = r["type"]
    flags = r["flags"]
    t = r["target"]

    target_kind = flags & 0x03

    if target_kind == 0:

        # Referencia interna.
        #
        # Para referencias de segmento:
        # target[0] = número de segmento
        #
        # Para offset/far:
        # los restantes bytes contienen información adicional.

        if typ == 0x02:
            return f"SEGMENTO {t[0]}"

        elif typ == 0x05:
            off = struct.unpack_from("<H", t, 0)[0]
            return f"OFFSET 0x{off:04X}"

        elif typ == 0x03:
            off = struct.unpack_from("<H", t, 0)[0]
            seg = t[2]
            return f"FAR {seg:02X}:0x{off:04X}"

        return "INTERNO"

    if target_kind == 1:
        return "IMPORTADO POR ORDINAL"

    if target_kind == 2:
        return "IMPORTADO POR NOMBRE"

    if target_kind == 3:
        return "OSFIXUP"

    return "DESCONOCIDO"


# ------------------------------------------------------------
# Seguir una cadena
# ------------------------------------------------------------

def seguir_cadena(inicio):

    cadena = []
    vistos = set()

    actual = inicio

    while True:

        if actual in vistos:
            cadena.append(("BUCLE", actual))
            break

        vistos.add(actual)

        if actual == 0xFFFF:
            break

        if actual >= SEG_SIZE - 1:
            cadena.append(("FUERA", actual))
            break

        siguiente = word(actual)

        cadena.append((actual, siguiente))

        if siguiente == 0xFFFF:
            break

        actual = siguiente

    return cadena


# ------------------------------------------------------------
# Objetivos de las llamadas de CalcularClick
# ------------------------------------------------------------

objetivos = {
    "CALL 1 OFFSET":   0x1DCB,
    "CALL 1 SEGMENTO": 0x1DCC,

    "CALL 2 OFFSET":   0x1DDD,
    "CALL 2 SEGMENTO": 0x1DDE,

    "CALL 3 OFFSET":   0x1DEF,
    "CALL 3 SEGMENTO": 0x1DF0,
}


print("============================================================")
print(" BUSCANDO LOS OBJETIVOS DENTRO DE LAS CADENAS")
print("============================================================")

encontrados = {}

for nombre, objetivo in objetivos.items():

    print()
    print(f"{nombre}: 0x{objetivo:04X}")

    hallado = False

    for r in relocs:

        # Las cadenas solo tienen sentido para las relocaciones
        # que usan source chains.
        if r["type"] not in (0x02, 0x03, 0x05):
            continue

        cadena = seguir_cadena(r["source"])

        posiciones = [
            x[0]
            for x in cadena
            if isinstance(x[0], int)
        ]

        if objetivo in posiciones:

            hallado = True

            print(
                f"  *** ENCONTRADO *** "
                f"registro={r['index']}"
            )

            print(
                f"  tipo       = 0x{r['type']:02X}"
            )

            print(
                f"  flags      = 0x{r['flags']:02X}"
            )

            print(
                f"  inicio     = 0x{r['source']:04X}"
            )

            print(
                f"  destino    = {destino(r)}"
            )

            print(
                "  cadena     = "
                + " -> ".join(
                    f"0x{x[0]:04X}"
                    for x in cadena
                    if isinstance(x[0], int)
                )
            )

            encontrados[nombre] = r

    if not hallado:
        print("  NO APARECE EN NINGUNA CADENA")


# ------------------------------------------------------------
# Resumen
# ------------------------------------------------------------

print()
print("============================================================")
print(" RESUMEN")
print("============================================================")

for nombre in objetivos:

    r = encontrados.get(nombre)

    if r is None:
        print(f"{nombre:18s} -> NO ENCONTRADO")
    else:
        print(
            f"{nombre:18s} -> "
            f"reloc {r['index']:3d}, "
            f"tipo={r['type']:02X}, "
            f"inicio=0x{r['source']:04X}, "
            f"{destino(r)}"
        )

print()
print("============================================================")
print(" FIN")
print("============================================================")
PY
```


### Qué estamos intentando demostrar


Tenemos ahora tres niveles distintos:


```
CalcularClick
     │
     ├── CALL FAR 1
     │       │
     │       └── relocación NE
     │              │
     │              └── cadena
     │                     │
     │                     └── destino real
     │
     ├── CALL FAR 2
     │       │
     │       └── relocación NE
     │
     └── CALL FAR 3
             │
             └── relocación NE
```


Si el script encuentra, por ejemplo:


```
CALL 1 SEGMENTO: 0x1DCC
  *** ENCONTRADO ***
  inicio = 0x....
  destino = SEGMENTO XX
```


entonces habremos recuperado **el número de segmento real** de la llamada.


Y si encontramos simultáneamente la relocación del `offset`, podremos obtener:


```
CALL 1 → segmento XX : offset YYYY
```


que es justamente la dirección lógica NE de la rutina llamada.


**Este procedimiento es mucho más fiable que buscar directamente `0x1DCC` en la tabla**, porque ahora estamos respetando la mecánica real de las cadenas de relocación NE.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> from pathlib import Path
> import struct
>
> data = Path("/storage/emulated/0/Download/armon/ARMON.EXE").read_bytes()
>
> SEG_START = 0x04B2C0
> SEG_SIZE  = 0x6ED0
> RELOC     = SEG_START + SEG_SIZE
>
> count = struct.unpack_from("<H", data, RELOC)[0]
> base = RELOC + 2
>
> print("============================================================")
> print(" NE — SEGUIMIENTO DE CADENAS DE RELOCACION")
> print("============================================================")
> print(f"Segmento 3 : archivo 0x{SEG_START:06X}")
+ 2)[0]

    # Los cuatro bytes del destino.
    target = data[p + 4:p + 8> print(f"Tamaño     : 0x{SEG_SIZE:04X}")
> print(f"Tabla      : 0x{RELOC:06X}")
> print(f"Entradas   : {count}")
--> print()
>
 Le> # ------------------------------------------------------------
----> # Leer relocaciones
ne> # ------------------------------------------------------------
-------

d>
> relocs = []
>
lags > for i in range(count):
>
>     p = base + i * 8
       # Para>
>     source_type = data[p]
>     flags       = data[p + 1]
>     source      = struct.unpack_from("<H", data, p + 2)[0]
far:
 >
>     # Los cuatro bytes del destino.
>     target = data[p + 4:p + 8]
>
>     relocs.append({
       off = struc>         "index": i,
>         "type": source_type,
>         "flags": flags,
>         "source": source,
>         "target": target,
>     })
>
>
> # ------------------------------------------------------------
> # Leer WORD dentro del segmento
> # ------------------------------------------------------------
>
ORTA> def word(off):
>
>     if off < 0 or off + 2 > SEG_SIZE:
  if ta>         return None
>
>     return struct.unpack_from(
>         "<H",
>         data,
>         SEG_START + off
>     )[0]
>
>
> # ------------------------------------------------------------
> # Interpretar destino
set()

    actual > # ------------------------------------------------------------
>
ual> def destino(r):
>
ak

        vistos.add(actua>     typ = r["type"]
>     flags = r["flags"]
>     t = r["target"]
>
= SEG_SIZE - 1:
   >     target_kind = flags & 0x03
>
ual))
          >     if target_kind == 0:
>
>         # Referencia interna.


        >         #
>         # Para referencias de segmento:
  if sig>         # target[0] = número de segmento
>         #
>         # Para offset/far:
>         # los restantes bytes contienen información ad
icional.
>
>         if typ == 0x02:
--
# O>             return f"SEGMENTO {t[0]}"
>
>         elif typ == 0x05:
---->             off = struct.unpack_from("<H", t, 0)[0]
>             return f"OFFSET 0x{off:04X}"
 ">
FSET":  >         elif typ == 0x03:
LL 2 >             off = struct.unpack_from("<H", t, 0)[0]
>             seg = t[2]
>             return f"FAR {seg:02X}:0x{off:04X}"
VOS DE>
>         return "INTERNO"
=======>
>     if target_kind == 1:
ontrados >         return "IMPORTADO POR ORDINAL"
>
t()
    prin>     if target_kind == 2:
ado = False
>         return "IMPORTADO POR NOMBRE"
>
>     if target_kind == 3:
>         return "OSFIXUP"
ido para las relocaciones
        # que usan source chains.
        if r["type"] not in (0x02, 0x03, 0x05):
            continue

        cadena = seguir_cadena(r["source"])

        posiciones = [
            x[0]
            for x in cadena
      >
>     return "DESCONOCIDO"
>
>
> # ------------------------------------------------------------
> # Seguir una cadena
(
   > # ------------------------------------------------------------
    pr>
> def seguir_cadena(inicio):
>
02X}"
         >     cadena = []
x{r['sou>     vistos = set()
>
>     actual = inicio
>
>     while True:
>
nt(
        >         if actual in vistos:
>             cadena.append(("BUCLE", actual))
>             break
>
>         vistos.add(actual)
>
 int)
               >         if actual == 0xFFFF:
>             break
ado:
>
rint("  >         if actual >= SEG_SIZE - 1:
>             cadena.append(("FUERA", actual))
----->             break
---------->
--------->         siguiente = word(actual)
>
>         cadena.append((actual, siguiente))
>
>         if siguiente == 0xFFFF:
>             break
encontrados.>
>         actual = siguiente
>
>     return cadena
RADO>

    else:
      >
> # ------------------------------------------------------------
 f"tipo={r> # Objetivos de las llamadas de CalcularClick
  )

print> # ------------------------------------------------------------
>
=================> objetivos = {
>     "CALL 1 OFFSET":   0x1DCB,
>     "CALL 1 SEGMENTO": 0x1DCC,
>
>     "CALL 2 OFFSET":   0x1DDD,
>     "CALL 2 SEGMENTO": 0x1DDE,
>
>     "CALL 3 OFFSET":   0x1DEF,
>     "CALL 3 SEGMENTO": 0x1DF0,
> }
>
>
> print("==============================================)
> print("============================================================")
>
> encontrados = {}
>
> for nombre, objetivo in objetivos.items():
>
>     print()
>     print(f"{nombre}: 0x{objetivo:04X}")
>
>     hallado = False
>
>     for r in relocs:
>
>         # Las cadenas solo tienen sentido para las relocaciones
>         # que usan source chains.
>         if r["type"] not in (0x02, 0x03, 0x05):
>             continue
>
>         cadena = seguir_cadena(r["source"])
>
>         posiciones = [
>             x[0]
>             for x in cadena
>             if isinstance(x[0], int)
>         ]
>
>         if objetivo in posiciones:
>
>             hallado = True
>
>             print(
>                 f"  *** ENCONTRADO *** "
>                 f"registro={r['index']}"
>             )
>
>             print(
>                 f"  tipo       = 0x{r['type']:02X}"
>             )
>
>             print(
>                 f"  flags      = 0x{r['flags']:02X}"
>             )
>
>             print(
>                 f"  inicio     = 0x{r['source']:04X}"
>             )
>
>             print(
>                 f"  destino    = {destino(r)}"
>             )
>
>             print(
>                 "  cadena     = "
>                 + " -> ".join(
>                     f"0x{x[0]:04X}"
>                     for x in cadena
>                     if isinstance(x[0], int)
>                 )
>             )
>
>             encontrados[nombre] = r
>
>     if not hallado:
>         print("  NO APARECE EN NINGUNA CADENA")
>
>
> # ------------------------------------------------------------
> # Resumen
> # ------------------------------------------------------------
>
> print()
> print("============================================================")
> print(" RESUMEN")
> print("============================================================")
>
> for nombre in objetivos:
>
>     r = encontrados.get(nombre)
>
>     if r is None:
>         print(f"{nombre:18s} -> NO ENCONTRADO")
>     else:
>         print(
>             f"{nombre:18s} -> "
>             f"reloc {r['index']:3d}, "
>             f"tipo={r['type']:02X}, "
>             f"inicio=0x{r['source']:04X}, "
>             f"{destino(r)}"
>         )
>
> print()
> print("============================================================")
> print(" FIN")
> print("============================================================")
> PY
============================================================
 NE — SEGUIMIENTO DE CADENAS DE RELOCACION
============================================================
Segmento 3 : archivo 0x04B2C0
Tamaño     : 0x6ED0
Tabla      : 0x052190
Entradas   : 244

============================================================
 BUSCANDO LOS OBJETIVOS DENTRO DE LAS CADENAS
============================================================

CALL 1 OFFSET: 0x1DCB
  NO APARECE EN NINGUNA CADENA

CALL 1 SEGMENTO: 0x1DCC
  *** ENCONTRADO *** registro=3
  tipo       = 0x02
  flags      = 0x00
  inicio     = 0x001C
  destino    = SEGMENTO 80
  cadena     = 0x001C -> 0x0018 -> 0x1047 -> 0x106E -> 0x1084 -> 0x10B6 -> 0x10DC -> 0x1102 -> 0x1128 -> 0x1147 -> 0x115D -> 0x1183 -> 0x1198 -> 0x11BE -> 0x11D3 -> 0x11F9 -> 0x120E -> 0x122A -> 0x1253 -> 0x126C -> 0x1307 -> 0x132D -> 0x1353 -> 0x137C -> 0x1393 -> 0x13CD -> 0x13F6 -> 0x1415 -> 0x1452 -> 0x1468 -> 0x1556 -> 0x1560 -> 0x157A -> 0x169D -> 0x1719 -> 0x1729 -> 0x17BB -> 0x17D5 -> 0x17F7 -> 0x1833 -> 0x1860 -> 0x188B -> 0x18BA -> 0x18FC -> 0x1906 -> 0x191C -> 0x195E -> 0x19CC -> 0x19FE -> 0x1A0F -> 0x1A45 -> 0x1A59 -> 0x1A7C -> 0x1B4F -> 0x1B71 -> 0x1BF3 -> 0x1C0E -> 0x1C6E -> 0x1D09 -> 0x1D3C -> 0x1DCC -> 0x1E02 -> 0x1E22 -> 0x1ECB -> 0x1F22 -> 0x1F31 -> 0x1FC8 -> 0x2303 -> 0x236B -> 0x237D -> 0x23C5 -> 0x245E -> 0x248B -> 0x24A2 -> 0x24B9 -> 0x24D0 -> 0x24E3 -> 0x24F1 -> 0x2500 -> 0x250F -> 0x251D -> 0x252C -> 0x2545 -> 0x2557 -> 0x25E2 -> 0x2719 -> 0x273B -> 0x274E -> 0x2760 -> 0x285C -> 0x2895 -> 0x28D7 -> 0x2950 -> 0x29D2 -> 0x2B3A -> 0x2B4F -> 0x2BBC -> 0x2BD1 -> 0x2C66 -> 0x2C87 -> 0x2CA8 -> 0x2CCC -> 0x2CF0 -> 0x2D14 -> 0x2D38 -> 0x2D71 -> 0x2DA7 -> 0x2DC8 -> 0x2DE9 -> 0x2E0A -> 0x2E2B -> 0x2E4C -> 0x2E6D -> 0x2E8E -> 0x2EC0 -> 0x2F19 -> 0x2F60 -> 0x2F65 -> 0x2F76 -> 0x2FA3 -> 0x2FB7 -> 0x2FDF -> 0x2FF8 -> 0x3008 -> 0x3022 -> 0x302A -> 0x3045 -> 0x3079 -> 0x30AF -> 0x30E6 -> 0x3108 -> 0x3116 -> 0x3136 -> 0x3141 -> 0x314C -> 0x3157 -> 0x3162 -> 0x316D -> 0x317D -> 0x318D -> 0x319C -> 0x31AC -> 0x31BB -> 0x31CA -> 0x31F2 -> 0x323E -> 0x3278 -> 0x33C1 -> 0x3437 -> 0x343E -> 0x3474 -> 0x34E2 -> 0x353C -> 0x358F -> 0x35C8 -> 0x35DF -> 0x35F9 -> 0x3601 -> 0x3613 -> 0x362D -> 0x3635 -> 0x3648 -> 0x3652 -> 0x3660 -> 0x3670 -> 0x368D -> 0x3698 -> 0x36C5 -> 0x36D5 -> 0x36F2 -> 0x36FD -> 0x372A -> 0x373A -> 0x3757 -> 0x3762 -> 0x381D -> 0x384D -> 0x387D -> 0x38AD -> 0x38D8 -> 0x38FF -> 0x3929 -> 0x398D -> 0x39DC -> 0x3A5E -> 0x3AC3 -> 0x3ADF -> 0x3B8D -> 0x3BB5 -> 0x3C2A -> 0x3C39 -> 0x3C6F -> 0x3CB1 -> 0x3D01 -> 0x3D32 -> 0x3E36 -> 0x3E60 -> 0x3EA7 -> 0x3ED1 -> 0x3F30 -> 0x3F5A -> 0x3F78 -> 0x3F9B -> 0x3FE1 -> 0x3FFA -> 0x4014 -> 0x4095 -> 0x40F1 -> 0x413D -> 0x414D -> 0x417A -> 0x41B2 -> 0x41E4 -> 0x4222 -> 0x427A -> 0x4324 -> 0x4356 -> 0x4394 -> 0x43EC -> 0x4477 -> 0x449B -> 0x44B7 -> 0x44DB -> 0x44F0 -> 0x453D -> 0x4559 -> 0x4589 -> 0x4602 -> 0x4635 -> 0x469B -> 0x46A0 -> 0x46B4 -> 0x46ED -> 0x4717 -> 0x472C -> 0x4789 -> 0x47E6 -> 0x4859 -> 0x48B5 -> 0x491C -> 0x496C -> 0x49AC -> 0x49D6 -> 0x4A19 -> 0x4A35 -> 0x4AA1 -> 0x4ACB -> 0x4ADF -> 0x4AF9 -> 0x4B3C -> 0x4B85 -> 0x4B8F -> 0x4BC3 -> 0x4BD2 -> 0x4BDC -> 0x4C16 -> 0x4C31 -> 0x4C5B -> 0x4C96 -> 0x4CB8 -> 0x4CDB -> 0x4CEB -> 0x4D19 -> 0x4D2C -> 0x4D53 -> 0x4D8D -> 0x4DDA -> 0x4DEA -> 0x4E0C -> 0x4E13 -> 0x4E21 -> 0x4E31 -> 0x4E4F -> 0x4E5F -> 0x4E81 -> 0x4E88 -> 0x4E96 -> 0x4EA6 -> 0x4EB9 -> 0x4EC9 -> 0x4EE3 -> 0x4EEB -> 0x4EFD -> 0x4F28 -> 0x4F38 -> 0x4F48 -> 0x4F5B -> 0x4F73 -> 0x4F7D -> 0x4F8D -> 0x4FF8 -> 0x5039 -> 0x5049 -> 0x5067 -> 0x5085 -> 0x508F -> 0x509B -> 0x50F0 -> 0x513D -> 0x514D -> 0x5169 -> 0x5179 -> 0x5191 -> 0x51A9 -> 0x51B9 -> 0x526B -> 0x52D8 -> 0x52E8 -> 0x534A -> 0x5355 -> 0x5360 -> 0x536B -> 0x5376 -> 0x538A -> 0x539A -> 0x53AA -> 0x53CC -> 0x5415 -> 0x5426 -> 0x5431 -> 0x5436 -> 0x548E -> 0x5499 -> 0x54DF -> 0x54EC -> 0x54F1 -> 0x54F6 -> 0x553E -> 0x5543 -> 0x5548 -> 0x5576 -> 0x557B -> 0x5586 -> 0x563F -> 0x56A9 -> 0x56E6 -> 0x5715 -> 0x5745 -> 0x57E0 -> 0x57FE -> 0x5812 -> 0x5847 -> 0x5852 -> 0x5874 -> 0x5896 -> 0x58A1 -> 0x58BE -> 0x58F2 -> 0x5924 -> 0x5953 -> 0x5983 -> 0x59B9 -> 0x5A00 -> 0x5A38 -> 0x5A5F -> 0x5B39 -> 0x5B6B -> 0x5B99 -> 0x5BC3 -> 0x5C1D -> 0x5C3F -> 0x5CA4 -> 0x5CD3 -> 0x5CED -> 0x5D15 -> 0x5D92 -> 0x5DBC -> 0x5DFD -> 0x5E0C -> 0x5E1F -> 0x5E30 -> 0x5E43 -> 0x5E54 -> 0x5E7D -> 0x5EBA -> 0x5ED3 -> 0x5F2B -> 0x5F55 -> 0x5F9F -> 0x5FC9 -> 0x5FD8 -> 0x6002 -> 0x600E -> 0x6025 -> 0x6031 -> 0x6044 -> 0x605F -> 0x6092 -> 0x60FC -> 0x610F -> 0x611B -> 0x6120 -> 0x6135 -> 0x614A -> 0x615D -> 0x6169 -> 0x616E -> 0x61C2 -> 0x61F6 -> 0x620B -> 0x6215 -> 0x621A -> 0x623E -> 0x6289 -> 0x629E -> 0x62E9 -> 0x6313 -> 0x6321 -> 0x635D -> 0x639B -> 0x63EC -> 0x6419 -> 0x6427 -> 0x64A2 -> 0x651D -> 0x65DB -> 0x65EE -> 0x6621 -> 0x6638 -> 0x664E -> 0x6660 -> 0x667C -> 0x66A3 -> 0x66F9 -> 0x6751 -> 0x67D5 -> 0x6859 -> 0x68DD -> 0x6954 -> 0x69A4 -> 0x6A38 -> 0x6A79 -> 0x6B12 -> 0x6B31 -> 0x6B93 -> 0x6C0A -> 0x6C71 -> 0x6CDF -> 0x6CFF -> 0x6D14 -> 0x6D1E -> 0x6D4C -> 0x6D63 -> 0x6D83 -> 0x6D98 -> 0x6DA2 -> 0x6DD2 -> 0x6DE1 -> 0x6E2C -> 0x6E9D -> 0x6ECC

CALL 2 OFFSET: 0x1DDD
  NO APARECE EN NINGUNA CADENA

CALL 2 SEGMENTO: 0x1DDE
  NO APARECE EN NINGUNA CADENA

CALL 3 OFFSET: 0x1DEF
  NO APARECE EN NINGUNA CADENA

CALL 3 SEGMENTO: 0x1DF0
  NO APARECE EN NINGUNA CADENA

============================================================
 RESUMEN
============================================================
CALL 1 OFFSET      -> NO ENCONTRADO
CALL 1 SEGMENTO    -> reloc   3, tipo=02, inicio=0x001C, SEGMENTO 80
CALL 2 OFFSET      -> NO ENCONTRADO
CALL 2 SEGMENTO    -> NO ENCONTRADO
CALL 3 OFFSET      -> NO ENCONTRADO
CALL 3 SEGMENTO    -> NO ENCONTRADO

============================================================
 FIN
============================================================
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Hay una corrección importante que ya podemos hacer con seguridad: **`CALL 1 SEGMENTO` está resuelto como segmento 80**, porque la cadena de la relocación tipo `02` pasa por `0x1DCC`. Eso está respaldado por la semántica oficial de las cadenas NE.


El `offset` es distinto porque en NE existe específicamente el tipo `05` (`OFFSET`), y además esta versión utiliza relocaciones **aditivas** (`flags=07`). En ese caso no debemos seguir la cadena exactamente como hicimos con el tipo `02`; el comportamiento de la relocación es diferente.


Vamos a aislar ahora **todos los registros tipo `05` alrededor de las tres llamadas**, además de todos los tipo `03`. Así podremos determinar exactamente cómo se construye cada dirección FAR.


```
Bashcd /storage/emulated/0/Download/armon

python3 - <<'PY'
from pathlib import Path
import struct

data = Path("/storage/emulated/0/Download/armon/ARMON.EXE").read_bytes()

SEG_START = 0x04B2C0
SEG_SIZE  = 0x6ED0
RELOC     = SEG_START + SEG_SIZE

count = struct.unpack_from("<H", data, RELOC)[0]
base = RELOC + 2

relocs = []

for i in range(count):
    p = base + i * 8

    typ    = data[p]
    flags  = data[p + 1]
    source = struct.unpack_from("<H", data, p + 2)[0]
    target = data[p + 4:p + 8]

    relocs.append({
        "index": i,
        "type": typ,
        "flags": flags,
        "source": source,
        "target": target,
    })


def target_internal(r):

    t = r["target"]

    if (r["flags"] & 3) != 0:
        return None

    seg = t[0]
    off = struct.unpack_from("<H", t, 2)[0]

    return seg, off


def word(off):
    return struct.unpack_from(
        "<H",
        data,
        SEG_START + off
    )[0]


print("============================================================")
print(" ANALISIS DE RELOCACIONES DE LAS TRES CALL FAR")
print("============================================================")
print()

print("Bytes originales de CalcularClick:")
print()

for off in range(0x1DC0, 0x1DF5, 2):

    b = data[SEG_START + off:SEG_START + off + 2]

    print(
        f"0x{off:04X}: "
        f"{b[0]:02X} {b[1]:02X} "
        f"WORD=0x{struct.unpack('<H', b)[0]:04X}"
    )

print()
print("============================================================")
print(" TIPO 05 — OFFSET")
print("============================================================")

for r in relocs:

    if r["type"] != 0x05:
        continue

    if 0x1D00 <= r["source"] <= 0x1E20:

        t = target_internal(r)

        print(
            f"registro {r['index']:3d} | "
            f"flags=0x{r['flags']:02X} | "
            f"source=0x{r['source']:04X} | "
            f"target="
            + " ".join(f"{x:02X}" for x in r["target"])
        )

        if t:
            print(
                f"    destino interno: "
                f"segmento={t[0]} "
                f"offset=0x{t[1]:04X}"
            )

print()
print("============================================================")
print(" TODOS LOS TIPO 05 CON DESTINO AL SEGMENTO 80")
print("============================================================")

for r in relocs:

    if r["type"] != 0x05:
        continue

    t = target_internal(r)

    if t and t[0] == 80:

        print(
            f"registro {r['index']:3d} | "
            f"flags=0x{r['flags']:02X} | "
            f"source=0x{r['source']:04X} | "
            f"destino=80:0x{t[1]:04X}"
        )

print()
print("============================================================")
print(" TODOS LOS TIPO 03 — FAR ADDRESS")
print("============================================================")

encontrados = 0

for r in relocs:

    if r["type"] != 0x03:
        continue

    encontrados += 1

    t = target_internal(r)

    if t:
        print(
            f"registro {r['index']:3d} | "
            f"flags=0x{r['flags']:02X} | "
            f"source=0x{r['source']:04X} | "
            f"destino={t[0]}:0x{t[1]:04X}"
        )
    else:
        print(
            f"registro {r['index']:3d} | "
            f"flags=0x{r['flags']:02X} | "
            f"source=0x{r['source']:04X} | "
            f"target="
            + " ".join(f"{x:02X}" for x in r["target"])
        )

print()
print(f"Cantidad de relocaciones tipo 03: {encontrados}")

print()
print("============================================================")
print(" TIPO 05 — TODOS LOS REGISTROS")
print("============================================================")

for r in relocs:

    if r["type"] == 0x05:

        t = target_internal(r)

        if t:
            destino = f"{t[0]}:0x{t[1]:04X}"
        else:
            destino = (
                "target="
                + " ".join(f"{x:02X}" for x in r["target"])
            )

        print(
            f"{r['index']:3d}: "
            f"flags={r['flags']:02X} "
            f"source=0x{r['source']:04X} "
            f"{destino}"
        )

print()
print("============================================================")
print(" FIN")
print("============================================================")
PY
```


Lo particularmente interesante será esta sección:


```
TIPO 05 — OFFSET
```


y después:


```
TODOS LOS TIPO 05 CON DESTINO AL SEGMENTO 80
```


Si aparece un registro tipo `05` asociado a `0x1DCB`, podremos reconstruir directamente:


```
CALL 1
    segmento = 80
    offset   = XXXX
```


y por tanto:


```
80:XXXX
```


que será la **dirección lógica NE de la primera rutina llamada por `CalcularClick`**.


Además, si resulta que **no existe ningún tipo `03`**, eso también será una información importante: significaría que este ejecutable de Delphi está construyendo las llamadas FAR mediante **dos relocaciones independientes: una `02` para el segmento y otra `05` para el offset**, en lugar de utilizar una única relocación `03`.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> from pathlib import Path
> import struct
>
> data = Path("/storage/emulated/0/Download/armon/ARMON.EXE").read_bytes()
>
> SEG_START = 0x04B2C0
> SEG_SIZE  = 0x6ED0
> RELOC     = SEG_START + SEG_SIZE
>
> count = struct.unpack_from("<H", data, RELOC)[0]
> base = RELOC + 2
>
> relocs = []
>
> for i in range(count):
>     p = base + i * 8
>
>     typ    = data[p]
>     flags  = data[p + 1]
>     source = struct.unpack_from("<H", data, p + 2)[0]
>     target = data[p + 4:p + 8]
>
>     relocs.append({
>         "index": i,
>         "type": typ,
>         "flags": flags,
>         "source": source,
>         "target": target,
>     })
>
>
> def target_internal(r):
>
>     t = r["target"]
>
>     if (r["flags"] & 3) != 0:
>         return None
>
>     seg = t[0]
>     off = struct.unpack_from("<H", t, 2)[0]
>
>     return seg, off
================================================")
print(" TODOS LOS TIPO 03 — FAR ADDRESS")
print("=========>
>
> def word(off):
>     return struct.unpack_from(
>         "<H",
>         data,
>         SEG_START + off
>     )[0]
>
>
> print("============================================================")
> print(" ANALISIS DE RELOCACIONES DE LAS TRES CALL FAR")
> print("============================================================")
:3d} | "
 > print()
>
> print("Bytes originales de CalcularClick:")
> print()
=0x{r['>
source'> for off in range(0x1DC0, 0x1DF5, 2):
t="
 >
    >     b = data[SEG_START + off:SEG_START + off + 2]
>
  >     print(
>         f"0x{off:04X}: "
t()
print(f"Cantidad de relocaciones>         f"{b[0]:02X} {b[1]:02X} "
=====>         f"WORD=0x{struct.unpack('<H', b)[0]:04X}"
>     )
>
(" TI> print()
> print("============================================================")
====")

> print(" TIPO 05 — OFFSET")
> print("============================================================")
>
internal(r)> for r in relocs:
          >
>     if r["type"] != 0x05:
>         continue
>
>     if 0x1D00 <= r["source"] <= 0x1E20:
>
>         t = target_internal(r)
>
>         print(
>             f"registro {r['index']:3d} | "
>             f"flags=0x{r['flags']:02X} | "
>             f"source=0x{r['source']:04X} | "
>             f"target="
>             + " ".join(f"{x:02X}" for x in r["target"])
>         )
>
>         if t:
>             print(
>                 f"    destino interno: "
>                 f"segmento={t[0]} "
>                 f"offset=0x{t[1]:04X}"
>             )
>
> print()
> print("==============================================)
> print("============================================================")
>
> for r in relocs:
>
>     if r["type"] != 0x05:
>         continue
>
>     t = target_internal(r)
>
>     if t and t[0] == 80:
>
>         print(
>             f"registro {r['index']:3d} | "
>             f"flags=0x{r['flags']:02X} | "
>             f"source=0x{r['source']:04X} | "
>             f"destino=80:0x{t[1]:04X}"
>         )
>
> print()
> print("============================================================")
> print(" TODOS LOS TIPO 03 — FAR ADDRESS")
> print("============================================================")
>
> encontrados = 0
>
> for r in relocs:
>
>     if r["type"] != 0x03:
>         continue
>
>     encontrados += 1
>
>     t = target_internal(r)
>
>     if t:
>         print(
>             f"registro {r['index']:3d} | "
>             f"flags=0x{r['flags']:02X} | "
>             f"source=0x{r['source']:04X} | "
>             f"destino={t[0]}:0x{t[1]:04X}"
>         )
>     else:
>         print(
>             f"registro {r['index']:3d} | "
>             f"flags=0x{r['flags']:02X} | "
>             f"source=0x{r['source']:04X} | "
>             f"target="
>             + " ".join(f"{x:02X}" for x in r["target"])
>         )
>
> print()
> print(f"Cantidad de relocaciones tipo 03: {encontrados}")
>
> print()
> print("============================================================")
> print(" TIPO 05 — TODOS LOS REGISTROS")
> print("============================================================")
>
> for r in relocs:
>
>     if r["type"] == 0x05:
>
>         t = target_internal(r)
>
>         if t:
>             destino = f"{t[0]}:0x{t[1]:04X}"
>         else:
>             destino = (
>                 "target="
>                 + " ".join(f"{x:02X}" for x in r["target"])
>             )
>
>         print(
>             f"{r['index']:3d}: "
>             f"flags={r['flags']:02X} "
>             f"source=0x{r['source']:04X} "
>             f"{destino}"
>         )
>
> print()
> print("============================================================")
> print(" FIN")
> print("============================================================")
> PY
============================================================
 ANALISIS DE RELOCACIONES DE LAS TRES CALL FAR
============================================================

Bytes originales de CalcularClick:

0x1DC0: 00 C9 WORD=0xC900
0x1DC2: C3 55 WORD=0x55C3
0x1DC4: 89 E5 WORD=0xE589
0x1DC6: B8 08 WORD=0x08B8
0x1DC8: 00 9A WORD=0x9A00
0x1DCA: 44 04 WORD=0x0444
0x1DCC: 02 1E WORD=0x1E02
0x1DCE: 83 EC WORD=0xEC83
0x1DD0: 08 C4 WORD=0xC408
0x1DD2: 7E 06 WORD=0x067E
0x1DD4: 06 57 WORD=0x5706
0x1DD6: 9A F4 WORD=0xF49A
0x1DD8: 18 EB WORD=0xEB18
0x1DDA: 1D 99 WORD=0x991D
0x1DDC: A3 5C WORD=0x5CA3
0x1DDE: 51 89 WORD=0x8951
0x1DE0: 16 5E WORD=0x5E16
0x1DE2: 51 C4 WORD=0xC451
0x1DE4: 7E 06 WORD=0x067E
0x1DE6: 06 57 WORD=0x5706
0x1DE8: 9A A9 WORD=0xA99A
0x1DEA: 18 AD WORD=0xAD18
0x1DEC: 1F 99 WORD=0x991F
0x1DEE: A3 58 WORD=0x58A3
0x1DF0: 51 89 WORD=0x8951
0x1DF2: 16 5A WORD=0x5A16
0x1DF4: 51 C9 WORD=0xC951

============================================================
 TIPO 05 — OFFSET
============================================================

============================================================
 TODOS LOS TIPO 05 CON DESTINO AL SEGMENTO 80
============================================================

============================================================
 TODOS LOS TIPO 03 — FAR ADDRESS
============================================================
registro  92 | flags=0x01 | source=0x3129 | target=06 00 45 00
registro 103 | flags=0x01 | source=0x3430 | target=07 00 68 00

Cantidad de relocaciones tipo 03: 2

============================================================
 TIPO 05 — TODOS LOS REGISTROS
============================================================
 15: flags=07 source=0x14D9 target=05 00 00 00
 16: flags=07 source=0x14DE target=05 00 00 00
 17: flags=07 source=0x14E6 target=04 00 00 00
 18: flags=07 source=0x14F0 target=02 00 00 00
 19: flags=07 source=0x14F4 target=06 00 00 00
 23: flags=07 source=0x17AA target=05 00 00 00
 24: flags=07 source=0x17AF target=03 00 00 00
 25: flags=07 source=0x17B5 target=05 00 00 00
 26: flags=07 source=0x17C4 target=05 00 00 00
 27: flags=07 source=0x17C9 target=03 00 00 00
 28: flags=07 source=0x17CF target=05 00 00 00
 29: flags=07 source=0x17E6 target=05 00 00 00
 30: flags=07 source=0x17EB target=03 00 00 00
 31: flags=07 source=0x17F1 target=05 00 00 00
 33: flags=07 source=0x1926 target=05 00 00 00
 34: flags=07 source=0x192B target=03 00 00 00
 35: flags=07 source=0x1931 target=05 00 00 00
 38: flags=07 source=0x1F4E target=05 00 00 00
 39: flags=07 source=0x1F52 target=05 00 00 00
 40: flags=07 source=0x1F57 target=06 00 00 00
 41: flags=07 source=0x1F70 target=05 00 00 00
 42: flags=07 source=0x1F75 target=05 00 00 00
 47: flags=07 source=0x2825 target=03 00 00 00
 48: flags=07 source=0x282B target=05 00 00 00
 49: flags=07 source=0x2830 target=06 00 00 00
 52: flags=07 source=0x2A4A target=05 00 00 00
 53: flags=07 source=0x2A4F target=06 00 00 00
 55: flags=07 source=0x2A86 target=05 00 00 00
 56: flags=07 source=0x2AA6 target=03 00 00 00
 57: flags=07 source=0x2AAC target=05 00 00 00
 58: flags=07 source=0x2AB1 target=05 00 00 00
 59: flags=07 source=0x2AB4 target=05 00 00 00
 60: flags=07 source=0x2AB9 target=06 00 00 00
 61: flags=07 source=0x2AC2 target=05 00 00 00
 62: flags=07 source=0x2AC6 target=03 00 00 00
 63: flags=07 source=0x2ACC target=05 00 00 00
 64: flags=07 source=0x2AD1 target=05 00 00 00
 65: flags=07 source=0x2AD6 target=06 00 00 00
 66: flags=07 source=0x2ADF target=05 00 00 00
 67: flags=07 source=0x2AE3 target=03 00 00 00
 68: flags=07 source=0x2AE9 target=05 00 00 00
 69: flags=07 source=0x2AEE target=05 00 00 00
 70: flags=07 source=0x2AF3 target=06 00 00 00
 71: flags=07 source=0x2AFC target=05 00 00 00
 72: flags=07 source=0x2B00 target=03 00 00 00
 73: flags=07 source=0x2B06 target=05 00 00 00
 74: flags=07 source=0x2B0B target=05 00 00 00
 75: flags=07 source=0x2B10 target=06 00 00 00
 76: flags=07 source=0x2B19 target=05 00 00 00
 77: flags=07 source=0x2B1E target=05 00 00 00
 78: flags=07 source=0x2B22 target=05 00 00 00
 79: flags=07 source=0x2B27 target=06 00 00 00
 80: flags=07 source=0x2B33 target=05 00 00 00
 81: flags=07 source=0x2BA8 target=05 00 00 00
 82: flags=07 source=0x2BAD target=06 00 00 00
 83: flags=07 source=0x2BB5 target=05 00 00 00
 84: flags=07 source=0x2BF0 target=05 00 00 00
 85: flags=07 source=0x2BF5 target=06 00 00 00
 86: flags=07 source=0x2D54 target=05 00 00 00
 87: flags=07 source=0x2D5C target=02 00 00 00
 88: flags=07 source=0x2D60 target=06 00 00 00
 89: flags=07 source=0x2D92 target=02 00 00 00
 90: flags=07 source=0x2D96 target=06 00 00 00
 94: flags=07 source=0x338B target=03 00 00 00
 95: flags=07 source=0x3391 target=05 00 00 00
 96: flags=07 source=0x3396 target=06 00 00 00
 97: flags=07 source=0x33A8 target=05 00 00 00
 98: flags=07 source=0x33B1 target=02 00 00 00
 99: flags=07 source=0x33B5 target=06 00 00 00
100: flags=07 source=0x33C7 target=05 00 00 00
101: flags=07 source=0x33D1 target=02 00 00 00
102: flags=07 source=0x33D5 target=06 00 00 00
104: flags=07 source=0x37EF target=03 00 00 00
105: flags=07 source=0x37F5 target=05 00 00 00
106: flags=07 source=0x37FA target=06 00 00 00
107: flags=07 source=0x37FC target=03 00 00 00
108: flags=07 source=0x3802 target=05 00 00 00
109: flags=07 source=0x3807 target=06 00 00 00
110: flags=07 source=0x381F target=03 00 00 00
111: flags=07 source=0x3825 target=05 00 00 00
112: flags=07 source=0x382A target=06 00 00 00
113: flags=07 source=0x382C target=03 00 00 00
114: flags=07 source=0x3832 target=05 00 00 00
115: flags=07 source=0x3837 target=06 00 00 00
116: flags=07 source=0x384F target=03 00 00 00
117: flags=07 source=0x3855 target=05 00 00 00
118: flags=07 source=0x385A target=06 00 00 00
119: flags=07 source=0x385C target=03 00 00 00
120: flags=07 source=0x3862 target=05 00 00 00
121: flags=07 source=0x3867 target=06 00 00 00
122: flags=07 source=0x387F target=03 00 00 00
123: flags=07 source=0x3885 target=05 00 00 00
124: flags=07 source=0x388A target=06 00 00 00
125: flags=07 source=0x388C target=03 00 00 00
126: flags=07 source=0x3892 target=05 00 00 00
127: flags=07 source=0x3897 target=06 00 00 00
128: flags=07 source=0x38AF target=03 00 00 00
129: flags=07 source=0x38B5 target=05 00 00 00
130: flags=07 source=0x38BA target=06 00 00 00
131: flags=07 source=0x38BC target=03 00 00 00
132: flags=07 source=0x38C2 target=05 00 00 00
133: flags=07 source=0x38C7 target=06 00 00 00
134: flags=07 source=0x38E4 target=05 00 00 00
135: flags=07 source=0x38E9 target=03 00 00 00
136: flags=07 source=0x38EF target=04 00 00 00
137: flags=07 source=0x38F3 target=03 00 00 00
138: flags=07 source=0x38F9 target=05 00 00 00
139: flags=07 source=0x390B target=05 00 00 00
140: flags=07 source=0x3910 target=03 00 00 00
141: flags=07 source=0x3919 target=04 00 00 00
142: flags=07 source=0x391D target=03 00 00 00
143: flags=07 source=0x3923 target=05 00 00 00
144: flags=07 source=0x3B7C target=05 00 00 00
145: flags=07 source=0x3B81 target=03 00 00 00
146: flags=07 source=0x3B87 target=05 00 00 00
147: flags=07 source=0x3BA4 target=05 00 00 00
148: flags=07 source=0x3BA9 target=03 00 00 00
149: flags=07 source=0x3BAF target=05 00 00 00
151: flags=07 source=0x475A target=05 00 00 00
152: flags=07 source=0x475E target=06 00 00 00
153: flags=07 source=0x4766 target=05 00 00 00
154: flags=07 source=0x476A target=05 00 00 00
155: flags=07 source=0x476F target=06 00 00 00
156: flags=07 source=0x479A target=05 00 00 00
157: flags=07 source=0x479F target=03 00 00 00
158: flags=07 source=0x47AA target=02 00 00 00
159: flags=07 source=0x47AE target=06 00 00 00
160: flags=07 source=0x47EC target=05 00 00 00
161: flags=07 source=0x47F1 target=03 00 00 00
162: flags=07 source=0x47F7 target=05 00 00 00
163: flags=07 source=0x47FB target=06 00 00 00
164: flags=07 source=0x480E target=05 00 00 00
165: flags=07 source=0x4813 target=03 00 00 00
166: flags=07 source=0x481E target=02 00 00 00
167: flags=07 source=0x4822 target=06 00 00 00
168: flags=07 source=0x485B target=05 00 00 00
169: flags=07 source=0x4860 target=03 00 00 00
170: flags=07 source=0x4866 target=05 00 00 00
171: flags=07 source=0x486B target=06 00 00 00
172: flags=07 source=0x48BA target=05 00 00 00
173: flags=07 source=0x48BF target=03 00 00 00
174: flags=07 source=0x48C5 target=05 00 00 00
175: flags=07 source=0x48C9 target=06 00 00 00
176: flags=07 source=0x48D1 target=05 00 00 00
177: flags=07 source=0x48D6 target=03 00 00 00
178: flags=07 source=0x48DC target=05 00 00 00
179: flags=07 source=0x48E1 target=06 00 00 00
180: flags=07 source=0x4959 target=05 00 00 00
181: flags=07 source=0x495E target=06 00 00 00
182: flags=07 source=0x4B08 target=05 00 00 00
183: flags=07 source=0x4B0D target=06 00 00 00
186: flags=07 source=0x5D43 target=05 00 00 00
187: flags=07 source=0x5D48 target=06 00 00 00
188: flags=07 source=0x66B4 target=05 00 00 00
189: flags=07 source=0x66B9 target=03 00 00 00
190: flags=07 source=0x66C4 target=02 00 00 00
191: flags=07 source=0x66C8 target=06 00 00 00
192: flags=07 source=0x670A target=05 00 00 00
193: flags=07 source=0x670F target=03 00 00 00
194: flags=07 source=0x671A target=02 00 00 00
195: flags=07 source=0x671E target=06 00 00 00
196: flags=07 source=0x678A target=05 00 00 00
197: flags=07 source=0x678E target=03 00 00 00
198: flags=07 source=0x6799 target=02 00 00 00
199: flags=07 source=0x679D target=06 00 00 00
200: flags=07 source=0x680E target=05 00 00 00
201: flags=07 source=0x6812 target=03 00 00 00
202: flags=07 source=0x681D target=02 00 00 00
203: flags=07 source=0x6821 target=06 00 00 00
204: flags=07 source=0x6892 target=05 00 00 00
205: flags=07 source=0x6896 target=03 00 00 00
206: flags=07 source=0x68A1 target=02 00 00 00
207: flags=07 source=0x68A5 target=06 00 00 00
208: flags=07 source=0x6916 target=05 00 00 00
209: flags=07 source=0x691A target=03 00 00 00
210: flags=07 source=0x6925 target=02 00 00 00
211: flags=07 source=0x6929 target=06 00 00 00
212: flags=07 source=0x6B33 target=03 00 00 00
213: flags=07 source=0x6B39 target=05 00 00 00
214: flags=07 source=0x6B3E target=05 00 00 00
215: flags=07 source=0x6B43 target=06 00 00 00
216: flags=07 source=0x6B98 target=05 00 00 00
217: flags=07 source=0x6B9D target=03 00 00 00
218: flags=07 source=0x6BA3 target=05 00 00 00
219: flags=07 source=0x6BA7 target=06 00 00 00
220: flags=07 source=0x6BAF target=03 00 00 00
221: flags=07 source=0x6BB5 target=05 00 00 00
222: flags=07 source=0x6BBA target=05 00 00 00
223: flags=07 source=0x6BBF target=06 00 00 00
224: flags=07 source=0x6C0C target=03 00 00 00
225: flags=07 source=0x6C12 target=05 00 00 00
226: flags=07 source=0x6C17 target=05 00 00 00
227: flags=07 source=0x6C1C target=05 00 00 00
228: flags=07 source=0x6C21 target=06 00 00 00
229: flags=07 source=0x6C76 target=05 00 00 00
230: flags=07 source=0x6C7B target=03 00 00 00
231: flags=07 source=0x6C81 target=05 00 00 00
232: flags=07 source=0x6C85 target=06 00 00 00
233: flags=07 source=0x6C8D target=03 00 00 00
234: flags=07 source=0x6C93 target=05 00 00 00
235: flags=07 source=0x6C98 target=05 00 00 00
236: flags=07 source=0x6C9D target=05 00 00 00
237: flags=07 source=0x6CA2 target=06 00 00 00
238: flags=07 source=0x6E0C target=05 00 00 00
239: flags=07 source=0x6E11 target=06 00 00 00
240: flags=07 source=0x6E65 target=05 00 00 00
241: flags=07 source=0x6E69 target=03 00 00 00
242: flags=07 source=0x6E74 target=02 00 00 00
243: flags=07 source=0x6E78 target=06 00 00 00

============================================================
 FIN
============================================================
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Ahora sí tenemos una conclusión muy importante. **La primera llamada ya está completamente resuelta:**


```
CALL FAR 80:0444
```


La razón es que cometimos antes un pequeño desplazamiento al identificar los bytes del `CALL`. En realidad:


```
9A 44 04 02 1E
   │────│ │────│
   offset segmento
   0444   1E02*
```


El `0x1E02` **no es el segmento definitivo en el archivo**; es el enlace de la cadena de relocación. La cadena nos llevó hasta `0x1DCC`, cuya relocación dice:


```
tipo = 02
destino = segmento 80
```


Y hay otro detalle decisivo: **no existe ninguna relocación tipo 05 para ese offset porque el offset `0444` no necesita ser relocalizado**. El loader solamente tiene que sustituir el segmento cuando carga el módulo.


Por tanto:


```
CalcularClick
    │
    └── CALL FAR 80:0444
```


Los otros dos `CALL FAR` se pueden resolver de la misma manera, pero ahora debemos usar **las posiciones reales** de sus operandos:


```
CALL 1
9A 44 04 02 1E
   ↑
offset en 0x1DCA
segmento en 0x1DCC

CALL 2
9A F4 18 EB 1D
   ↑
offset en 0x1DD7
segmento en 0x1DD9

CALL 3
9A A9 18 AD 1F
   ↑
offset en 0x1DE9
segmento en 0x1DEB
```


Es decir, los offsets que debemos localizar mediante las cadenas son **0x1DCC, 0x1DD9 y 0x1DEB**, no los que estábamos utilizando para las llamadas 2 y 3.


Ejecuta este análisis, que ya parte de las posiciones correctas y además muestra directamente el `CALL` reconstruido:


```
Bashcd /storage/emulated/0/Download/armon

python3 - <<'PY'
from pathlib import Path
import struct

data = Path("/storage/emulated/0/Download/armon/ARMON.EXE").read_bytes()

SEG_START = 0x04B2C0
SEG_SIZE  = 0x6ED0
RELOC     = SEG_START + SEG_SIZE

count = struct.unpack_from("<H", data, RELOC)[0]
base = RELOC + 2

relocs = []

for i in range(count):
    p = base + i * 8

    relocs.append({
        "index": i,
        "type": data[p],
        "flags": data[p + 1],
        "source": struct.unpack_from("<H", data, p + 2)[0],
        "target": data[p + 4:p + 8],
    })


def word(off):
    return struct.unpack_from(
        "<H",
        data,
        SEG_START + off
    )[0]


def chain(start):

    result = []
    seen = set()
    cur = start

    while cur != 0xFFFF:

        if cur in seen:
            result.append(cur)
            break

        seen.add(cur)

        if cur >= SEG_SIZE - 1:
            break

        result.append(cur)
        cur = word(cur)

    return result


def target_segment(r):

    # Tipo 02 + referencia interna.
    if r["type"] == 0x02 and (r["flags"] & 3) == 0:
        return r["target"][0]

    return None


# ------------------------------------------------------------
# POSICIONES REALES DE LOS CALL FAR
# ------------------------------------------------------------

calls = [
    {
        "nombre": "CALL 1",
        "opcode": 0x1DC9,
        "offset": 0x1DCA,
        "segmento": 0x1DCC,
        "offset_real": 0x0444,
    },
    {
        "nombre": "CALL 2",
        "opcode": 0x1DD7,
        "offset": 0x1DD7,
        "segmento": 0x1DD9,
        "offset_real": 0x18F4,
    },
    {
        "nombre": "CALL 3",
        "opcode": 0x1DE9,
        "offset": 0x1DE9,
        "segmento": 0x1DEB,
        "offset_real": 0x18A9,
    },
]


print("============================================================")
print(" RESOLUCION DE LOS CALL FAR DE CalcularClick")
print("============================================================")

for c in calls:

    print()
    print(c["nombre"])

    print(
        f"  opcode       = 0x{c['opcode']:04X}"
    )

    print(
        f"  offset       = 0x{c['offset']:04X}"
    )

    print(
        f"  segmento     = 0x{c['segmento']:04X}"
    )

    print(
        f"  offset bruto = 0x{c['offset_real']:04X}"
    )

    # --------------------------------------------------------
    # Buscar la relocación de segmento.
    # --------------------------------------------------------

    encontrada = None
    cadena_encontrada = None

    for r in relocs:

        if r["type"] != 0x02:
            continue

        cadena = chain(r["source"])

        if c["segmento"] in cadena:

            encontrada = r
            cadena_encontrada = cadena
            break

    if encontrada is None:

        print("  segmento -> NO RESUELTO")

        continue

    seg = target_segment(encontrada)

    print()
    print(
        f"  relocación  = registro {encontrada['index']}"
    )

    print(
        f"  cadena inicia en = 0x{encontrada['source']:04X}"
    )

    print(
        f"  segmento destino = {seg}"
    )

    print(
        f"  offset destino   = 0x{c['offset_real']:04X}"
    )

    print()
    print(
        f"  *** DIRECCION LOGICA = {seg}:"
        f"{c['offset_real']:04X} ***"
    )

    # Mostrar solamente el tramo de cadena próximo al objetivo.
    if c["segmento"] in cadena_encontrada:

        pos = cadena_encontrada.index(c["segmento"])

        inicio = max(0, pos - 3)
        fin = min(len(cadena_encontrada), pos + 4)

        tramo = cadena_encontrada[inicio:fin]

        print(
            "  tramo cadena = "
            + " -> ".join(
                f"0x{x:04X}" for x in tramo
            )
        )


print()
print("============================================================")
print(" RESULTADO FINAL")
print("============================================================")

print()

for c in calls:

    encontrada = None

    for r in relocs:

        if r["type"] != 0x02:
            continue

        cadena = chain(r["source"])

        if c["segmento"] in cadena:
            encontrada = r
            break

    if encontrada:

        seg = target_segment(encontrada)

        print(
            f"{c['nombre']:7s} -> "
            f"{seg}:{c['offset_real']:04X}"
        )

    else:

        print(
            f"{c['nombre']:7s} -> NO RESUELTO"
        )

print()
print("============================================================")
print(" FIN")
print("============================================================")
PY
```


### Lo que ya podemos afirmar


La primera llamada es:


```
CalcularClick
    ↓
CALL FAR 80:0444
```


Y si las otras dos cadenas se resuelven igual, tendremos las tres direcciones:


```
CalcularClick
 ├── 80:0444
 ├── XX:18F4
 └── YY:18A9
```


Después podremos extraer **los segmentos 80, XX e YY**, localizar exactamente `0444`, `18F4` y `18A9` dentro de ellos y desensamblar las tres rutinas. Ese será el primer punto en que podremos empezar a reconstruir la lógica Pascal de `CalcularClick` con bastante precisión.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> from pathlib import Path
> import struct
>
> data = Path("/storage/emulated/0/Download/armon/ARMON.EXE").read_bytes()
>
> SEG_START = 0x04B2C0
> SEG_SIZE  = 0x6ED0
> RELOC     = SEG_START + SEG_SIZE
>
> count = struct.unpack_from("<H", data, RELOC)[0]
> base = RELOC + 2
>
> relocs = []
>
> for i in range(count):
>     p = base + i * 8
>
>     relocs.append({
>         "index": i,
>         "type": data[p],
>         "flags": data[p + 1],
>         "source": struct.unpack_from("<H", data, p + 2)[0],
>         "target": data[p + 4:p + 8],
C,
        "offset_real": 0x0444,
    },
    {
        "nom>     })
>
>
> def word(off):
>     return struct.unpack_from(
>         "<H",
>         data,
>         SEG_START + off
>     )[0]
>
>
> def chain(start):
>
>     result = []
>     seen = set()
>     cur = start
>
>     while cur != 0xFFFF:
>
>         if cur in seen:
>             result.append(cur)
>             break
>
>         seen.add(cur)
>
>         if cur >= SEG_SIZE - 1:
>             break
>
>         result.append(cur)
>         cur = word(cur)
"] != >
>     return result
["segme>
>
ena:

    > def target_segment(r):
>
>     # Tipo 02 + referencia interna.
>     if r["type"] == 0x02 and (r["flags"] & 3) == 0:
   if encontrada is None:

        print("  segmento -> NO RESUELTO")

        continue

    seg = target_segment(encontrada)

    print()
    print(
        f"  relocación  = registro {encontrada['index']}"
    )

    print(
        f"  cadena inicia en = 0x{encontrada['source']:04X}"
    )

    print(
        f"  segmento destino = {seg}"
    )

    print(
        f"  of>         return r["target"][0]
)

    # Mostr>
amente>     return None
>
>
mo a> # ------------------------------------------------------------
> # POSICIONES REALES DE LOS CALL FAR
ax(0, > # ------------------------------------------------------------
>
> calls = [
        tramo>     {
>         "nombre": "CALL 1",
a>         "opcode": 0x1DC9,
>         "offset": 0x1DCA,
x in >         "segmento": 0x1DCC,
>         "offset_real": 0x0444,
>     },
>     {
>         "nombre": "CALL 2",
>         "opcode": 0x1DD7,
>         "offset": 0x1DD7,
>         "segmento": 0x1DD9,
>         "offset_real": 0x18F4,
>     },
] in cadena:
      >     {
  >         "nombre": "CALL 3",
>         "opcode": 0x1DE9,
>         "offset": 0x1DE9,
>         "segmento": 0x1DEB,
>         "offset_real": 0x18A9,
>     },
> ]
>
>
> print("============================================================")
> print(" RESOLUCION DE LOS CALL FAR DE CalcularClick")
> print("============================================================")
>
================> for c in calls:
>
>     print()
>     print(c["nombre"])
>
>     print(
>         f"  opcode       = 0x{c['opcode']:04X}"
>     )
>
>     print(
>         f"  offset       = 0x{c['offset']:04X}"
>     )
>
>     print(
>         f"  segmento     = 0x{c['segmento']:04X}"
>     )
>
>     print(
>         f"  offset bruto = 0x{c['offset_real']:04X}"
>     )
>
>     # --------------------------------------------------------
>     # Buscar la relocación de segmento.
>     # --------------------------------------------------------
>
>     encontrada = None
>     cadena_encontrada = None
>
>     for r in relocs:
>
>         if r["type"] != 0x02:
>             continue
>
>         cadena = chain(r["source"])
>
>         if c["segmento"] in cadena:
>
>             encontrada = r
>             cadena_encontrada = cadena
>             break
>
>     if encontrada is None:
>
>         print("  segmento -> NO RESUELTO")
>
>         continue
>
>     seg = target_segment(encontrada)
>
>     print()
>     print(
>         f"  relocación  = registro {encontrada['index'
]}"
>     )
>
>     print(
>         f"  cadena inicia en = 0x{encontrada['source']:04X}"
>     )
>
>     print(
>         f"  segmento destino = {seg}"
>     )
>
>     print(
>         f"  offset destino   = 0x{c['offset_real']:04X}"
>     )
>
>     print()
>     print(
>         f"  *** DIRECCION LOGICA = {seg}:"
>         f"{c['offset_real']:04X} ***"
>     )
>
>     # Mostrar solamente el tramo de cadena próximo al
objetivo.
>     if c["segmento"] in cadena_encontrada:
>
>         pos = cadena_encontrada.index(c["segmento"])
>
>         inicio = max(0, pos - 3)
>         fin = min(len(cadena_encontrada), pos + 4)
>
>         tramo = cadena_encontrada[inicio:fin]
>
>         print(
>             "  tramo cadena = "
>             + " -> ".join(
>                 f"0x{x:04X}" for x in tramo
>             )
>         )
>
>
> print()
> print("============================================================")
> print(" RESULTADO FINAL")
> print("============================================================")
>
> print()
>
> for c in calls:
>
>     encontrada = None
>
>     for r in relocs:
>
>         if r["type"] != 0x02:
>             continue
>
>         cadena = chain(r["source"])
>
>         if c["segmento"] in cadena:
>             encontrada = r
>             break
>
>     if encontrada:
>
>         seg = target_segment(encontrada)
>
>         print(
>             f"{c['nombre']:7s} -> "
>             f"{seg}:{c['offset_real']:04X}"
>         )
>
>     else:
>
>         print(
>             f"{c['nombre']:7s} -> NO RESUELTO"
>         )
>
> print()
> print("============================================================")
> print(" FIN")
> print("============================================================")
> PY
============================================================
 RESOLUCION DE LOS CALL FAR DE CalcularClick
============================================================

CALL 1
  opcode       = 0x1DC9
  offset       = 0x1DCA
  segmento     = 0x1DCC
  offset bruto = 0x0444

  relocación  = registro 3
  cadena inicia en = 0x001C
  segmento destino = 80
  offset destino   = 0x0444

  *** DIRECCION LOGICA = 80:0444 ***
  tramo cadena = 0x1C6E -> 0x1D09 -> 0x1D3C -> 0x1DCC -> 0x1E02 -> 0x1E22 -> 0x1ECB

CALL 2
  opcode       = 0x1DD7
  offset       = 0x1DD7
  segmento     = 0x1DD9
  offset bruto = 0x18F4

  relocación  = registro 1
  cadena inicia en = 0x0078
  segmento destino = 76
  offset destino   = 0x18F4

  *** DIRECCION LOGICA = 76:18F4 ***
  tramo cadena = 0x1D6C -> 0x1D84 -> 0x1D94 -> 0x1DD9 -> 0x1DEB -> 0x1FAD -> 0x257A

CALL 3
  opcode       = 0x1DE9
  offset       = 0x1DE9
  segmento     = 0x1DEB
  offset bruto = 0x18A9

  relocación  = registro 1
  cadena inicia en = 0x0078
  segmento destino = 76
  offset destino   = 0x18A9

  *** DIRECCION LOGICA = 76:18A9 ***
  tramo cadena = 0x1D84 -> 0x1D94 -> 0x1DD9 -> 0x1DEB -> 0x1FAD -> 0x257A -> 0x258C

============================================================
 RESULTADO FINAL
============================================================

CALL 1  -> 80:0444
CALL 2  -> 76:18F4
CALL 3  -> 76:18A9

============================================================
 FIN
============================================================
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Sí. Este resultado permite corregir una cosa importante de la interpretación anterior.


### Lo que ya podemos afirmar


Las tres llamadas son:


```
CALL FAR 1 → 1E02:0444   → segmento lógico 80:0444
CALL FAR 2 → 1DEB:18F4  → segmento lógico 76:18F4
CALL FAR 3 → 1FA9:18A9  → segmento lógico 76:18A9
```


Pero **todavía no debemos llamar a `80:0444`, `76:18F4` y `76:18A9` direcciones definitivas de las rutinas**.


La razón es que acabamos de descubrir algo más preciso: en una NE, la entrada de relocación de tipo `02` corrige la **palabra segmento**, mientras que el tratamiento del desplazamiento depende de la clase de relocación. Además, `flags=07` significa que esas entradas `type=05` que estamos viendo no son referencias internas normales; por tanto, no podemos emparejarlas simplemente como hicimos antes.


En particular, tu salida demuestra algo muy valioso:


> **La relocación 3 modifica la palabra segmento situada en `0x1DCC` y la convierte en el segmento 80.**


Y la relocación 1 modifica tanto `0x1DD9` como `0x1DEB`, convirtiéndolas en el segmento 76.


Eso explica perfectamente por qué los valores que veíamos originalmente (`1E02`, `1DEB`, `1FA9`) **no eran segmentos reales**, sino eslabones de las cadenas de relocación.


La estructura NE utiliza precisamente cadenas de offsets de origen para las correcciones de relocación.


### Hay otro detalle importante


Tu segundo y tercer `CALL FAR` quedan:


```
9A F4 18 1? ?
```


y


```
9A A9 18 1? ?
```


Es decir:


```
CALL FAR 76:18F4
CALL FAR 76:18A9
```


Esto es especialmente interesante porque **ambas llamadas pertenecen al mismo segmento 76**.


Por tanto, lo siguiente que debemos hacer ya no es seguir buscando a ciegas. Debemos identificar **qué procedimiento comienza en `segmento 76 + offset 18F4` y cuál comienza en `76:18A9`**, y después hacer lo mismo con `80:0444`.


Para ello necesitamos localizar físicamente los segmentos 76 y 80 en el ejecutable y desensamblar esos offsets.


Ejecuta **este único bloque**:


```
Bashcd /storage/emulated/0/Download/armon

python3 - <<'PY'
from pathlib import Path
import struct

EXE = Path("ARMON.EXE")
data = EXE.read_bytes()

# Tabla NE
ne = 0x100
seg_count = struct.unpack_from("<H", data, ne + 0x1C)[0]
seg_table = ne + struct.unpack_from("<H", data, ne + 0x22)[0]
shift = struct.unpack_from("<H", data, ne + 0x32)[0]

print("=" * 70)
print(" LOCALIZACION DE LOS SEGMENTOS 76 Y 80")
print("=" * 70)
print(f"Segmentos NE : {seg_count}")
print(f"Shift        : {shift}  (unidad={1 << shift} bytes)")
print()

segmentos = {}

for n in range(1, seg_count + 1):
    p = seg_table + (n - 1) * 8

    sector = struct.unpack_from("<H", data, p)[0]
    length = struct.unpack_from("<H", data, p + 2)[0]
    flags  = struct.unpack_from("<H", data, p + 4)[0]
    minalloc = struct.unpack_from("<H", data, p + 6)[0]

    if length == 0:
        real_size = 0x10000
    else:
        real_size = length

    file_offset = sector << shift

    segmentos[n] = {
        "file_offset": file_offset,
        "size": real_size,
        "flags": flags,
        "minalloc": minalloc,
    }

    if n in (76, 80):
        print(
            f"SEGMENTO {n:02d}: "
            f"archivo=0x{file_offset:06X} "
            f"tam=0x{real_size:04X} "
            f"flags=0x{flags:04X}"
        )

print()

# ------------------------------------------------------------
# Extraer los tres destinos
# ------------------------------------------------------------

destinos = [
    ("CALL 1", 80, 0x0444),
    ("CALL 2", 76, 0x18F4),
    ("CALL 3", 76, 0x18A9),
]

print("=" * 70)
print(" CONTEXTO BINARIO DE LOS TRES DESTINOS")
print("=" * 70)

for nombre, seg, off in destinos:

    if seg not in segmentos:
        print(f"{nombre}: segmento {seg} no encontrado")
        continue

    s = segmentos[seg]
    start = s["file_offset"]

    if off >= s["size"]:
        print(
            f"{nombre}: offset 0x{off:04X} "
            f"fuera del segmento"
        )
        continue

    pos = start + off

    print()
    print(
        f"{nombre}: "
        f"segmento={seg} "
        f"offset=0x{off:04X} "
        f"archivo=0x{pos:06X}"
    )

    # 32 bytes antes y 96 después
    a = max(0, off - 32)
    b = min(s["size"], off + 96)

    print(
        f"  rango segmento: "
        f"0x{a:04X}..0x{b:04X}"
    )

    chunk = data[start + a:start + b]

    for i in range(0, len(chunk), 16):

        row = chunk[i:i+16]

        print(
            f"  {a+i:04X}: "
            + " ".join(f"{x:02X}" for x in row)
        )

print()

# ------------------------------------------------------------
# Guardar los segmentos completos para desensamblarlos
# ------------------------------------------------------------

out = Path.home() / "armon_code" / "destinos"
out.mkdir(parents=True, exist_ok=True)

for seg in (76, 80):

    if seg not in segmentos:
        continue

    s = segmentos[seg]

    blob = data[
        s["file_offset"]:
        s["file_offset"] + s["size"]
    ]

    filename = out / f"segmento_{seg:02d}.bin"
    filename.write_bytes(blob)

    print(
        f"Guardado: {filename} "
        f"({len(blob)} bytes)"
    )

print()
print("=" * 70)
print(" FIN")
print("=" * 70)
PY
```


**Lo que buscamos en la siguiente salida** es particularmente sencillo:


- en el segmento **76**, qué bytes hay alrededor de `0x18A9`;
- en el segmento **76**, qué bytes hay alrededor de `0x18F4`;
- en el segmento **80**, qué bytes hay alrededor de `0x0444`.


Si allí aparecen nuevamente patrones como:


```
55 89 E5
```


habremos localizado directamente **tres procedimientos Delphi internos**.


Y entonces podremos empezar a construir el camino:


```
CalcularClick
     │
     ├── segmento 80:0444
     │       └── procedimiento Delphi A
     │
     ├── segmento 76:18F4
     │       └── procedimiento Delphi B
     │
     └── segmento 76:18A9
             └── procedimiento Delphi C
```


Eso ya nos acerca bastante más a reconstruir la lógica original de `CalcularClick`, en lugar de limitarnos a desensamblar instrucciones aisladas.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> from pathlib import Path
> import struct
>
> EXE = Path("ARMON.EXE")
> data = EXE.read_bytes()
>
> # Tabla NE
> ne = 0x100
> seg_count = struct.unpack_from("<H", data, ne + 0x1C)[0]
> seg_table = ne + struct.unpack_from("<H", data, ne + 0x22)[0]
> shift = struct.unpack_from("<H", data, ne + 0x32)[0]
>
> print("=" * 70)
> print(" LOCALIZACION DE LOS SEGMENTOS 76 Y 80")
> print("=" * 70)
> print(f"Segmentos NE : {seg_count}")
> print(f"Shift        : {shift}  (unidad={1 << shift} bytes)")
> print()
>
> segmentos = {}
>
> for n in range(1, seg_count + 1):
>     p = seg_table + (n - 1) * 8
>
>     sector = struct.unpack_from("<H", data, p)[0]
>     length = struct.unpack_from("<H", data, p + 2)[0]
>     flags  = struct.unpack_from("<H", data, p + 4)[0]
>     minalloc = struct.unpack_from("<H", data, p + 6)[0]
>
>     if length == 0:
>         real_size = 0x10000
>     else:
>         real_size = length
>
>     file_offset = sector << shift
>
>     segmentos[n] = {
>         "file_offset": file_offset,
>         "size": real_size,
>         "flags": flags,
>         "minalloc": minalloc,
>     }
>
>     if n in (76, 80):
>         print(
>             f"SEGMENTO {n:02d}: "
>             f"archivo=0x{file_offset:06X} "
>             f"tam=0x{real_size:04X} "
>             f"flags=0x{flags:04X}"
>         )
>
> print()
>
> # ------------------------------------------------------------
> # Extraer los tres destinos
> # ------------------------------------------------------------
>
> destinos = [
>     ("CALL 1", 80, 0x0444),
>     ("CALL 2", 76, 0x18F4),
>     ("CALL 3", 76, 0x18A9),
> ]
>
> print("=" * 70)
> print(" CONTEXTO BINARIO DE LOS TRES DESTINOS")
> print("=" * 70)
>
> for nombre, seg, off in destinos:
>
>     if seg not in segmentos:
>         print(f"{nombre}: segmento {seg} no encontrado")
>         continue
>
>     s = segmentos[seg]
>     start = s["file_offset"]
>
>     if off >= s["size"]:
>         print(
>             f"{nombre}: offset 0x{off:04X} "
>             f"fuera del segmento"
>         )
>         continue
>
>     pos = start + off
>
>     print()
>     print(
>         f"{nombre}: "
>         f"segmento={seg} "
>         f"offset=0x{off:04X} "
>         f"archivo=0x{pos:06X}"
>     )
>
>     # 32 bytes antes y 96 después
>     a = max(0, off - 32)
>     b = min(s["size"], off + 96)
>
>     print(
>         f"  rango segmento: "
>         f"0x{a:04X}..0x{b:04X}"
>     )
>
>     chunk = data[start + a:start + b]
>
>     for i in range(0, len(chunk), 16):
>
>         row = chunk[i:i+16]
>
>         print(
>             f"  {a+i:04X}: "
>             + " ".join(f"{x:02X}" for x in row)
>         )
>
> print()
>
> # ---------------------------------------------------s
> # ------------------------------------------------------------
>
> out = Path.home() / "armon_code" / "destinos"
> out.mkdir(parents=True, exist_ok=True)
>
> for seg in (76, 80):
>
>     if seg not in segmentos:
>         continue
>
>     s = segmentos[seg]
>
>     blob = data[
>         s["file_offset"]:
>         s["file_offset"] + s["size"]
>     ]
>
>     filename = out / f"segmento_{seg:02d}.bin"
>     filename.write_bytes(blob)
>
>     print(
>         f"Guardado: {filename} "
>         f"({len(blob)} bytes)"
>     )
>
> print()
> print("=" * 70)
> print(" FIN")
> print("=" * 70)
> PY
======================================================================
 LOCALIZACION DE LOS SEGMENTOS 76 Y 80
======================================================================
Segmentos NE : 81
Shift        : 6  (unidad=64 bytes)

SEGMENTO 76: archivo=0x01BAC0 tam=0x6EB3 flags=0x1D50
SEGMENTO 80: archivo=0x032A00 tam=0x261A flags=0x0D50

======================================================================
 CONTEXTO BINARIO DE LOS TRES DESTINOS
======================================================================

CALL 1: segmento=80 offset=0x0444 archivo=0x032E44
  rango segmento: 0x0424..0x04A4
  0424: 26 3B 05 72 0F 26 3B 55 06 7C 08 7F 07 26 3B 45
  0434: 04 77 01 CB B8 04 00 E9 22 FC B8 05 00 E9 1C FC
  0444: 05 00 04 72 19 2B C4 73 15 F7 D8 36 3B 06 0A 00
  0454: 72 0C 36 3B 06 0C 00 73 04 36 A3 0C 00 CB B8 CA
  0464: 00 E9 27 FC C6 06 44 25 01 2E 80 3E AF 04 CD 74
  0474: 3D 55 8B EC 83 EC 0A 50 DB 7E F6 DD 06 84 25 9B
  0484: DC 36 8C 25 9B DC 0E 8C 25 9B DC 2E 84 25 9B DC
  0494: 1E 94 25 9B 9B DF E0 9B 25 00 01 C1 E8 07 48 A2

CALL 2: segmento=76 offset=0x18F4 archivo=0x01D3B4
  rango segmento: 0x18D4..0x1954
  18D4: C4 7E 06 06 57 9A F4 18 EE 18 50 9A 6E 06 2D 19
  18E4: 52 50 C4 7E 06 06 57 9A EB 1B 24 19 C9 CA 06 00
  18F4: C8 0A 00 00 8D 7E F6 16 57 C4 7E 06 06 57 26 C4
  1904: 3D 26 FF 5D 34 83 C4 04 8B 46 FC 89 46 FE 8B 46
  1914: FE C9 CA 04 00 55 89 E5 C4 7E 06 06 57 9A A9 18
  1924: 39 19 50 FF 76 0A 9A 6E 06 67 19 52 50 C4 7E 06
  1934: 06 57 9A EB 1B 93 19 C9 CA 06 00 C8 0C 02 00 C4
  1944: 7E 06 26 8B 45 1A 26 0B 45 1C 75 53 8D BE F4 FD

CALL 3: segmento=76 offset=0x18A9 archivo=0x01D369
  rango segmento: 0x1889..0x1909
  1889: C4 7E 06 26 8B 45 22 C4 7E 0A 26 89 45 04 C4 7E
  1899: 06 26 8B 45 24 C4 7E 0A 26 89 45 06 C9 CA 04 00
  18A9: C8 0A 00 00 8D 7E F6 16 57 C4 7E 06 06 57 26 C4
  18B9: 3D 26 FF 5D 34 83 C4 04 8B 46 FA 89 46 FE 8B 46
  18C9: FE C9 CA 04 00 55 89 E5 FF 76 0A C4 7E 06 06 57
  18D9: 9A F4 18 EE 18 50 9A 6E 06 2D 19 52 50 C4 7E 06
  18E9: 06 57 9A EB 1B 24 19 C9 CA 06 00 C8 0A 00 00 8D
  18F9: 7E F6 16 57 C4 7E 06 06 57 26 C4 3D 26 FF 5D 34

Guardado: /data/data/com.termux/files/home/armon_code/destinos/segmento_76.bin (28339 bytes)
Guardado: /data/data/com.termux/files/home/armon_code/destinos/segmento_80.bin (9754 bytes)

======================================================================
 FIN
======================================================================
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

La salida cambia bastante nuestra interpretación:


- **76:18A9** comienza con `C8 0A 00 00` → `ENTER 0x000A,0`, por lo que **sí tiene aspecto de entrada de procedimiento**.
- **76:18F4** comienza igualmente con `ENTER 0x000A,0`, así que también parece una rutina.
- **80:0444** es diferente: empieza en `05 00 04...` y no tiene prólogo evidente. Podría ser una entrada interna, un bloque alcanzado por salto o código generado de otra manera. No conviene clasificarlo todavía.


Además, hay una pista muy interesante: las rutinas de 76 se llaman entre sí. En `76:18D7` aparece:


```
9A F4 18 EE 18
```


y en `76:18E7`:


```
9A EB 1B 24 19
```


Es decir, **el segmento 76 contiene una red de procedimientos relacionados**, no simples datos.


El siguiente paso correcto es desensamblar **las tres zonas**, incluyendo suficiente contexto para detectar dónde empiezan y terminan realmente las rutinas. Ejecuta este único bloque:


```
Bashcd /storage/emulated/0/Download/armon

python3 - <<'PY'
from pathlib import Path
import subprocess
import tempfile

base = Path.home() / "armon_code" / "destinos"

targets = [
    ("SEG76_18A9", base / "segmento_76.bin", 0x18A9, 0x18F4),
    ("SEG76_18F4", base / "segmento_76.bin", 0x18F4, 0x1944),
    ("SEG80_0444", base / "segmento_80.bin", 0x0400, 0x04A0),
]

for nombre, archivo, inicio, fin in targets:

    data = archivo.read_bytes()
    bloque = data[inicio:fin]

    tmp = Path(tempfile.mktemp(suffix=".bin"))
    tmp.write_bytes(bloque)

    print()
    print("=" * 80)
    print(f"{nombre}")
    print(f"segmento : {archivo.name}")
    print(f"rango    : 0x{inicio:04X} - 0x{fin:04X}")
    print("=" * 80)

    # llvm-mc puede recibir los bytes directamente como lista hexadecimal.
    hexbytes = " ".join(f"0x{x:02X}" for x in bloque)

    try:
        r = subprocess.run(
            [
                "llvm-mc",
                "--disassemble",
                "--triple=i386-unknown-none-code16",
                "--x86-asm-syntax=intel",
            ],
            input=hexbytes + "\n",
            text=True,
            capture_output=True
        )

        print(r.stdout)

        if r.stderr:
            print("STDERR:")
            print(r.stderr)

    except Exception as e:
        print("ERROR:", e)

    tmp.unlink(missing_ok=True)

print()
print("=" * 80)
print("FIN")
print("=" * 80)
PY
```


Aquí nos interesa especialmente determinar si `76:18A9` y `76:18F4` tienen una estructura del tipo:


```
ENTER
...
CALL
...
LEAVE / RETF
```


y, sobre todo, **qué hacen con los parámetros que reciben**.


Hay una observación adicional muy prometedora: ambas rutinas terminan con:


```
C9 CA 04 00
```


que corresponde a:


```
asmleave
retf 4
```


Por tanto, **cada una recibe exactamente un parámetro FAR de 4 bytes**.


Eso ya nos permite empezar a reconstruir sus firmas conceptuales, aunque todavía no sepamos los tipos Delphi:


```
Rutina 76:18A9
    parámetro FAR de 4 bytes
    ...
    retf 4

Rutina 76:18F4
    parámetro FAR de 4 bytes
    ...
    retf 4
```


Ese detalle puede ser muy útil para identificar si son procedimientos que reciben un objeto, un puntero, una cadena, una estructura o algún otro tipo Delphi.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> from pathlib import Path
> import subprocess
> import tempfile
>
> base = Path.home() / "armon_code" / "destinos"
>
> targets = [
>     ("SEG76_18A9", base / "segmento_76.bin", 0x18A9, 0x18F4),
>     ("SEG76_18F4", base / "segmento_76.bin", 0x18F4, 0x1944),
>     ("SEG80_0444", base / "segmento_80.bin", 0x0400, 0x04A0),
> ]
>
> for nombre, archivo, inicio, fin in targets:
>
>     data = archivo.read_bytes()
>     bloque = data[inicio:fin]
>
>     tmp = Path(tempfile.mktemp(suffix=".bin"))
>     tmp.write_bytes(bloque)
>
>     print()
>     print("=" * 80)
>     print(f"{nombre}")
>     print(f"segmento : {archivo.name}")
>     print(f"rango    : 0x{inicio:04X} - 0x{fin:04X}")
>     print("=" * 80)
>
>     # llvm-mc puede recibir los bytes directamente como lista hexadecimal.
>     hexbytes = " ".join(f"0x{x:02X}" for x in bloque)
>
>     try:
>         r = subprocess.run(
>             [
>                 "llvm-mc",
>                 "--disassemble",
>                 "--triple=i386-unknown-none-code16",
>                 "--x86-asm-syntax=intel",
>             ],
>             input=hexbytes + "\n",
>             text=True,
>             capture_output=True
>         )
>
>         print(r.stdout)
>
>         if r.stderr:
>             print("STDERR:")
>             print(r.stderr)
>
>     except Exception as e:
>         print("ERROR:", e)
>
>     tmp.unlink(missing_ok=True)
>
> print()
> print("=" * 80)
> print("FIN")
> print("=" * 80)
> PY

================================================================================
SEG76_18A9
segmento : segmento_76.bin
rango    : 0x18A9 - 0x18F4
================================================================================
        enter   $10, $0
        leaw    -10(%bp), %di
        pushw   %ss
        pushw   %di
        lesw    6(%bp), %di
        pushw   %es
        pushw   %di
        lesw    %es:(%di), %di
        lcallw  *%es:52(%di)
        addw    $4, %sp
        movw    -6(%bp), %ax
        movw    %ax, -2(%bp)
        movw    -2(%bp), %ax
        leave
        lretw   $4
        pushw   %bp
        movw    %sp, %bp
        pushw   10(%bp)
        lesw    6(%bp), %di
        pushw   %es
        pushw   %di
        lcallw  $6382, $6388                    # imm = 0x18EE
                                        # imm = 0x18F4
        pushw   %ax
        lcallw  $6445, $1646                    # imm = 0x192D
                                        # imm = 0x66E
        pushw   %dx
        pushw   %ax
        lesw    6(%bp), %di
        pushw   %es
        pushw   %di
        lcallw  $6436, $7147                    # imm = 0x1924
                                        # imm = 0x1BEB
        leave
        lretw   $6


================================================================================
SEG76_18F4
segmento : segmento_76.bin
rango    : 0x18F4 - 0x1944
================================================================================
        enter   $10, $0
        leaw    -10(%bp), %di
        pushw   %ss
        pushw   %di
        lesw    6(%bp), %di
        pushw   %es
        pushw   %di
        lesw    %es:(%di), %di
        lcallw  *%es:52(%di)
        addw    $4, %sp
        movw    -4(%bp), %ax
        movw    %ax, -2(%bp)
        movw    -2(%bp), %ax
        leave
        lretw   $4
        pushw   %bp
        movw    %sp, %bp
        lesw    6(%bp), %di
        pushw   %es
        pushw   %di
        lcallw  $6457, $6313                    # imm = 0x1939
                                        # imm = 0x18A9
        pushw   %ax
        pushw   10(%bp)
        lcallw  $6503, $1646                    # imm = 0x1967
                                        # imm = 0x66E
        pushw   %dx
        pushw   %ax
        lesw    6(%bp), %di
        pushw   %es
        pushw   %di
        lcallw  $6547, $7147                    # imm = 0x1993
                                        # imm = 0x1BEB
        leave
        lretw   $6
        enter   $524, $0                        # imm = 0x20C

STDERR:
<stdin>:1:396: warning: invalid instruction encoding
0xC8 0x0A 0x00 0x00 0x8D 0x7E 0xF6 0x16 0x57 0xC4 0x7E 0x06 0x06 0x57 0x26 0xC4 0x3D 0x26 0xFF 0x5D 0x34 0x83 0xC4 0x04 0x8B 0x46 0xFC 0x89 0x46 0xFE 0x8B 0x46 0xFE 0xC9 0xCA 0x04 0x00 0x55 0x89 0xE5 0xC4 0x7E 0x06 0x06 0x57 0x9A 0xA9 0x18 0x39 0x19 0x50 0xFF 0x76 0x0A 0x9A 0x6E 0x06 0x67 0x19 0x52 0x50 0xC4 0x7E 0x06 0x06 0x57 0x9A 0xEB 0x1B 0x93 0x19 0xC9 0xCA 0x06 0x00 0xC8 0x0C 0x02 0x00 0xC4
                                                                                                                                                                                                                                                                                                                                                                                                           ^


================================================================================
SEG80_0444
segmento : segmento_80.bin
rango    : 0x0400 - 0x04A0
================================================================================
        movw    $49203, %sp                     # imm = 0xC033
        xchgw   %ax, 9504
        lretw
        cmpw    $0, 9504
        jne     1
        lretw
        movw    $0, %ax
        jmp     -950
        movw    %sp, %si
        movw    %ss:2(%si), %es
        cmpw    %es:2(%di), %dx
        jg      7
        jl      20
        cmpw    %es:(%di), %ax
        jb      15
        cmpw    %es:6(%di), %dx
        jl      8
        jg      7
        cmpw    %es:4(%di), %ax
        ja      1
        lretw
        movw    $4, %ax
        jmp     -990
        movw    $5, %ax
        jmp     -996
        addw    $1024, %ax                      # imm = 0x400
        jb      25
        subw    %sp, %ax
        jae     21
        negw    %ax
        cmpw    %ss:10, %ax
        jb      12
        cmpw    %ss:12, %ax
        jae     4
        movw    %ax, %ss:12
        lretw
        movw    $202, %ax
        jmp     -985
        movb    $1, 9540
        cmpb    $-51, %cs:1199
        je      61
        pushw   %bp
        movw    %sp, %bp
        subw    $10, %sp
        pushw   %ax
        fstpt   -10(%bp)
        fldl    9604
        wait
        fdivl   9612
        wait
        fmull   9612
        wait
        fsubrl  9604
        wait
        fcompl  9620
        wait
        wait
        fnstsw  %ax
        wait
        andw    $256, %ax                       # imm = 0x100

STDERR:
<stdin>:1:796: warning: invalid instruction encoding
0xBC 0x33 0xC0 0x87 0x06 0x20 0x25 0xCB 0x83 0x3E 0x20 0x25 0x00 0x75 0x01 0xCB 0xB8 0x00 0x00 0xE9 0x4A 0xFC 0x8B 0xF4 0x36 0x8E 0x44 0x02 0x26 0x3B 0x55 0x02 0x7F 0x07 0x7C 0x14 0x26 0x3B 0x05 0x72 0x0F 0x26 0x3B 0x55 0x06 0x7C 0x08 0x7F 0x07 0x26 0x3B 0x45 0x04 0x77 0x01 0xCB 0xB8 0x04 0x00 0xE9 0x22 0xFC 0xB8 0x05 0x00 0xE9 0x1C 0xFC 0x05 0x00 0x04 0x72 0x19 0x2B 0xC4 0x73 0x15 0xF7 0xD8 0x36 0x3B 0x06 0x0A 0x00 0x72 0x0C 0x36 0x3B 0x06 0x0C 0x00 0x73 0x04 0x36 0xA3 0x0C 0x00 0xCB 0xB8 0xCA 0x00 0xE9 0x27 0xFC 0xC6 0x06 0x44 0x25 0x01 0x2E 0x80 0x3E 0xAF 0x04 0xCD 0x74 0x3D 0x55 0x8B 0xEC 0x83 0xEC 0x0A 0x50 0xDB 0x7E 0xF6 0xDD 0x06 0x84 0x25 0x9B 0xDC 0x36 0x8C 0x25 0x9B 0xDC 0x0E 0x8C 0x25 0x9B 0xDC 0x2E 0x84 0x25 0x9B 0xDC 0x1E 0x94 0x25 0x9B 0x9B 0xDF 0xE0 0x9B 0x25 0x00 0x01 0xC1
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           ^


================================================================================
FIN
================================================================================
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Perfecto. Aquí aparece una pista mucho más fuerte que simplemente “son procedimientos”.


### 1. `76:18A9` y `76:18F4` son rutinas reales


Ambas tienen exactamente esta estructura:


```
asmENTER 10,0
...
LEAVE
RETF 4
```


Por tanto:


- reservan 10 bytes de variables locales;
- reciben **un único parámetro FAR de 4 bytes**;
- devuelven un valor en `DX:AX`;
- son rutinas independientes.


Además, ambas tienen prácticamente la misma estructura.


### 2. Las dos hacen una llamada virtual


La parte crucial de `76:18A9` es:


```
asmlesw    6(%bp), %di
pushw   %es
pushw   %di

lesw    %es:(%di), %di

lcallw  *%es:52(%di)
```


El flujo conceptual es:


```
parámetro FAR
     ↓
puntero a objeto/estructura
     ↓
obtiene una referencia situada al comienzo del objeto
     ↓
lee una dirección de su tabla
     ↓
llama indirectamente mediante esa tabla
```


Y la llamada se realiza en:


```
offset 0x34 = 52 decimal
```


Eso tiene una apariencia muy característica de **despacho mediante una tabla de métodos**, compatible con la forma en que Delphi implementa llamadas virtuales en objetos.


### 3. Las dos rutinas son casi gemelas


Podemos representarlas así:


```
76:18A9
    parámetro FAR
       │
       ├── obtiene objeto
       │
       ├── llama método en +34h
       │
       └── devuelve DX:AX

76:18F4
    parámetro FAR
       │
       ├── obtiene objeto
       │
       ├── llama método en +34h
       │
       └── devuelve DX:AX
```


Pero hay una diferencia fundamental.


En `18A9`:


```
asmmovw -6(%bp), %ax
movw %ax, -2(%bp)
movw -2(%bp), %ax
```


Mientras que en `18F4`:


```
asmmovw -4(%bp), %ax
movw %ax, -2(%bp)
movw -2(%bp), %ax
```


Es decir, **cada una extrae una palabra diferente de sus variables locales después de la llamada virtual**.


Eso sugiere que probablemente no son dos algoritmos matemáticos independientes, sino **wrappers/getters relacionados con dos propiedades o dos métodos de un objeto**.


### 4. Y esto explica algo de `CalcularClick`


Recordemos:


```
asmles di,[bp+6]
push es
push di
call 76:18A9
cwd
mov [5158],ax
mov [515A],dx
```


y antes:


```
asmles di,[bp+6]
push es
push di
call 76:18F4
cwd
mov [515C],ax
mov [515E],dx
```


Por tanto `CalcularClick` está haciendo:


```
objeto
  │
  ├── rutina 76:18F4 → resultado 32 bits → 515C:515E
  │
  └── rutina 76:18A9 → resultado 32 bits → 5158:515A
```


Tenemos ya una hipótesis estructural bastante sólida:


```
                     CalcularClick
                           │
                     objeto/registro
                           │
              ┌────────────┴────────────┐
              ↓                         ↓
        76:18F4                    76:18A9
              │                         │
        método virtual              método virtual
           +34h                        +34h
              │                         │
          resultado                 resultado
           32 bits                   32 bits
              │                         │
          515C:515E                5158:515A
```


### 5. Hay una corrección importante sobre `80:0444`


Ahora podemos descartar la interpretación inicial de que `80:0444` era necesariamente el comienzo de una rutina.


Sus primeros bytes:


```
asm05 00 04
72 19
2B C4
73 15
...
CB
```


son perfectamente plausibles como **código situado en medio de una rutina**, pero no hay un prólogo Delphi allí.


Y además vemos saltos hacia atrás:


```
asmE9 22 FC
E9 1C FC
E9 27 FC
```


Por lo que `80:0444` parece pertenecer a una **rutina mayor que comenzó antes de 0444**.


Eso es importante: **no debemos desensamblar `80:0444` como si fuera una función independiente**.


---


## El siguiente objetivo


Ahora necesitamos averiguar **qué son exactamente las rutinas virtuales llamadas mediante `+34h`**.


La manera más fiable es localizar todas las referencias a:


```
76:18A9
76:18F4
```


y, sobre todo, reconstruir el objeto que `CalcularClick` les está pasando.


También debemos identificar el procedimiento que contiene `80:0444`, buscando hacia atrás hasta encontrar su verdadero inicio.


El siguiente análisis debería hacer tres cosas simultáneamente:


```
1. localizar todos los CALL → 76:18A9
2. localizar todos los CALL → 76:18F4
3. buscar hacia atrás desde 80:0444
   hasta el comienzo real de su procedimiento
```


Eso nos permitirá pasar de **desensamblado** a **reconstrucción de la estructura Delphi original**, que es justamente lo que buscamos.

## Usuario

Continuemos

## Asistente

Ejecuta este bloque único. Hace tres búsquedas:


1. `CALL FAR` cuyo offset sea `18A9`.
2. `CALL FAR` cuyo offset sea `18F4`.
3. posibles comienzos de procedimiento antes de `80:0444`.


```
Bashcd /storage/emulated/0/Download/armon

python3 - <<'PY'
from pathlib import Path
import struct

exe = Path("ARMON.EXE")
data = exe.read_bytes()

NE = 0x100
SEG_COUNT = struct.unpack_from("<H", data, NE + 0x1C)[0]
SEG_TABLE = NE + struct.unpack_from("<H", data, NE + 0x22)[0]
SHIFT = struct.unpack_from("<H", data, NE + 0x32)[0]

segments = {}

for n in range(1, SEG_COUNT + 1):
    p = SEG_TABLE + (n - 1) * 8
    sector = struct.unpack_from("<H", data, p)[0]
    length = struct.unpack_from("<H", data, p + 2)[0]
    flags = struct.unpack_from("<H", data, p + 4)[0]

    size = 0x10000 if length == 0 else length
    fileoff = sector << SHIFT

    segments[n] = {
        "fileoff": fileoff,
        "size": size,
        "flags": flags
    }

# ============================================================
# 1. BUSCAR CALL FAR CON OFFSETS 18A9 Y 18F4
# ============================================================

print("=" * 80)
print(" REFERENCIAS A 76:18A9 Y 76:18F4")
print("=" * 80)

targets = {
    bytes([0x9A, 0xA9, 0x18]): "18A9",
    bytes([0x9A, 0xF4, 0x18]): "18F4",
}

found = []

for seg, s in segments.items():

    start = s["fileoff"]
    blob = data[start:start + s["size"]]

    for pattern, target in targets.items():

        pos = 0

        while True:
            pos = blob.find(pattern, pos)

            if pos < 0:
                break

            # 5 bytes de una instrucción CALL FAR:
            # 9A offset_lo offset_hi segment_lo segment_hi
            raw = blob[pos:pos + 5]

            segword = (
                raw[3] |
                (raw[4] << 8)
            ) if len(raw) >= 5 else None

            print()
            print(
                f"CALL -> offset {target}: "
                f"segmento fuente {seg}:"
                f"0x{pos:04X}"
            )

            print(
                "  archivo      = "
                f"0x{start + pos:06X}"
            )

            print(
                "  bytes        = "
                + " ".join(f"{x:02X}" for x in raw)
            )

            if segword is not None:
                print(
                    f"  palabra segmento bruta = "
                    f"0x{segword:04X}"
                )

            found.append((seg, pos, target))

            pos += 1

print()
print(
    f"Total referencias encontradas: {len(found)}"
)

# ============================================================
# 2. BUSCAR PRÓLOGOS CERCA DE 80:0444
# ============================================================

print()
print("=" * 80)
print(" POSIBLES COMIENZOS DE PROCEDIMIENTO CERCA DE 80:0444")
print("=" * 80)

seg = 80
s = segments[seg]
blob = data[s["fileoff"]:s["fileoff"] + s["size"]]

objetivo = 0x0444

# Patrones típicos encontrados en este ejecutable:
# 55 89 E5 = PUSH BP / MOV BP,SP
# 55 8B EC = PUSH BP / MOV BP,SP
# C8 xx xx 00 = ENTER
patterns = [
    (bytes.fromhex("55 89 E5"), "55 89 E5"),
    (bytes.fromhex("55 8B EC"), "55 8B EC"),
]

candidatos = []

inicio = max(0, objetivo - 0x400)
fin = objetivo

for p in range(inicio, fin):

    for pattern, nombre in patterns:

        if blob[p:p + len(pattern)] == pattern:
            candidatos.append((p, nombre))

    # ENTER imm16, imm8
    if (
        p + 3 < len(blob)
        and blob[p] == 0xC8
        and blob[p + 3] == 0x00
    ):
        candidatos.append(
            (p, "ENTER")
        )

for p, nombre in candidatos:

    distancia = objetivo - p

    print(
        f"candidato 80:{p:04X} "
        f"({nombre}) "
        f"distancia=+0x{distancia:X}"
    )

print()
print(
    "Número de candidatos:",
    len(candidatos)
)

# ============================================================
# 3. MOSTRAR CONTEXTO DE CADA CANDIDATO
# ============================================================

print()
print("=" * 80)
print(" CONTEXTO DE LOS CANDIDATOS")
print("=" * 80)

for p, nombre in candidatos:

    print()
    print(
        f"--- 80:{p:04X} ({nombre}) ---"
    )

    a = p
    b = min(len(blob), p + 0x80)

    for q in range(a, b, 16):

        row = blob[q:min(q + 16, b)]

        print(
            f"{q:04X}: "
            + " ".join(f"{x:02X}" for x in row)
        )

print()
print("=" * 80)
print(" FIN")
print("=" * 80)
PY
```


### Qué esperamos encontrar


La primera sección es la más importante. Si aparece, por ejemplo:


```
CALL -> offset 18A9: segmento fuente 3:0x....
CALL -> offset 18F4: segmento fuente 3:0x....
```


podremos relacionar directamente esas llamadas con `CalcularClick`.


Y si aparecen otras fuentes, podremos construir un pequeño **grafo de llamadas**:


```
CalcularClick
      │
      ├── 76:18A9
      │
      └── 76:18F4
             ↑
             │
       otros procedimientos
```


La segunda sección nos permitirá dejar de tratar `80:0444` aisladamente y encontrar el **inicio verdadero de la rutina que lo contiene**.


Hay una precaución importante: una coincidencia `9A A9 18` o `9A F4 18` demuestra el **offset del CALL**, pero todavía debemos resolver su palabra de segmento mediante la relocación NE. Por eso la salida de este paso no debe interpretarse aún como una dirección completa hasta cruzarla con la tabla de relocaciones.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
xe = Path("ARMON.EXE")
data = exe.read_bytes()

NE = 0x100
SEG_COUNT = struct.unpack_from("<H", data, NE + 0x1C)[0]
SEG_TABLE = NE + struct.unpack_from("<H", data, NE + 0x22)[0]
SHIFT = struct.unpack_from("<H", data, NE + 0x32)[0]

segments = {}

for n in range(1, SEG_COUNT + 1):
    p = SEG_TABLE + (n - 1) * 8
    sector = struct.unpack_from("<H", data, p)[0]
    length = struct.unpack_from("<H", data, p + 2)[0]
    flags = struct.unpack_from("<H", data, p + 4)[0]

    size = 0x10.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> from pathlib import Path
> import struct
>
> exe = Path("ARMON.EXE")
> data = exe.read_bytes()
>
> NE = 0x100
> SEG_COUNT = struct.unpack_from("<H", data, NE + 0x1C)[0]
> SEG_TABLE = NE + struct.unpack_from("<H", data, NE + 0x22)[0]
> SHIFT = struct.unpack_from("<H", data, NE + 0x32)[0]
>
> segments = {}
>
> for n in range(1, SEG_COUNT + 1):
>     p = SEG_TABLE + (n - 1) * 8
>     sector = struct.unpack_from("<H", data, p)[0]
>     length = struct.unpack_from("<H", data, p + 2)[0]
>     flags = struct.unpack_from("<H", data, p + 4)[0]
>
>     size = 0x10000 if length == 0 else length
>     fileoff = sector << SHIFT
>
>     segments[n] = {
>         "fileoff": fileoff,
>         "size": size,
>         "flags": flags
>     }
None

            print()
         >
   print(
     > # ============================================================
> # 1. BUSCAR CALL FAR CON OFFSETS 18A9 Y 18F4
          f> # ============================================================
>
> print("=" * 80)
> print(" REFERENCIAS A 76:18A9 Y 76:18F4")
> print("=" * 80)
>
> targets = {
>     bytes([0x9A, 0xA9, 0x18]): "18A9",
>     bytes([0x9A, 0xF4, 0x18]): "18F4",
> }
>
> found = []
>
> for seg, s in segments.items():
>
a se>     start = s["fileoff"]
>     blob = data[start:start + s["size"]]
(seg, pos, ta>
>     for pattern, target in targets.items():
>
d)}"
)
>         pos = 0
>
>         while True:
>             pos = blob.find(pattern, pos)
>
>             if pos < 0:
>                 break
>
>             # 5 bytes de una instrucción CALL FAR:
>             # 9A offset_lo offset_hi segment_lo segment_hi
>             raw = blob[pos:pos + 5]
>
>             segword = (
>                 raw[3] |
>                 (raw[4] << 8)
>             ) if len(raw) >= 5 else None
>


  >             print()
>             print(
>                 f"CALL -> offset {target}: "
 == 0xC8
    >                 f"segmento fuente {seg}:"
>                 f"0x{pos:04X}"
>             )
>
>             print(
>                 "  archivo      = "
>                 f"0x{start + pos:06X}"
>             )
>
>             print(
>                 "  bytes        = "
>                 + " ".join(f"{x:02X}" for x in raw)
>             )
>
>             if segword is not None:
>                 print(
>                     f"  palabra segmento bruta = "
>                     f"0x{segword:04X}"
>                 )
>
>             found.append((seg, pos, target))
>
>             pos += 1
>
> print()
> print(
>     f"Total referencias encontradas: {len(found)}"
> )
>
> # ============================================================
> # 2. BUSCAR PRÓLOGOS CERCA DE 80:0444
> # ============================================================
>
> print()
> print("=" * 80)
> print(" POSIBLES COMIENZOS DE PROCEDIMIENTO CERCA DE 80:0444")
> print("=" * 80)
>
> seg = 80
> s = segments[seg]
> blob = data[s["fileoff"]:s["fileoff"] + s["size"]]
>
> objetivo = 0x0444
>
> # Patrones típicos encontrados en este ejecutable:
> # 55 89 E5 = PUSH BP / MOV BP,SP
> # 55 8B EC = PUSH BP / MOV BP,SP
> # C8 xx xx 00 = ENTER
> patterns = [
>     (bytes.fromhex("55 89 E5"), "55 89 E5"),
>     (bytes.fromhex("55 8B EC"), "55 8B EC"),
> ]
>
> candidatos = []
>
> inicio = max(0, objetivo - 0x400)
> fin = objetivo
>
> for p in range(inicio, fin):
>
>     for pattern, nombre in patterns:
>
>         if blob[p:p + len(pattern)] == pattern:
>             candidatos.append((p, nombre))
>
>     # ENTER imm16, imm8
>     if (
>         p + 3 < len(blob)
>         and blob[p] == 0xC8
>         and blob[p + 3] == 0x00
>     ):
>         candidatos.append(
>             (p, "ENTER")
>         )
>
> for p, nombre in candidatos:
>
>     distancia = objetivo - p
>
>     print(
>         f"candidato 80:{p:04X} "
>         f"({nombre}) "
>         f"distancia=+0x{distancia:X}"
>     )
>
> print()
> print(
>     "Número de candidatos:",
>     len(candidatos)
> )
>
> # ============================================================
> # 3. MOSTRAR CONTEXTO DE CADA CANDIDATO
> # ============================================================
>
> print()
> print("=" * 80)
> print(" CONTEXTO DE LOS CANDIDATOS")
> print("=" * 80)
>
> for p, nombre in candidatos:
>
>     print()
>     print(
>         f"--- 80:{p:04X} ({nombre}) ---"
>     )
>
>     a = p
>     b = min(len(blob), p + 0x80)
>
>     for q in range(a, b, 16):
>
>         row = blob[q:min(q + 16, b)]
>
>         print(
>             f"{q:04X}: "
>             + " ".join(f"{x:02X}" for x in row)
>         )
>
> print()
> print("=" * 80)
> print(" FIN")
> print("=" * 80)
> PY
================================================================================
 REFERENCIAS A 76:18A9 Y 76:18F4
================================================================================

CALL -> offset 18A9: segmento fuente 2:0x978F
  archivo      = 0x0461CF
  bytes        = 9A A9 18 DC 97
  palabra segmento bruta = 0x97DC

CALL -> offset 18A9: segmento fuente 2:0x9B0C
  archivo      = 0x04654C
  bytes        = 9A A9 18 3F 9B
  palabra segmento bruta = 0x9B3F

CALL -> offset 18A9: segmento fuente 2:0x9BA1
  archivo      = 0x0465E1
  bytes        = 9A A9 18 C3 9B
  palabra segmento bruta = 0x9BC3

CALL -> offset 18A9: segmento fuente 2:0x9BC0
  archivo      = 0x046600
  bytes        = 9A A9 18 D6 9B
  palabra segmento bruta = 0x9BD6

CALL -> offset 18F4: segmento fuente 2:0x97D9
  archivo      = 0x046219
  bytes        = 9A F4 18 0F 9B
  palabra segmento bruta = 0x9B0F

CALL -> offset 18F4: segmento fuente 2:0x9B3C
  archivo      = 0x04657C
  bytes        = 9A F4 18 A4 9B
  palabra segmento bruta = 0x9BA4

CALL -> offset 18F4: segmento fuente 2:0x9C20
  archivo      = 0x046660
  bytes        = 9A F4 18 42 9C
  palabra segmento bruta = 0x9C42

CALL -> offset 18F4: segmento fuente 2:0x9C3F
  archivo      = 0x04667F
  bytes        = 9A F4 18 56 9C
  palabra segmento bruta = 0x9C56

CALL -> offset 18A9: segmento fuente 3:0x1CD9
  archivo      = 0x04CF99
  bytes        = 9A A9 18 14 1D
  palabra segmento bruta = 0x1D14

CALL -> offset 18A9: segmento fuente 3:0x1DE8
  archivo      = 0x04D0A8
  bytes        = 9A A9 18 AD 1F
  palabra segmento bruta = 0x1FAD

CALL -> offset 18A9: segmento fuente 3:0x2589
  archivo      = 0x04D849
  bytes        = 9A A9 18 0C 27
  palabra segmento bruta = 0x270C

CALL -> offset 18F4: segmento fuente 3:0x1CC4
  archivo      = 0x04CF84
  bytes        = 9A F4 18 DC 1C
  palabra segmento bruta = 0x1CDC

CALL -> offset 18F4: segmento fuente 3:0x1DD6
  archivo      = 0x04D096
  bytes        = 9A F4 18 EB 1D
  palabra segmento bruta = 0x1DEB

CALL -> offset 18F4: segmento fuente 3:0x2577
  archivo      = 0x04D837
  bytes        = 9A F4 18 8C 25
  palabra segmento bruta = 0x258C

CALL -> offset 18A9: segmento fuente 4:0x039D
  archivo      = 0x052CDD
  bytes        = 9A A9 18 AD 03
  palabra segmento bruta = 0x03AD

CALL -> offset 18F4: segmento fuente 4:0x03AA
  archivo      = 0x052CEA
  bytes        = 9A F4 18 C7 03
  palabra segmento bruta = 0x03C7

CALL -> offset 18A9: segmento fuente 6:0x32C4
  archivo      = 0x066E84
  bytes        = 9A A9 18 00 33
  palabra segmento bruta = 0x3300

CALL -> offset 18F4: segmento fuente 6:0x32B3
  archivo      = 0x066E73
  bytes        = 9A F4 18 C7 32
  palabra segmento bruta = 0x32C7

CALL -> offset 18A9: segmento fuente 62:0x05B0
  archivo      = 0x2C3730
  bytes        = 9A A9 18 DC 05
  palabra segmento bruta = 0x05DC

CALL -> offset 18F4: segmento fuente 62:0x05A3
  archivo      = 0x2C3723
  bytes        = 9A F4 18 B3 05
  palabra segmento bruta = 0x05B3

CALL -> offset 18A9: segmento fuente 67:0x317D
  archivo      = 0x2DF4BD
  bytes        = 9A A9 18 A5 31
  palabra segmento bruta = 0x31A5

CALL -> offset 18F4: segmento fuente 67:0x3170
  archivo      = 0x2DF4B0
  bytes        = 9A F4 18 80 31
  palabra segmento bruta = 0x3180

CALL -> offset 18A9: segmento fuente 68:0x345C
  archivo      = 0x2E4EDC
  bytes        = 9A A9 18 6A 34
  palabra segmento bruta = 0x346A

CALL -> offset 18A9: segmento fuente 68:0x3895
  archivo      = 0x2E5315
  bytes        = 9A A9 18 CD 3D
  palabra segmento bruta = 0x3DCD

CALL -> offset 18A9: segmento fuente 68:0x41B9
  archivo      = 0x2E5C39
  bytes        = 9A A9 18 3C 42
  palabra segmento bruta = 0x423C

CALL -> offset 18A9: segmento fuente 68:0x4C17
  archivo      = 0x2E6697
  bytes        = 9A A9 18 2D 4C
  palabra segmento bruta = 0x4C2D

CALL -> offset 18A9: segmento fuente 68:0x4C2A
  archivo      = 0x2E66AA
  bytes        = 9A A9 18 66 4C
  palabra segmento bruta = 0x4C66

CALL -> offset 18A9: segmento fuente 68:0x4C63
  archivo      = 0x2E66E3
  bytes        = 9A A9 18 9A 4C
  palabra segmento bruta = 0x4C9A

CALL -> offset 18A9: segmento fuente 68:0x4C97
  archivo      = 0x2E6717
  bytes        = 9A A9 18 C3 4C
  palabra segmento bruta = 0x4CC3

CALL -> offset 18A9: segmento fuente 68:0x4CC0
  archivo      = 0x2E6740
  bytes        = 9A A9 18 71 4D
  palabra segmento bruta = 0x4D71

CALL -> offset 18A9: segmento fuente 68:0x59E2
  archivo      = 0x2E7462
  bytes        = 9A A9 18 17 5A
  palabra segmento bruta = 0x5A17

CALL -> offset 18A9: segmento fuente 68:0x5BA8
  archivo      = 0x2E7628
  bytes        = 9A A9 18 B8 5B
  palabra segmento bruta = 0x5BB8

CALL -> offset 18A9: segmento fuente 68:0x5C37
  archivo      = 0x2E76B7
  bytes        = 9A A9 18 49 5C
  palabra segmento bruta = 0x5C49

CALL -> offset 18A9: segmento fuente 68:0x5C46
  archivo      = 0x2E76C6
  bytes        = 9A A9 18 ED 5C
  palabra segmento bruta = 0x5CED

CALL -> offset 18A9: segmento fuente 68:0x68FC
  archivo      = 0x2E837C
  bytes        = 9A A9 18 0E 69
  palabra segmento bruta = 0x690E

CALL -> offset 18A9: segmento fuente 68:0x690B
  archivo      = 0x2E838B
  bytes        = 9A A9 18 27 69
  palabra segmento bruta = 0x6927

CALL -> offset 18F4: segmento fuente 68:0x3467
  archivo      = 0x2E4EE7
  bytes        = 9A F4 18 87 38
  palabra segmento bruta = 0x3887

CALL -> offset 18F4: segmento fuente 68:0x3884
  archivo      = 0x2E5304
  bytes        = 9A F4 18 98 38
  palabra segmento bruta = 0x3898

CALL -> offset 18F4: segmento fuente 68:0x4239
  archivo      = 0x2E5CB9
  bytes        = 9A F4 18 B5 48
  palabra segmento bruta = 0x48B5

CALL -> offset 18F4: segmento fuente 68:0x5BB5
  archivo      = 0x2E7635
  bytes        = 9A F4 18 29 5C
  palabra segmento bruta = 0x5C29

CALL -> offset 18F4: segmento fuente 68:0x5C26
  archivo      = 0x2E76A6
  bytes        = 9A F4 18 3A 5C
  palabra segmento bruta = 0x5C3A

CALL -> offset 18F4: segmento fuente 68:0x6924
  archivo      = 0x2E83A4
  bytes        = 9A F4 18 36 69
  palabra segmento bruta = 0x6936

CALL -> offset 18F4: segmento fuente 68:0x6933
  archivo      = 0x2E83B3
  bytes        = 9A F4 18 AE 69
  palabra segmento bruta = 0x69AE

CALL -> offset 18A9: segmento fuente 70:0x3115
  archivo      = 0x003A15
  bytes        = 9A A9 18 59 31
  palabra segmento bruta = 0x3159

CALL -> offset 18A9: segmento fuente 70:0x34B8
  archivo      = 0x003DB8
  bytes        = 9A A9 18 C9 34
  palabra segmento bruta = 0x34C9

CALL -> offset 18A9: segmento fuente 70:0x35CD
  archivo      = 0x003ECD
  bytes        = 9A A9 18 E4 35
  palabra segmento bruta = 0x35E4

CALL -> offset 18A9: segmento fuente 71:0x2201
  archivo      = 0x0065C1
  bytes        = 9A A9 18 19 22
  palabra segmento bruta = 0x2219

CALL -> offset 18A9: segmento fuente 71:0x228F
  archivo      = 0x00664F
  bytes        = 9A A9 18 A7 22
  palabra segmento bruta = 0x22A7

CALL -> offset 18F4: segmento fuente 71:0x2216
  archivo      = 0x0065D6
  bytes        = 9A F4 18 71 22
  palabra segmento bruta = 0x2271

CALL -> offset 18F4: segmento fuente 71:0x22A4
  archivo      = 0x006664
  bytes        = 9A F4 18 30 23
  palabra segmento bruta = 0x2330

CALL -> offset 18A9: segmento fuente 76:0x1921
  archivo      = 0x01D3E1
  bytes        = 9A A9 18 39 19
  palabra segmento bruta = 0x1939

CALL -> offset 18F4: segmento fuente 76:0x18D9
  archivo      = 0x01D399
  bytes        = 9A F4 18 EE 18
  palabra segmento bruta = 0x18EE

CALL -> offset 18A9: segmento fuente 77:0x1BBE
  archivo      = 0x0247BE
  bytes        = 9A A9 18 5B 1D
  palabra segmento bruta = 0x1D5B

CALL -> offset 18A9: segmento fuente 77:0x202A
  archivo      = 0x024C2A
  bytes        = 9A A9 18 3C 20
  palabra segmento bruta = 0x203C

CALL -> offset 18A9: segmento fuente 77:0x2300
  archivo      = 0x024F00
  bytes        = 9A A9 18 12 23
  palabra segmento bruta = 0x2312

CALL -> offset 18A9: segmento fuente 77:0x230F
  archivo      = 0x024F0F
  bytes        = 9A A9 18 28 23
  palabra segmento bruta = 0x2328

CALL -> offset 18A9: segmento fuente 77:0x2325
  archivo      = 0x024F25
  bytes        = 9A A9 18 46 23
  palabra segmento bruta = 0x2346

CALL -> offset 18A9: segmento fuente 77:0x2343
  archivo      = 0x024F43
  bytes        = 9A A9 18 85 23
  palabra segmento bruta = 0x2385

CALL -> offset 18A9: segmento fuente 77:0x2BCF
  archivo      = 0x0257CF
  bytes        = 9A A9 18 3C 2C
  palabra segmento bruta = 0x2C3C

CALL -> offset 18A9: segmento fuente 77:0x3D37
  archivo      = 0x026937
  bytes        = 9A A9 18 45 3D
  palabra segmento bruta = 0x3D45

CALL -> offset 18A9: segmento fuente 77:0x45F2
  archivo      = 0x0271F2
  bytes        = 9A A9 18 0C 46
  palabra segmento bruta = 0x460C

CALL -> offset 18A9: segmento fuente 77:0x51F2
  archivo      = 0x027DF2
  bytes        = 9A A9 18 14 52
  palabra segmento bruta = 0x5214

CALL -> offset 18F4: segmento fuente 77:0x1B9F
  archivo      = 0x02479F
  bytes        = 9A F4 18 C1 1B
  palabra segmento bruta = 0x1BC1

CALL -> offset 18F4: segmento fuente 77:0x2039
  archivo      = 0x024C39
  bytes        = 9A F4 18 73 20
  palabra segmento bruta = 0x2073

CALL -> offset 18F4: segmento fuente 77:0x2382
  archivo      = 0x024F82
  bytes        = 9A F4 18 94 23
  palabra segmento bruta = 0x2394

CALL -> offset 18F4: segmento fuente 77:0x2391
  archivo      = 0x024F91
  bytes        = 9A F4 18 AA 23
  palabra segmento bruta = 0x23AA

CALL -> offset 18F4: segmento fuente 77:0x23A7
  archivo      = 0x024FA7
  bytes        = 9A F4 18 C8 23
  palabra segmento bruta = 0x23C8

CALL -> offset 18F4: segmento fuente 77:0x23C5
  archivo      = 0x024FC5
  bytes        = 9A F4 18 84 24
  palabra segmento bruta = 0x2484

CALL -> offset 18F4: segmento fuente 77:0x2BC2
  archivo      = 0x0257C2
  bytes        = 9A F4 18 D2 2B
  palabra segmento bruta = 0x2BD2

CALL -> offset 18F4: segmento fuente 77:0x3D42
  archivo      = 0x026942
  bytes        = 9A F4 18 50 3D
  palabra segmento bruta = 0x3D50

CALL -> offset 18F4: segmento fuente 77:0x45DB
  archivo      = 0x0271DB
  bytes        = 9A F4 18 F5 45
  palabra segmento bruta = 0x45F5

CALL -> offset 18F4: segmento fuente 77:0x5211
  archivo      = 0x027E11
  bytes        = 9A F4 18 C2 52
  palabra segmento bruta = 0x52C2

Total referencias encontradas: 72

================================================================================
 POSIBLES COMIENZOS DE PROCEDIMIENTO CERCA DE 80:0444
================================================================================
candidato 80:0183 (55 8B EC) distancia=+0x2C1
candidato 80:019D (55 8B EC) distancia=+0x2A7
candidato 80:01BD (55 8B EC) distancia=+0x287
candidato 80:01EB (55 8B EC) distancia=+0x259

Número de candidatos: 4

================================================================================
 CONTEXTO DE LOS CANDIDATOS
================================================================================

--- 80:0183 (55 8B EC) ---
0183: 55 8B EC 1E 8B 46 06 E8 92 00 8B E5 5D 4D 72 03
0193: CA 02 00 B8 01 00 E9 C4 FE 45 55 8B EC 1E 8B 46
01A3: 06 8B 4E 08 8B 5E 0A E8 8F 01 8B E5 5D 4D 72 03
01B3: CA 06 00 B8 02 00 E9 A4 FE 45 55 8B EC 1E B8 00
01C3: 10 50 9A FF FF 00 00 8B 0E 3C 25 E3 15 8E C1 26
01D3: 03 06 08 00 83 D2 00 26 8B 0E 0A 00 3B 0E 3C 25
01E3: 75 EB 8B E5 5D 4D CB 45 55 8B EC 1E 33 C0 50 50
01F3: 9A FF FF 00 00 0B D2 75 1E 8B 0E 3C 25 E3 18 8E

--- 80:019D (55 8B EC) ---
019D: 55 8B EC 1E 8B 46 06 8B 4E 08 8B 5E 0A E8 8F 01
01AD: 8B E5 5D 4D 72 03 CA 06 00 B8 02 00 E9 A4 FE 45
01BD: 55 8B EC 1E B8 00 10 50 9A FF FF 00 00 8B 0E 3C
01CD: 25 E3 15 8E C1 26 03 06 08 00 83 D2 00 26 8B 0E
01DD: 0A 00 3B 0E 3C 25 75 EB 8B E5 5D 4D CB 45 55 8B
01ED: EC 1E 33 C0 50 50 9A FF FF 00 00 0B D2 75 1E 8B
01FD: 0E 3C 25 E3 18 8E C1 26 3B 06 08 00 73 04 26 A1
020D: 08 00 26 8B 0E 0A 00 3B 0E 3C 25 75 E8 8B E5 5D

--- 80:01BD (55 8B EC) ---
01BD: 55 8B EC 1E B8 00 10 50 9A FF FF 00 00 8B 0E 3C
01CD: 25 E3 15 8E C1 26 03 06 08 00 83 D2 00 26 8B 0E
01DD: 0A 00 3B 0E 3C 25 75 EB 8B E5 5D 4D CB 45 55 8B
01ED: EC 1E 33 C0 50 50 9A FF FF 00 00 0B D2 75 1E 8B
01FD: 0E 3C 25 E3 18 8E C1 26 3B 06 08 00 73 04 26 A1
020D: 08 00 26 8B 0E 0A 00 3B 0E 3C 25 75 E8 8B E5 5D
021D: 4D CB 0B C0 74 60 A3 86 62 8B 1E 28 25 0B 1E 2A
022D: 25 74 06 50 FF 1E 28 25 58 3B 06 3E 25 72 1F E8

--- 80:01EB (55 8B EC) ---
01EB: 55 8B EC 1E 33 C0 50 50 9A FF FF 00 00 0B D2 75
01FB: 1E 8B 0E 3C 25 E3 18 8E C1 26 3B 06 08 00 73 04
020B: 26 A1 08 00 26 8B 0E 0A 00 3B 0E 3C 25 75 E8 8B
021B: E5 5D 4D CB 0B C0 74 60 A3 86 62 8B 1E 28 25 0B
022B: 1E 2A 25 74 06 50 FF 1E 28 25 58 3B 06 3E 25 72
023B: 1F E8 48 00 73 45 83 3E 3E 25 00 74 20 A1 86 62
024B: 8B 1E 40 25 83 EB 0C 3B C3 77 12 E8 48 00 EB 0B
025B: E8 43 00 73 26 A1 86 62 E8 21 00 73 1E A1 2C 25

================================================================================
 FIN
================================================================================
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Perfecto. El siguiente paso es **no interpretar todavía los nombres de las rutinas**, sino reconstruir exactamente qué hacen y a qué segmentos llaman. Así evitamos asignarles una función equivocada.


Vamos a hacer tres cosas en una sola ejecución:


1. Desensamblar exactamente `76:18A9`.
2. Desensamblar exactamente `76:18F4`.
3. Desensamblar `80:0444` desde su verdadero punto de entrada, no desde `80:0400`.
4. Para cada `CALL FAR` encontrado, resolver automáticamente su segmento mediante las cadenas de relocación NE.
5. Mostrar las direcciones lógicas reales de las llamadas.


Ejecuta **todo este único bloque**:


```
Bashcd /storage/emulated/0/Download/armon

python3 - <<'PY'
from pathlib import Path
import struct
import subprocess

EXE = Path("ARMON.EXE")
data = EXE.read_bytes()

NE = 0x100
SEG_COUNT = struct.unpack_from("<H", data, NE + 0x1C)[0]
SEG_TABLE = NE + struct.unpack_from("<H", data, NE + 0x22)[0]
SHIFT = struct.unpack_from("<H", data, NE + 0x32)[0]

# ============================================================
# TABLA DE SEGMENTOS
# ============================================================

segments = {}

for n in range(1, SEG_COUNT + 1):
    p = SEG_TABLE + (n - 1) * 8

    sector = struct.unpack_from("<H", data, p)[0]
    length = struct.unpack_from("<H", data, p + 2)[0]
    flags  = struct.unpack_from("<H", data, p + 4)[0]

    size = 0x10000 if length == 0 else length
    fileoff = sector << SHIFT

    segments[n] = {
        "fileoff": fileoff,
        "size": size,
        "flags": flags
    }

# ============================================================
# RELOCACIONES DEL SEGMENTO 3
# ============================================================

SEG3 = 3
seg3 = segments[SEG3]

SEG3_START = seg3["fileoff"]
SEG3_SIZE  = seg3["size"]
RELOC      = SEG3_START + SEG3_SIZE

reloc_count = struct.unpack_from("<H", data, RELOC)[0]
reloc_base = RELOC + 2

relocs = []

for i in range(reloc_count):

    p = reloc_base + i * 8

    rtype  = data[p]
    flags  = data[p + 1]
    source = struct.unpack_from("<H", data, p + 2)[0]
    target = data[p + 4:p + 8]

    relocs.append({
        "index": i,
        "type": rtype,
        "flags": flags,
        "source": source,
        "target": target
    })

# ============================================================
# SEGUIR UNA CADENA DE RELOCACION
# ============================================================

def word_at_seg3(off):
    return struct.unpack_from(
        "<H",
        data,
        SEG3_START + off
    )[0]


def chain(start):

    result = []
    seen = set()
    cur = start

    while cur != 0xFFFF:

        if cur in seen:
            break

        if cur >= SEG3_SIZE - 1:
            break

        seen.add(cur)
        result.append(cur)

        cur = word_at_seg3(cur)

    return result


# ============================================================
# RESOLVER SEGMENTO DE UNA RELOCACION INTERNA
# ============================================================

def target_segment(r):

    # Target kind = bits 0..1 de flags.
    # 00 = referencia interna.
    if r["type"] != 0x02:
        return None

    if (r["flags"] & 3) != 0:
        return None

    # Para relocación tipo 02 interna,
    # el primer WORD del target contiene el segmento.
    return r["target"][0]


# ============================================================
# CONSTRUIR MAPA:
# CADA POSICION DE CADENA -> SEGMENTO DESTINO
# ============================================================

source_to_target = {}

for r in relocs:

    if r["type"] != 0x02:
        continue

    if (r["flags"] & 3) != 0:
        continue

    seg = target_segment(r)

    if seg is None:
        continue

    ch = chain(r["source"])

    for x in ch:
        source_to_target[x] = {
            "segment": seg,
            "reloc_index": r["index"],
            "chain_start": r["source"]
        }

# ============================================================
# DESENSAMBLADOR
# ============================================================

def disassemble(blob):

    hexbytes = " ".join(
        f"0x{x:02X}"
        for x in blob
    )

    r = subprocess.run(
        [
            "llvm-mc",
            "--disassemble",
            "--triple=i386-unknown-none-code16",
            "--x86-asm-syntax=intel"
        ],
        input=hexbytes + "\n",
        text=True,
        capture_output=True
    )

    return r.stdout, r.stderr


# ============================================================
# ANALIZAR UNA RUTINA
# ============================================================

def analizar(nombre, segnum, inicio, fin):

    s = segments[segnum]

    blob = data[
        s["fileoff"] + inicio:
        s["fileoff"] + fin
    ]

    print()
    print("=" * 80)
    print(nombre)
    print("=" * 80)

    print(
        f"Direccion inicial : {segnum}:{inicio:04X}"
    )

    print(
        f"Direccion archivo  : 0x{s['fileoff'] + inicio:06X}"
    )

    print(
        f"Longitud           : 0x{len(blob):04X}"
    )

    print()
    print("--- DESENSAMBLADO ---")
    print()

    asm, stderr = disassemble(blob)

    print(asm)

    if stderr:
        print("--- AVISOS DEL DESENSAMBLADOR ---")
        print(stderr)

    # --------------------------------------------------------
    # Buscar CALL FAR reales dentro del bloque.
    #
    # 9A xx xx xx xx
    #
    # El segmento está en inicio relativo + 3.
    # --------------------------------------------------------

    print("--- CALL FAR Y RELOCACIONES ---")
    print()

    calls = []

    for i in range(len(blob) - 4):

        if blob[i] != 0x9A:
            continue

        off = (
            blob[i + 1] |
            (blob[i + 2] << 8)
        )

        seg_source = (
            inicio + i + 3
        )

        raw_seg = (
            blob[i + 3] |
            (blob[i + 4] << 8)
        )

        info = source_to_target.get(seg_source)

        if info:

            target_seg = info["segment"]

            print(
                f"{segnum}:{inicio+i:04X}  "
                f"CALL FAR -> "
                f"{target_seg}:{off:04X}"
            )

            print(
                f"    fuente relocacion = "
                f"0x{seg_source:04X}"
            )

            print(
                f"    registro relocacion = "
                f"{info['reloc_index']}"
            )

            print(
                f"    cadena inicia en = "
                f"0x{info['chain_start']:04X}"
            )

            print(
                f"    palabra segmento bruta = "
                f"0x{raw_seg:04X}"
            )

            calls.append(
                (i, target_seg, off)
            )

        else:

            print(
                f"{segnum}:{inicio+i:04X}  "
                f"CALL FAR -> "
                f"??:{off:04X}"
            )

            print(
                f"    fuente relocacion = "
                f"0x{seg_source:04X}"
            )

            print(
                f"    palabra segmento bruta = "
                f"0x{raw_seg:04X}"
            )

    if not calls:
        print("No se encontraron CALL FAR resolubles.")

    return calls


# ============================================================
# 1. RUTINA 76:18A9
# ============================================================

calls_18A9 = analizar(
    "RUTINA 76:18A9",
    76,
    0x18A9,
    0x18F4
)


# ============================================================
# 2. RUTINA 76:18F4
# ============================================================

calls_18F4 = analizar(
    "RUTINA 76:18F4",
    76,
    0x18F4,
    0x1921
)


# ============================================================
# 3. RUTINA 80:0444
# ============================================================

calls_0444 = analizar(
    "RUTINA 80:0444",
    80,
    0x0444,
    0x0464
)


# ============================================================
# RESUMEN
# ============================================================

print()
print("=" * 80)
print("RESUMEN DE DIRECCIONES")
print("=" * 80)

print()
print("76:18A9")
for _, seg, off in calls_18A9:
    print(f"    -> {seg}:{off:04X}")

print()
print("76:18F4")
for _, seg, off in calls_18F4:
    print(f"    -> {seg}:{off:04X}")

print()
print("80:0444")
for _, seg, off in calls_0444:
    print(f"    -> {seg}:{off:04X}")

print()
print("=" * 80)
print("FIN")
print("=" * 80)
PY
```


### Qué buscamos específicamente


El resultado debería permitirnos establecer algo mucho más útil que simplemente el ensamblador:


```
CalcularClick
    |
    +--> 80:0444
    |
    +--> 76:18F4
    |
    +--> 76:18A9
```


y después:


```
76:18A9
    |
    +--> ¿qué rutina?
    +--> ¿qué parámetros?
    +--> ¿qué devuelve?

76:18F4
    |
    +--> ¿qué rutina?
    +--> ¿qué parámetros?
    +--> ¿qué devuelve?

80:0444
    |
    +--> ¿rutina RTL de Delphi?
    +--> ¿rutina de gestión de memoria?
    +--> ¿rutina matemática?
    +--> ¿rutina propia de ARMÓN?
```


Hay además una pista importante: el ejecutable conserva `Borland Delphi` y su estructura de formularios/eventos, y Delphi 1 fue precisamente el compilador de 16 bits para Windows 3.x.


**Pégame la salida completa de este bloque.** Con ella podemos empezar a poner nombres funcionales a estas rutinas y avanzar desde `CalcularClick` hacia el código matemático original de ARMÓN.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> from pathlib import Path
> import struct
> import subprocess
>
> EXE = Path("ARMON.EXE")
> data = EXE.read_bytes()
>
> NE = 0x100
> SEG_COUNT = struct.unpack_from("<H", data, NE + 0x1C)[0]
> SEG_TABLE = NE + struct.unpack_from("<H", data, NE + 0x22)[0]
> SHIFT = struct.unpack_from("<H", data, NE + 0x32)[0]
>
> # ============================================================
> # TABLA DE SEGMENTOS
> # ============================================================
>
> segments = {}
>
> for n in range(1, SEG_COUNT + 1):
>     p = SEG_TABLE + (n - 1) * 8
>
>     sector = struct.unpack_from("<H", data, p)[0]
>     length = struct.unpack_from("<H", data, p + 2)[0]
>     flags  = struct.unpack_from("<H", data, p + 4)[0]
>
>     size = 0x10000 if length == 0 else length
>     fileoff = sector << SHIFT
>
>     segments[n] = {
>         "fileoff": fileoff,
>         "size": size,
>         "flags": flags
>     }
>
> # ============================================================
> # RELOCACIONES DEL SEGMENTO 3
> # ============================================================
>
> SEG3 = 3
> seg3 = segments[SEG3]
>
 > SEG3_START = seg3["fileoff"]
> SEG3_SIZE  = seg3["size"]
> RELOC      = SEG3_START + SEG3_SIZE
>                                                      ]
> reloc_base = RELOC + 2npack_from("<H", data, RELOC)[0]
>
> relocs = []
>
> for i in range(reloc_count):
:{i>
>     p = reloc_base + i * 8
>
>     rtype  = data[p]
>     flags  = data[p + 1]
>     source = struct.unpack_from("<H", data, p + 2)[0]
>     target = data[p + 4:p + 8]
>
>     relocs.append({
>         "index": i,
>         "type": rtype,
>         "flags": flags,
>         "source": source,
>         "target": target
>     })
>
> # ============================================================
> # SEGUIR UNA CADENA DE RELOCACION
> # ============================================================
>
 > def word_at_seg3(off):
>     return struct.unpack_from(
>         "<H",
>         data,
>         SEG3_START + off
>     )[0]
>
 >
> def chain(start):
>
>     result = []
>     seen = set()
>     cur = start
>
 >     while cur != 0xFFFF:
>
>         if cur in seen:
>             break
>
>         if cur >= SEG3_SIZE - 1:
>             break
>
>         seen.add(cur)
>         result.append(cur)
>
>         cur = word_at_seg3(cur)
>
>     return result
>
>
> # ============================================================
> # RESOLVER SEGMENTO DE UNA RELOCACION INTERNA
> # ============================================================
>
)> def target_segment(r):
>
>     # Target kind = bits 0..1 de flags.
>     # 00 = referencia interna.
>     if r["type"] != 0x02:
==============>         return None
==============>
>     if (r["flags"] & 3) != 0:
>         return None
>
>     # Para relocación tipo 02 interna,
>     # el primer WORD del target contiene el segmento.
>     return r["target"][0]
>
>
> # ============================================================
> # CONSTRUIR MAPA:
> # CADA POSICION DE CADENA -> SEGMENTO DESTINO
> # ============================================================
>
7> source_to_target = {}
>
x> for r in relocs:
>
>     if r["type"] != 0x02:
>         continue
>
=>     if (r["flags"] & 3) != 0:
>         continue
>
>     seg = target_segment(r)
>
=>     if seg is None:
>         continue
>
>     ch = chain(r["source"])
>
:>     for x in ch:
>         source_to_target[x] = {
>             "segment": seg,
>             "reloc_index": r["index"],
>             "chain_start": r["source"]
>         }
>
=> # ============================================================
> # DESENSAMBLADOR
> # ============================================================
>
 > def disassemble(blob):
>
{>     hexbytes = " ".join(
>         f"0x{x:02X}"
>         for x in blob
>     )
>
>     r = subprocess.run(
>         [
>             "llvm-mc",
>             "--disassemble",
>             "--triple=i386-unknown-none-code16",
>             "--x86-asm-syntax=intel"
>         ],
>         input=hexbytes + "\n",
>         text=True,
>         capture_output=True
>     )
>
>     return r.stdout, r.stderr
>
>
> # ============================================================
> # ANALIZAR UNA RUTINA
> # ============================================================
>
> def analizar(nombre, segnum, inicio, fin):
>
>     s = segments[segnum]
>
>     blob = data[
>         s["fileoff"] + inicio:
>         s["fileoff"] + fin
>     ]
>
>     print()
>     print("=" * 80)
>     print(nombre)
>     print("=" * 80)
>
>     print(
>         f"Direccion inicial : {segnum}:{inicio:04X}"
>     )
>
>     print(
>         f"Direccion archivo  : 0x{s['fileoff'] + inicio:06X}"
>     )
>
>     print(
>         f"Longitud           : 0x{len(blob):04X}"
>     )
>
>     print()
>     print("--- DESENSAMBLADO ---")
>     print()
>
>     asm, stderr = disassemble(blob)
>
>     print(asm)
>
>     if stderr:
>         print("--- AVISOS DEL DESENSAMBLADOR ---")
>         print(stderr)
>
>     # --------------------------------------------------------
>     # Buscar CALL FAR reales dentro del bloque.
>     #
>     # 9A xx xx xx xx
>     #
>     # El segmento está en inicio relativo + 3.
>     # --------------------------------------------------------
>
>     print("--- CALL FAR Y RELOCACIONES ---")
>     print()
>
>     calls = []
>
>     for i in range(len(blob) - 4):
>
>         if blob[i] != 0x9A:
>             continue
>
>         off = (
>             blob[i + 1] |
>             (blob[i + 2] << 8)
>         )
>
>         seg_source = (
>             inicio + i + 3
>         )
>
>         raw_seg = (
>             blob[i + 3] |
>             (blob[i + 4] << 8)
>         )
>
>         info = source_to_target.get(seg_source)
>
>         if info:
>
>             target_seg = info["segment"]
>
>             print(
>                 f"{segnum}:{inicio+i:04X}  "
>                 f"CALL FAR -> "
>                 f"{target_seg}:{off:04X}"
>             )
>
>             print(
>                 f"    fuente relocacion = "
>                 f"0x{seg_source:04X}"
>             )
>
>             print(
>                 f"    registro relocacion = "
>                 f"{info['reloc_index']}"
>             )
>
>             print(
>                 f"    cadena inicia en = "
>                 f"0x{info['chain_start']:04X}"
>             )
>
>             print(
>                 f"    palabra segmento bruta = "
>                 f"0x{raw_seg:04X}"
>             )
>
>             calls.append(
>                 (i, target_seg, off)
>             )
>
>         else:
>
>             print(
>                 f"{segnum}:{inicio+i:04X}  "
>                 f"CALL FAR -> "
>                 f"??:{off:04X}"
>             )
>
>             print(
>                 f"    fuente relocacion = "
>                 f"0x{seg_source:04X}"
>             )
>
>             print(
>                 f"    palabra segmento bruta = "
>                 f"0x{raw_seg:04X}"
>             )
>
>     if not calls:
>         print("No se encontraron CALL FAR resolubles.")
>
>     return calls
>
>
> # ============================================================
> # 1. RUTINA 76:18A9
> # ============================================================
>
> calls_18A9 = analizar(
>     "RUTINA 76:18A9",
>     76,
>     0x18A9,
>     0x18F4
> )
>
>
> # ============================================================
> # 2. RUTINA 76:18F4
> # ============================================================
>
> calls_18F4 = analizar(
>     "RUTINA 76:18F4",
>     76,
>     0x18F4,
>     0x1921
> )
>
>
> # ============================================================
> # 3. RUTINA 80:0444
> # ============================================================
>
> calls_0444 = analizar(
>     "RUTINA 80:0444",
>     80,
>     0x0444,
>     0x0464
> )
>
>
> # ============================================================
> # RESUMEN
> # ============================================================
>
> print()
> print("=" * 80)
> print("RESUMEN DE DIRECCIONES")
> print("=" * 80)
>
> print()
> print("76:18A9")
> for _, seg, off in calls_18A9:
>     print(f"    -> {seg}:{off:04X}")
>
> print()
> print("76:18F4")
> for _, seg, off in calls_18F4:
>     print(f"    -> {seg}:{off:04X}")
>
> print()
> print("80:0444")
> for _, seg, off in calls_0444:
>     print(f"    -> {seg}:{off:04X}")
>
> print()
> print("=" * 80)
> print("FIN")
> print("=" * 80)
> PY

================================================================================
RUTINA 76:18A9
================================================================================
Direccion inicial : 76:18A9
Direccion archivo  : 0x01D369
Longitud           : 0x004B

--- DESENSAMBLADO ---

        enter   $10, $0
        leaw    -10(%bp), %di
        pushw   %ss
        pushw   %di
        lesw    6(%bp), %di
        pushw   %es
        pushw   %di
        lesw    %es:(%di), %di
        lcallw  *%es:52(%di)
        addw    $4, %sp
        movw    -6(%bp), %ax
        movw    %ax, -2(%bp)
        movw    -2(%bp), %ax
        leave
        lretw   $4
        pushw   %bp
        movw    %sp, %bp
        pushw   10(%bp)
        lesw    6(%bp), %di
        pushw   %es
        pushw   %di
        lcallw  $6382, $6388                    # imm = 0x18EE
                                        # imm = 0x18F4
        pushw   %ax
        lcallw  $6445, $1646                    # imm = 0x192D
                                        # imm = 0x66E
        pushw   %dx
        pushw   %ax
        lesw    6(%bp), %di
        pushw   %es
        pushw   %di
        lcallw  $6436, $7147                    # imm = 0x1924
                                        # imm = 0x1BEB
        leave
        lretw   $6

--- CALL FAR Y RELOCACIONES ---

76:18D9  CALL FAR -> ??:18F4
    fuente relocacion = 0x18DC
    palabra segmento bruta = 0x18EE
76:18DF  CALL FAR -> ??:066E
    fuente relocacion = 0x18E2
    palabra segmento bruta = 0x192D
76:18EB  CALL FAR -> ??:1BEB
    fuente relocacion = 0x18EE
    palabra segmento bruta = 0x1924
No se encontraron CALL FAR resolubles.

================================================================================
RUTINA 76:18F4
================================================================================
Direccion inicial : 76:18F4
Direccion archivo  : 0x01D3B4
Longitud           : 0x002D

--- DESENSAMBLADO ---

        enter   $10, $0
        leaw    -10(%bp), %di
        pushw   %ss
        pushw   %di
        lesw    6(%bp), %di
        pushw   %es
        pushw   %di
        lesw    %es:(%di), %di
        lcallw  *%es:52(%di)
        addw    $4, %sp
        movw    -4(%bp), %ax
        movw    %ax, -2(%bp)
        movw    -2(%bp), %ax
        leave
        lretw   $4
        pushw   %bp
        movw    %sp, %bp
        lesw    6(%bp), %di
        pushw   %es
        pushw   %di

--- CALL FAR Y RELOCACIONES ---

No se encontraron CALL FAR resolubles.

================================================================================
RUTINA 80:0444
================================================================================
Direccion inicial : 80:0444
Direccion archivo  : 0x032E44
Longitud           : 0x0020

--- DESENSAMBLADO ---

        addw    $1024, %ax                      # imm = 0x400
        jb      25
        subw    %sp, %ax
        jae     21
        negw    %ax
        cmpw    %ss:10, %ax
        jb      12
        cmpw    %ss:12, %ax
        jae     4
        movw    %ax, %ss:12
        lretw

--- AVISOS DEL DESENSAMBLADOR ---
<stdin>:1:151: warning: invalid instruction encoding
0x05 0x00 0x04 0x72 0x19 0x2B 0xC4 0x73 0x15 0xF7 0xD8 0x36 0x3B 0x06 0x0A 0x00 0x72 0x0C 0x36 0x3B 0x06 0x0C 0x00 0x73 0x04 0x36 0xA3 0x0C 0x00 0xCB 0xB8 0xCA
                                                                                                                                                      ^
<stdin>:1:156: warning: invalid instruction encoding
0x05 0x00 0x04 0x72 0x19 0x2B 0xC4 0x73 0x15 0xF7 0xD8 0x36 0x3B 0x06 0x0A 0x00 0x72 0x0C 0x36 0x3B 0x06 0x0C 0x00 0x73 0x04 0x36 0xA3 0x0C 0x00 0xCB 0xB8 0xCA
                                                                                                                                                           ^

--- CALL FAR Y RELOCACIONES ---

No se encontraron CALL FAR resolubles.

================================================================================
RESUMEN DE DIRECCIONES
================================================================================

76:18A9

76:18F4

80:0444

================================================================================
FIN
================================================================================
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Sí. Aquí apareció un dato importante: **nuestro mapa de cadenas de relocación no está resolviendo correctamente las rutinas 76:18A9 y 76:18F4**. No debemos concluir todavía que sus `CALL FAR` no tienen destino.


El problema concreto es este:


- En `76:18D9`, el `CALL FAR` está en `18D9`.
- Su palabra de segmento está en `18DC`.
- Pero `source_to_target` no encuentra `18DC`.
- Eso indica que **la cadena de relocación de ese segmento debe analizarse directamente**, no reutilizar el mapa construido a partir de segmento 3.


Además, `80:0444` tampoco parece comenzar una rutina: al llegar allí estamos en mitad de una secuencia que empieza antes. Por tanto, **el destino `80:0444` debe investigarse hacia atrás**, no asumir que `0444` es prólogo.


### Siguiente paso


Vamos a hacer una inspección mucho más precisa de las tres direcciones:


1. Buscar las relocaciones reales de **segmento 76**.
2. Resolver las cadenas de `76:18DC`, `76:18E2` y `76:18EE`.
3. Confirmar los segmentos destino de los tres `CALL FAR` de `76:18A9`.
4. Hacer lo mismo con los `CALL FAR` de `76:18F4`.
5. En segmento 80, buscar **la relocación que apunta a `80:0444`** y localizar el comienzo real de la rutina.
6. Mostrar los bytes anteriores y posteriores a cada destino.


Usa este bloque único:


```
Bashcd /storage/emulated/0/Download/armon

python3 - <<'PY'
from pathlib import Path
import struct

EXE = Path("ARMON.EXE")
data = EXE.read_bytes()

NE = 0x100
SEG_COUNT = struct.unpack_from("<H", data, NE + 0x1C)[0]
SEG_TABLE = NE + struct.unpack_from("<H", data, NE + 0x22)[0]
SHIFT = struct.unpack_from("<H", data, NE + 0x32)[0]

# ============================================================
# TABLA DE SEGMENTOS
# ============================================================

segments = {}

for n in range(1, SEG_COUNT + 1):
    p = SEG_TABLE + (n - 1) * 8

    sector = struct.unpack_from("<H", data, p)[0]
    length = struct.unpack_from("<H", data, p + 2)[0]
    flags  = struct.unpack_from("<H", data, p + 4)[0]

    size = 0x10000 if length == 0 else length
    fileoff = sector << SHIFT

    segments[n] = {
        "fileoff": fileoff,
        "size": size,
        "flags": flags
    }

# ============================================================
# LEER RELOCACIONES DE UN SEGMENTO
# ============================================================

def read_relocations(segnum):

    s = segments[segnum]

    start = s["fileoff"]
    size  = s["size"]

    reloc_base = start + size

    count = struct.unpack_from("<H", data, reloc_base)[0]

    records = []

    for i in range(count):

        p = reloc_base + 2 + i * 8

        rtype  = data[p]
        flags  = data[p + 1]
        source = struct.unpack_from("<H", data, p + 2)[0]
        target = data[p + 4:p + 8]

        records.append({
            "index": i,
            "type": rtype,
            "flags": flags,
            "source": source,
            "target": target
        })

    return records

# ============================================================
# SEGUIR CADENA DE RELOCACION DENTRO DE UN SEGMENTO
# ============================================================

def word_at(segnum, off):

    s = segments[segnum]

    return struct.unpack_from(
        "<H",
        data,
        s["fileoff"] + off
    )[0]


def chain(segnum, start):

    s = segments[segnum]

    result = []
    seen = set()

    cur = start

    while cur != 0xFFFF:

        if cur in seen:
            result.append(("BUCLE", cur))
            break

        if cur < 0 or cur + 1 >= s["size"]:
            result.append(("FUERA", cur))
            break

        seen.add(cur)

        result.append(("POS", cur))

        cur = word_at(segnum, cur)

    return result

# ============================================================
# INFORMACION DE TARGET
# ============================================================

def decode_internal_target(r):

    if r["type"] != 0x02:
        return None

    # bits 0-1:
    # 00 = referencia interna
    if (r["flags"] & 3) != 0:
        return None

    # Para este tipo de relocación:
    # target[0:2] = segmento
    target_seg = struct.unpack_from(
        "<H",
        r["target"],
        0
    )[0]

    return target_seg

# ============================================================
# MOSTRAR RELOCACIONES QUE AFECTAN UNA POSICION
# ============================================================

def inspect_source(segnum, source):

    print()
    print("=" * 80)
    print(f"SEGMENTO {segnum}: FUENTE 0x{source:04X}")
    print("=" * 80)

    relocs = read_relocations(segnum)

    encontrados = []

    for r in relocs:

        if r["type"] != 0x02:
            continue

        # Solo relocaciones internas
        if (r["flags"] & 3) != 0:
            continue

        ch = chain(segnum, r["source"])

        posiciones = [
            x[1]
            for x in ch
            if x[0] == "POS"
        ]

        if source not in posiciones:
            continue

        target_seg = decode_internal_target(r)

        print()
        print(
            f"registro     : {r['index']}"
        )

        print(
            f"tipo         : {r['type']:02X}"
        )

        print(
            f"flags        : {r['flags']:02X}"
        )

        print(
            f"inicio cadena: 0x{r['source']:04X}"
        )

        print(
            f"segmento destino: {target_seg}"
        )

        print(
            "cadena:"
        )

        print(
            " -> ".join(
                f"0x{x[1]:04X}"
                for x in ch
                if x[0] == "POS"
            )
        )

        encontrados.append(
            (r, target_seg, ch)
        )

    if not encontrados:
        print()
        print("NO SE ENCONTRO UNA RELOCACION INTERNA.")

    return encontrados

# ============================================================
# 1. LOS TRES CALL DE 76:18A9
# ============================================================

print()
print("#" * 80)
print("# CALLS DE 76:18A9")
print("#" * 80)

r1 = inspect_source(76, 0x18DC)
r2 = inspect_source(76, 0x18E2)
r3 = inspect_source(76, 0x18EE)

# ============================================================
# 2. CALLS DE 76:18F4
#
# La rutina contiene:
#
# 18F4 ...
# CALL 18A9
#
# Hay que localizar automáticamente los offsets de segmento.
# ============================================================

print()
print("#" * 80)
print("# CALLS FAR DENTRO DE 76:18F4")
print("#" * 80)

s76 = segments[76]

blob76 = data[
    s76["fileoff"] + 0x18F4:
    s76["fileoff"] + 0x1921
]

for i in range(len(blob76) - 4):

    if blob76[i] != 0x9A:
        continue

    off = (
        blob76[i + 1] |
        (blob76[i + 2] << 8)
    )

    source = 0x18F4 + i + 3

    rawseg = (
        blob76[i + 3] |
        (blob76[i + 4] << 8)
    )

    print()
    print(
        f"CALL en 76:{0x18F4+i:04X}"
    )

    print(
        f"offset destino : 0x{off:04X}"
    )

    print(
        f"source reloc   : 0x{source:04X}"
    )

    print(
        f"segmento bruto : 0x{rawseg:04X}"
    )

    inspect_source(76, source)

# ============================================================
# 3. BUSCAR TODAS LAS RELOCACIONES QUE APUNTEN A SEGMENTO 80
# ============================================================

print()
print("#" * 80)
print("# TODAS LAS RELOCACIONES INTERNAS HACIA SEGMENTO 80")
print("#" * 80)

for segnum in range(1, SEG_COUNT + 1):

    try:
        relocs = read_relocations(segnum)
    except Exception:
        continue

    for r in relocs:

        if r["type"] != 0x02:
            continue

        if (r["flags"] & 3) != 0:
            continue

        target_seg = decode_internal_target(r)

        if target_seg != 80:
            continue

        ch = chain(segnum, r["source"])

        posiciones = [
            x[1]
            for x in ch
            if x[0] == "POS"
        ]

        # Mostrar solamente cadenas que contengan 0444
        if 0x0444 in posiciones:

            print()
            print(
                f"SEGMENTO FUENTE : {segnum}"
            )

            print(
                f"REGISTRO        : {r['index']}"
            )

            print(
                f"INICIO CADENA   : 0x{r['source']:04X}"
            )

            print(
                "CADENA:"
            )

            print(
                " -> ".join(
                    f"0x{x:04X}"
                    for x in posiciones
                )
            )

# ============================================================
# 4. BUSCAR 80:0444 COMO POSICION DE RELOCACION
# ============================================================

print()
print("#" * 80)
print("# RELOCACIONES QUE AFECTAN DIRECTAMENTE 80:0444")
print("#" * 80)

relocs80 = read_relocations(80)

for r in relocs80:

    if r["type"] != 0x02:
        continue

    if (r["flags"] & 3) != 0:
        continue

    ch = chain(80, r["source"])

    posiciones = [
        x[1]
        for x in ch
        if x[0] == "POS"
    ]

    if 0x0444 in posiciones:

        target_seg = decode_internal_target(r)

        print()
        print(
            f"registro      : {r['index']}"
        )

        print(
            f"inicio cadena : 0x{r['source']:04X}"
        )

        print(
            f"destino seg   : {target_seg}"
        )

        print(
            "cadena:"
        )

        print(
            " -> ".join(
                f"0x{x:04X}"
                for x in posiciones
            )
        )

# ============================================================
# 5. CONTEXTO RAW DE 80:0444
# ============================================================

print()
print("#" * 80)
print("# CONTEXTO BINARIO DE 80:0444")
print("#" * 80)

s80 = segments[80]

for centro in [0x0444]:

    inicio = max(0, centro - 0x40)
    fin = min(s80["size"], centro + 0x80)

    print()
    print(
        f"80:{inicio:04X} - 80:{fin:04X}"
    )

    for p in range(inicio, fin, 16):

        row = data[
            s80["fileoff"] + p:
            s80["fileoff"] + min(p + 16, fin)
        ]

        print(
            f"{p:04X}: "
            + " ".join(
                f"{x:02X}"
                for x in row
            )
        )

# ============================================================
# FIN
# ============================================================

print()
print("=" * 80)
print("FIN DE LA INVESTIGACION")
print("=" * 80)
PY
```


Hay además una observación importante de lo que acabamos de obtener: **`76:18A9` y `76:18F4` parecen formar un par de rutinas auxiliares estrechamente relacionadas**. En `18A9` aparecen llamadas a `18F4`, `066E` y `1BEB`; y en `18F4` aparece una llamada de vuelta a `18A9`. Eso puede ser perfectamente una pareja de rutinas Delphi generadas por el compilador, pero primero debemos resolver sus segmentos reales.


El siguiente resultado nos permitirá pasar de **“bytes y relocaciones” a “procedimientos concretos del programa”**.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
ur = start

    while cur != 0xFFFF:

        if cur in seen:
            result.append(("BUCLE", cur))
            break

        if cur < 0 or cur + 1 >= s["size"]:
            result.append(("FUERA", cur))
            break

        seen.add(cur)

        result.append(("POS", cur))

        cur = .../Download/armon $
.../Download/armon $ python3 - <<'PY'
> from pathlib import Path
> import struct
>
> EXE = Path("ARMON.EXE")
> data = EXE.read_bytes()
>
> NE = 0x100
> SEG_COUNT = struct.unpack_from("<H", data, NE + 0x1C)[0]
> SEG_TABLE = NE + struct.unpack_from("<H", data, NE + 0x22)[0]
> SHIFT = struct.unpack_from("<H", data, NE + 0x32)[0]
>
> # ============================================================
> # TABLA DE SEGMENTOS
> # ============================================================
aciones internas
        if (r["flags"] & 3) >
> segments = {}
>
> for n in range(1, SEG_COUNT + 1):
>     p = SEG_TABLE + (n - 1) * 8
>
=>     sector = struct.unpack_from("<H", data, p)[0]
>     length = struct.unpack_from("<H", data, p + 2)[0]
>     flags  = struct.unpack_from("<H", data, p + 4)[0]
>
2>     size = 0x10000 if length == 0 else length
>     fileoff = sector << SHIFT
>
>     segments[n] = {
= inspect_source(76, 0x18EE)

# ============================================================>         "fileoff": fileoff,
 L>         "size": size,
>         "flags": flags
>     }
>
> # ============================================================
> # LEER RELOCACIONES DE UN SEGMENTO
> # ============================================================
 CALLS F>
> def read_relocations(segnum):
>
>     s = segments[segnum]
>
>     start = s["fileoff"]
>     size  = s["size"]
>
 >     reloc_base = start + size
>
o>     count = struct.unpack_from("<H", data, reloc_base)[0]
>
>     records = []
>
 >     for i in range(count):
>
>         p = reloc_base + 2 + i * 8
>

>         rtype  = data[p]
>         flags  = data[p + 1]
>         source = struct.unpack_from("<H", data, p + 2)[0]
>         target = data[p + 4:p + 8]
>
L>         records.append({
>             "index": i,
>             "type": rtype,
>             "flags": flags,
>             "source": source,
>             "target": target
>         })
>
e>     return records
>
e> # ============================================================
> # SEGUIR CADENA DE RELOCACION DENTRO DE UN SEGMENTO
> # ============================================================
>
=> def word_at(segnum, off):
>
=>     s = segments[segnum]
>
=>     return struct.unpack_from(
>         "<H",
>         data,
>         s["fileoff"] + off
>     )[0]
>
>
> def chain(segnum, start):
>
e>     s = segments[segnum]
>
t>     result = []
>     seen = set()
>
i>     cur = start
>
e>     while cur != 0xFFFF:
>
>         if cur in seen:
>             result.append(("BUCLE", cur))
>             break
>
>         if cur < 0 or cur + 1 >= s["size"]:
>             result.append(("FUERA", cur))
>             break
>
 >         seen.add(cur)
>
>         result.append(("POS", cur))
>
=>         cur = word_at(segnum, cur)
>
 >     return result
>
> # ============================================================
> # INFORMACION DE TARGET
> # ============================================================
>
 > def decode_internal_target(r):
>
>     if r["type"] != 0x02:
>         return None
ME>
N>     # bits 0-1:
>     # 00 = referencia interna
>     if (r["flags"] & 3) != 0:
>         return None
>
>     # Para este tipo de relocación:
>     # target[0:2] = segmento
>     target_seg = struct.unpack_from(
>         "<H",
>         r["target"],
         )

            print(
    >         0
>     )[0]
>
>     return target_seg
>
> # ============================================================
> # MOSTRAR RELOCACIONES QUE AFECTAN UNA POSICION
> # ============================================================
>
> def inspect_source(segnum, source):
>
>     print()
>     print("=" * 80)                                  )
>     print("=" * 80)O {segnum}: FUENTE 0x{source:04X}")
>
>     relocs = read_relocations(segnum)
>
>     encontrados = []
>
>     for r in relocs:
>
>         if r["type"] != 0x02:
>             continue
>
>         # Solo relocaciones internas
>         if (r["flags"] & 3) != 0:
>             continue
>
>         ch = chain(segnum, r["source"])
>
>         posiciones = [
>             x[1]
>             for x in ch
>             if x[0] == "POS"
>         ]
>
0>         if source not in posiciones:
>             continue
>
a>         target_seg = decode_internal_target(r)
>
 >         print()
>         print(
>             f"registro     : {r['index']}"
>         )
>
a>         print(
>             f"tipo         : {r['type']:02X}"
>         )
>
t>         print(
>             f"flags        : {r['flags']:02X}"
>         )
>
>         print(
>             f"inicio cadena: 0x{r['source']:04X}"
>         )
>
>         print(
>             f"segmento destino: {target_seg}"
>         )
>
>         print(
>             "cadena:"
>         )
>
=>         print(
>             " -> ".join(
>                 f"0x{x[1]:04X}"
>                 for x in ch
>                 if x[0] == "POS"
>             )
>         )
>
>         encontrados.append(
>             (r, target_seg, ch)
>         )
>
 >     if not encontrados:
>         print()
>         print("NO SE ENCONTRO UNA RELOCACION INTERNA.")
>
>     return encontrados
>
> # ============================================================
> # 1. LOS TRES CALL DE 76:18A9
> # ============================================================
>
> print()
> print("#" * 80)
> print("# CALLS DE 76:18A9")
> print("#" * 80)
>
> r1 = inspect_source(76, 0x18DC)
> r2 = inspect_source(76, 0x18E2)
> r3 = inspect_source(76, 0x18EE)
>
> # ============================================================
> # 2. CALLS DE 76:18F4
=======> #
> # La rutina contiene:
> #
==> # 18F4 ...
> # CALL 18A9
> #
> # Hay que localizar automáticamente los offsets de seg
mento.
> # ============================================================
>
> print()
> print("#" * 80)
> print("# CALLS FAR DENTRO DE 76:18F4")
> print("#" * 80)
>
> s76 = segments[76]
>
> blob76 = data[
>     s76["fileoff"] + 0x18F4:
>     s76["fileoff"] + 0x1921
> ]
>
> for i in range(len(blob76) - 4):
>
>     if blob76[i] != 0x9A:
>         continue
>
>     off = (
>         blob76[i + 1] |
>         (blob76[i + 2] << 8)
>     )
>
>     source = 0x18F4 + i + 3
>
>     rawseg = (
>         blob76[i + 3] |
>         (blob76[i + 4] << 8)
>     )
>
>     print()
>     print(
>         f"CALL en 76:{0x18F4+i:04X}"
>     )
>
>     print(
>         f"offset destino : 0x{off:04X}"
>     )
>
>     print(
>         f"source reloc   : 0x{source:04X}"
>     )
>
>     print(
>         f"segmento bruto : 0x{rawseg:04X}"
>     )
>
>     inspect_source(76, source)
>
> # ============================================================
> # 3. BUSCAR TODAS LAS RELOCACIONES QUE APUNTEN A SEGMENTO 80
> # ============================================================
>
> print()
> print("#" * 80)
> print("# TODAS LAS RELOCACIONES INTERNAS HACIA SEGMENTO 80")
> print("#" * 80)
>
> for segnum in range(1, SEG_COUNT + 1):
>
>     try:
>         relocs = read_relocations(segnum)
>     except Exception:
>         continue
>
>     for r in relocs:
>
>         if r["type"] != 0x02:
>             continue
>
>         if (r["flags"] & 3) != 0:
>             continue
>
>         target_seg = decode_internal_target(r)
>
>         if target_seg != 80:
>             continue
>
>         ch = chain(segnum, r["source"])
>
>         posiciones = [
>             x[1]
>             for x in ch
>             if x[0] == "POS"
>         ]
>                                                      4
>         if 0x0444 in posiciones:nas que contengan 0444
>
>             print()
>             print(
>                 f"SEGMENTO FUENTE : {segnum}"
>             )
>
>             print(
>                 f"REGISTRO        : {r['index']}"
>             )
>
>             print(
>                 f"INICIO CADENA   : 0x{r['source']:04X}"
>             )
>
>             print(
>                 "CADENA:"
>             )
>
>             print(
>                 " -> ".join(
>                     f"0x{x:04X}"
>                     for x in posiciones
>                 )
>             )
>
> # ============================================================
> # 4. BUSCAR 80:0444 COMO POSICION DE RELOCACION
> # ============================================================
>
> print()
> print("#" * 80)
> print("# RELOCACIONES QUE AFECTAN DIRECTAMENTE 80:0444")
> print("#" * 80)
>
> relocs80 = read_relocations(80)
>
> for r in relocs80:
>
>     if r["type"] != 0x02:
>         continue
>
>     if (r["flags"] & 3) != 0:
>         continue
>
>     ch = chain(80, r["source"])
>
>     posiciones = [
>         x[1]
>         for x in ch
>         if x[0] == "POS"
>     ]
>
>     if 0x0444 in posiciones:
>
>         target_seg = decode_internal_target(r)
>
>         print()
>         print(
>             f"registro      : {r['index']}"
>         )
>
>         print(
>             f"inicio cadena : 0x{r['source']:04X}"
>         )
>
>         print(
>             f"destino seg   : {target_seg}"
>         )
>
>         print(
>             "cadena:"
>         )
>
>         print(
>             " -> ".join(
>                 f"0x{x:04X}"
>                 for x in posiciones
>             )
>         )
>
> # ============================================================
> # 5. CONTEXTO RAW DE 80:0444
> # ============================================================
>
> print()
> print("#" * 80)
> print("# CONTEXTO BINARIO DE 80:0444")
> print("#" * 80)
>
> s80 = segments[80]
>
> for centro in [0x0444]:
>
>     inicio = max(0, centro - 0x40)
>     fin = min(s80["size"], centro + 0x80)
>
>     print()
>     print(
>         f"80:{inicio:04X} - 80:{fin:04X}"
>     )
>
>     for p in range(inicio, fin, 16):
>
>         row = data[
>             s80["fileoff"] + p:
>             s80["fileoff"] + min(p + 16, fin)
>         ]
>
>         print(
>             f"{p:04X}: "
>             + " ".join(
>                 f"{x:02X}"
>                 for x in row
>             )
>         )
>
> # ============================================================
> # FIN
> # ============================================================
>
> print()
> print("=" * 80)
> print("FIN DE LA INVESTIGACION")
> print("=" * 80)
> PY

################################################################################
# CALLS DE 76:18A9
################################################################################

================================================================================
SEGMENTO 76: FUENTE 0x18DC
================================================================================

registro     : 2
tipo         : 02
flags        : 00
inicio cadena: 0x00AC
segmento destino: 76
cadena:
0x00AC -> 0x0094 -> 0x00CF -> 0x00F5 -> 0x013B -> 0x038B -> 0x039F -> 0x03CB -> 0x03CF -> 0x03D3 -> 0x03A7 -> 0x03AF -> 0x03B3 -> 0x03D7 -> 0x03B7 -> 0x03DB -> 0x03C7 -> 0x0397 -> 0x03DF -> 0x03E3 -> 0x03E7 -> 0x03EB -> 0x0442 -> 0x0446 -> 0x044A -> 0x044E -> 0x0452 -> 0x0456 -> 0x045A -> 0x045E -> 0x0462 -> 0x0466 -> 0x046A -> 0x046E -> 0x0472 -> 0x0476 -> 0x047A -> 0x047E -> 0x0482 -> 0x0486 -> 0x048A -> 0x048E -> 0x0492 -> 0x0496 -> 0x049A -> 0x049E -> 0x04A2 -> 0x04A6 -> 0x04AA -> 0x04AE -> 0x04B2 -> 0x04B6 -> 0x04BA -> 0x04BE -> 0x04C2 -> 0x04C6 -> 0x04CA -> 0x04CE -> 0x04DC -> 0x04F9 -> 0x0516 -> 0x0532 -> 0x0550 -> 0x0567 -> 0x056F -> 0x058A -> 0x058E -> 0x0617 -> 0x061B -> 0x061F -> 0x0623 -> 0x05B3 -> 0x0627 -> 0x062B -> 0x062F -> 0x05F3 -> 0x05F7 -> 0x05FB -> 0x0633 -> 0x05DB -> 0x0637 -> 0x0603 -> 0x05E7 -> 0x05EF -> 0x05BF -> 0x0607 -> 0x060B -> 0x060F -> 0x063B -> 0x0613 -> 0x05C7 -> 0x05CF -> 0x05D7 -> 0x05FF -> 0x05DF -> 0x05AF -> 0x06D1 -> 0x06D5 -> 0x06D9 -> 0x06DD -> 0x06E1 -> 0x06E5 -> 0x06E9 -> 0x06ED -> 0x06F1 -> 0x06F5 -> 0x06F9 -> 0x06FD -> 0x0701 -> 0x0705 -> 0x0709 -> 0x070D -> 0x0711 -> 0x0715 -> 0x0719 -> 0x071D -> 0x0721 -> 0x0725 -> 0x0729 -> 0x072D -> 0x0731 -> 0x0735 -> 0x0739 -> 0x073D -> 0x0741 -> 0x0745 -> 0x0749 -> 0x074D -> 0x0751 -> 0x0755 -> 0x0759 -> 0x075D -> 0x0761 -> 0x0765 -> 0x0769 -> 0x076D -> 0x0771 -> 0x0775 -> 0x0779 -> 0x077D -> 0x0781 -> 0x0785 -> 0x0789 -> 0x078D -> 0x0791 -> 0x0795 -> 0x0799 -> 0x079D -> 0x07A1 -> 0x07A5 -> 0x07A9 -> 0x07AD -> 0x07B1 -> 0x07B5 -> 0x07B9 -> 0x07BD -> 0x07C1 -> 0x07C5 -> 0x07C9 -> 0x07CD -> 0x07D1 -> 0x07D5 -> 0x07E6 -> 0x07EA -> 0x0893 -> 0x086B -> 0x083B -> 0x082F -> 0x0843 -> 0x086F -> 0x0873 -> 0x0877 -> 0x084B -> 0x0853 -> 0x0857 -> 0x087B -> 0x085B -> 0x087F -> 0x0883 -> 0x0887 -> 0x088B -> 0x088F -> 0x082B -> 0x08AB -> 0x08C0 -> 0x08C4 -> 0x0971 -> 0x0965 -> 0x0921 -> 0x08F1 -> 0x0949 -> 0x094D -> 0x0951 -> 0x0955 -> 0x08E5 -> 0x0959 -> 0x095D -> 0x0961 -> 0x0925 -> 0x0929 -> 0x092D -> 0x090D -> 0x0969 -> 0x0935 -> 0x0919 -> 0x0939 -> 0x093D -> 0x0941 -> 0x096D -> 0x0945 -> 0x08F9 -> 0x0901 -> 0x0909 -> 0x0931 -> 0x0911 -> 0x08E1 -> 0x0988 -> 0x099C -> 0x09A0 -> 0x0A29 -> 0x0A4D -> 0x09FD -> 0x0A51 -> 0x0A55 -> 0x0A41 -> 0x09CD -> 0x0A25 -> 0x0A2D -> 0x0A31 -> 0x09C1 -> 0x0A35 -> 0x0A39 -> 0x0A3D -> 0x0A01 -> 0x0A05 -> 0x0A09 -> 0x09E9 -> 0x0A45 -> 0x0A11 -> 0x09F5 -> 0x0A15 -> 0x0A19 -> 0x0A1D -> 0x0A49 -> 0x0A21 -> 0x09D5 -> 0x09DD -> 0x09E5 -> 0x0A0D -> 0x09ED -> 0x09BD -> 0x0A69 -> 0x0A7A -> 0x0A7E -> 0x0D6E -> 0x0DD4 -> 0x0E16 -> 0x0E54 -> 0x0EC8 -> 0x0F07 -> 0x0F17 -> 0x0F7B -> 0x1032 -> 0x104E -> 0x105F -> 0x10AC -> 0x1163 -> 0x1192 -> 0x1263 -> 0x1287 -> 0x1384 -> 0x13E9 -> 0x1592 -> 0x15B2 -> 0x1606 -> 0x1619 -> 0x162C -> 0x16F2 -> 0x1731 -> 0x176B -> 0x1775 -> 0x18DC -> 0x18EE -> 0x1924 -> 0x1939 -> 0x1993 -> 0x1B7D -> 0x1B9F -> 0x1BE5 -> 0x1C59 -> 0x1C71 -> 0x1CA8 -> 0x1CB2 -> 0x1CDC -> 0x1CF6 -> 0x1D19 -> 0x1D3A -> 0x1D4D -> 0x1D67 -> 0x1D9E -> 0x1DC6 -> 0x1DE5 -> 0x1E6C -> 0x1E9B -> 0x1ECF -> 0x1F11 -> 0x1F60 -> 0x1F8A -> 0x1F97 -> 0x1FBE -> 0x1FD4 -> 0x1FDF -> 0x20D2 -> 0x2156 -> 0x21DD -> 0x21F0 -> 0x220A -> 0x2252 -> 0x2290 -> 0x22C3 -> 0x22E2 -> 0x2335 -> 0x234F -> 0x23A9 -> 0x23E4 -> 0x23FF -> 0x253C -> 0x254A -> 0x2595 -> 0x26FE -> 0x270B -> 0x27F1 -> 0x2816 -> 0x2867 -> 0x287E -> 0x28BF -> 0x28D6 -> 0x28FB -> 0x2922 -> 0x2978 -> 0x29EE -> 0x2A04 -> 0x2A1B -> 0x2A47 -> 0x2A73 -> 0x2A9F -> 0x2BA8 -> 0x2BFA -> 0x2C24 -> 0x2C4E -> 0x2C71 -> 0x2C81 -> 0x2C9B -> 0x2CCD -> 0x2D2D -> 0x2D59 -> 0x2D88 -> 0x2DD0 -> 0x2DFA -> 0x2E36 -> 0x2E41 -> 0x2ED1 -> 0x2EEF -> 0x2F08 -> 0x2F1E -> 0x2F69 -> 0x2F7E -> 0x3074 -> 0x309D -> 0x30AA -> 0x30C8 -> 0x30DE -> 0x30E9 -> 0x3106 -> 0x3110 -> 0x3122 -> 0x3142 -> 0x3394 -> 0x3404 -> 0x342B -> 0x3531 -> 0x354F -> 0x3573 -> 0x35E4 -> 0x35F2 -> 0x3614 -> 0x3671 -> 0x36A6 -> 0x36BB -> 0x374F -> 0x37B6 -> 0x381D -> 0x383D -> 0x3850 -> 0x3863 -> 0x386F -> 0x388D -> 0x3897 -> 0x38A3 -> 0x38C3 -> 0x38D7 -> 0x38F4 -> 0x38FE -> 0x390A -> 0x3922 -> 0x3932 -> 0x393C -> 0x3A02 -> 0x3A27 -> 0x3A7D -> 0x3BE0 -> 0x3CAA -> 0x3CE9 -> 0x3D25 -> 0x3D73 -> 0x3DA5 -> 0x3DC4 -> 0x3E36 -> 0x3E8B -> 0x3F8B -> 0x3FF3 -> 0x4021 -> 0x402E -> 0x4038 -> 0x4061 -> 0x40EB -> 0x412E -> 0x41B9 -> 0x41C3 -> 0x41CD -> 0x4317 -> 0x4346 -> 0x439E -> 0x43EC -> 0x4455 -> 0x4478 -> 0x4486 -> 0x44B8 -> 0x44D6 -> 0x44EF -> 0x452C -> 0x4583 -> 0x4618 -> 0x462C -> 0x4660 -> 0x467C -> 0x469A -> 0x479D -> 0x47BD -> 0x490A -> 0x4A4A -> 0x4A71 -> 0x4AB9 -> 0x4ADC -> 0x4AF6 -> 0x4B10 -> 0x4D11 -> 0x4D1B -> 0x4D55 -> 0x4D66 -> 0x4D82 -> 0x4DA5 -> 0x4E12 -> 0x4E20 -> 0x4F9A -> 0x5069 -> 0x5095 -> 0x512E -> 0x518A -> 0x51B6 -> 0x524B -> 0x5296 -> 0x5315 -> 0x5497 -> 0x54A4 -> 0x54D0 -> 0x556B -> 0x55A9 -> 0x55BD -> 0x55D3 -> 0x55E9 -> 0x55FF -> 0x5623 -> 0x5634 -> 0x5684 -> 0x568E -> 0x56C3 -> 0x56E7 -> 0x56FE -> 0x5708 -> 0x572B -> 0x5738 -> 0x5760 -> 0x5776 -> 0x5785 -> 0x57A7 -> 0x57C6 -> 0x57D3 -> 0x57F9 -> 0x5835 -> 0x584B -> 0x5861 -> 0x5877 -> 0x59C8 -> 0x5A54 -> 0x5A6B -> 0x5AA6 -> 0x5B15 -> 0x5B4E -> 0x5B89 -> 0x5BAA -> 0x5C04 -> 0x5C4B -> 0x5CA5 -> 0x5CCF -> 0x5CF7 -> 0x5D09 -> 0x5D2F -> 0x5D4F -> 0x5D5C -> 0x5D7A -> 0x5D8A -> 0x5DA0 -> 0x5DB3 -> 0x5DC1 -> 0x5DE4 -> 0x5E06 -> 0x5E4F -> 0x5EB6 -> 0x5ED9 -> 0x5EEB -> 0x5F5B -> 0x6055 -> 0x6070 -> 0x6096 -> 0x60C6 -> 0x60F8 -> 0x613F -> 0x625A -> 0x6278 -> 0x62A2 -> 0x62C4 -> 0x6301 -> 0x6353 -> 0x639D -> 0x6435 -> 0x6459 -> 0x6628 -> 0x663A -> 0x667A -> 0x670A -> 0x67A4 -> 0x67F3 -> 0x6815 -> 0x685B -> 0x6864 -> 0x688A -> 0x68C0 -> 0x68D5 -> 0x696B -> 0x697A -> 0x69E5 -> 0x6A4E -> 0x6ADD -> 0x6B07 -> 0x6B24 -> 0x6B34 -> 0x6B51 -> 0x6B7C -> 0x6B8B -> 0x6C0B -> 0x6C47 -> 0x6C50 -> 0x6C76 -> 0x6CAC -> 0x6CCD -> 0x6CD7 -> 0x6E6E -> 0x6E7B -> 0x6E83 -> 0x6E8B

================================================================================
SEGMENTO 76: FUENTE 0x18E2
================================================================================

registro     : 4
tipo         : 02
flags        : 00
inicio cadena: 0x0098
segmento destino: 78
cadena:
0x0098 -> 0x009C -> 0x00A0 -> 0x03AB -> 0x03BB -> 0x03BF -> 0x03C3 -> 0x039B -> 0x03A3 -> 0x0387 -> 0x04E0 -> 0x05D3 -> 0x05E3 -> 0x05EB -> 0x05C3 -> 0x05CB -> 0x07FB -> 0x084F -> 0x085F -> 0x0863 -> 0x0867 -> 0x083F -> 0x0847 -> 0x0905 -> 0x0915 -> 0x091D -> 0x08F5 -> 0x08FD -> 0x09E1 -> 0x09F1 -> 0x09F9 -> 0x09D1 -> 0x09D9 -> 0x11E0 -> 0x1204 -> 0x121E -> 0x1258 -> 0x130C -> 0x1345 -> 0x13BD -> 0x14B5 -> 0x15E4 -> 0x1647 -> 0x18E2 -> 0x192D -> 0x1967 -> 0x1B6D -> 0x1BD0 -> 0x205A -> 0x20A5 -> 0x20C1 -> 0x212A -> 0x26BD -> 0x26DD -> 0x2EBB -> 0x2F9D -> 0x2FD4 -> 0x2FFC -> 0x302E -> 0x3059 -> 0x3172 -> 0x33AA -> 0x33F6 -> 0x349B -> 0x34C8 -> 0x3502 -> 0x358A -> 0x3980 -> 0x399E -> 0x3C7E -> 0x3EDB -> 0x3EFB -> 0x3FE8 -> 0x40E0 -> 0x429A -> 0x42BE -> 0x4715 -> 0x4836 -> 0x4870 -> 0x48B5 -> 0x495E -> 0x49CF -> 0x4A1E -> 0x5FAD -> 0x5FF5 -> 0x6011 -> 0x637E -> 0x63F1 -> 0x640D -> 0x6602 -> 0x6618 -> 0x664F -> 0x6697 -> 0x66FA -> 0x6E0A -> 0x6E25 -> 0x6E92

================================================================================
SEGMENTO 76: FUENTE 0x18EE
================================================================================

registro     : 2
tipo         : 02
flags        : 00
inicio cadena: 0x00AC
segmento destino: 76
cadena:
0x00AC -> 0x0094 -> 0x00CF -> 0x00F5 -> 0x013B -> 0x038B -> 0x039F -> 0x03CB -> 0x03CF -> 0x03D3 -> 0x03A7 -> 0x03AF -> 0x03B3 -> 0x03D7 -> 0x03B7 -> 0x03DB -> 0x03C7 -> 0x0397 -> 0x03DF -> 0x03E3 -> 0x03E7 -> 0x03EB -> 0x0442 -> 0x0446 -> 0x044A -> 0x044E -> 0x0452 -> 0x0456 -> 0x045A -> 0x045E -> 0x0462 -> 0x0466 -> 0x046A -> 0x046E -> 0x0472 -> 0x0476 -> 0x047A -> 0x047E -> 0x0482 -> 0x0486 -> 0x048A -> 0x048E -> 0x0492 -> 0x0496 -> 0x049A -> 0x049E -> 0x04A2 -> 0x04A6 -> 0x04AA -> 0x04AE -> 0x04B2 -> 0x04B6 -> 0x04BA -> 0x04BE -> 0x04C2 -> 0x04C6 -> 0x04CA -> 0x04CE -> 0x04DC -> 0x04F9 -> 0x0516 -> 0x0532 -> 0x0550 -> 0x0567 -> 0x056F -> 0x058A -> 0x058E -> 0x0617 -> 0x061B -> 0x061F -> 0x0623 -> 0x05B3 -> 0x0627 -> 0x062B -> 0x062F -> 0x05F3 -> 0x05F7 -> 0x05FB -> 0x0633 -> 0x05DB -> 0x0637 -> 0x0603 -> 0x05E7 -> 0x05EF -> 0x05BF -> 0x0607 -> 0x060B -> 0x060F -> 0x063B -> 0x0613 -> 0x05C7 -> 0x05CF -> 0x05D7 -> 0x05FF -> 0x05DF -> 0x05AF -> 0x06D1 -> 0x06D5 -> 0x06D9 -> 0x06DD -> 0x06E1 -> 0x06E5 -> 0x06E9 -> 0x06ED -> 0x06F1 -> 0x06F5 -> 0x06F9 -> 0x06FD -> 0x0701 -> 0x0705 -> 0x0709 -> 0x070D -> 0x0711 -> 0x0715 -> 0x0719 -> 0x071D -> 0x0721 -> 0x0725 -> 0x0729 -> 0x072D -> 0x0731 -> 0x0735 -> 0x0739 -> 0x073D -> 0x0741 -> 0x0745 -> 0x0749 -> 0x074D -> 0x0751 -> 0x0755 -> 0x0759 -> 0x075D -> 0x0761 -> 0x0765 -> 0x0769 -> 0x076D -> 0x0771 -> 0x0775 -> 0x0779 -> 0x077D -> 0x0781 -> 0x0785 -> 0x0789 -> 0x078D -> 0x0791 -> 0x0795 -> 0x0799 -> 0x079D -> 0x07A1 -> 0x07A5 -> 0x07A9 -> 0x07AD -> 0x07B1 -> 0x07B5 -> 0x07B9 -> 0x07BD -> 0x07C1 -> 0x07C5 -> 0x07C9 -> 0x07CD -> 0x07D1 -> 0x07D5 -> 0x07E6 -> 0x07EA -> 0x0893 -> 0x086B -> 0x083B -> 0x082F -> 0x0843 -> 0x086F -> 0x0873 -> 0x0877 -> 0x084B -> 0x0853 -> 0x0857 -> 0x087B -> 0x085B -> 0x087F -> 0x0883 -> 0x0887 -> 0x088B -> 0x088F -> 0x082B -> 0x08AB -> 0x08C0 -> 0x08C4 -> 0x0971 -> 0x0965 -> 0x0921 -> 0x08F1 -> 0x0949 -> 0x094D -> 0x0951 -> 0x0955 -> 0x08E5 -> 0x0959 -> 0x095D -> 0x0961 -> 0x0925 -> 0x0929 -> 0x092D -> 0x090D -> 0x0969 -> 0x0935 -> 0x0919 -> 0x0939 -> 0x093D -> 0x0941 -> 0x096D -> 0x0945 -> 0x08F9 -> 0x0901 -> 0x0909 -> 0x0931 -> 0x0911 -> 0x08E1 -> 0x0988 -> 0x099C -> 0x09A0 -> 0x0A29 -> 0x0A4D -> 0x09FD -> 0x0A51 -> 0x0A55 -> 0x0A41 -> 0x09CD -> 0x0A25 -> 0x0A2D -> 0x0A31 -> 0x09C1 -> 0x0A35 -> 0x0A39 -> 0x0A3D -> 0x0A01 -> 0x0A05 -> 0x0A09 -> 0x09E9 -> 0x0A45 -> 0x0A11 -> 0x09F5 -> 0x0A15 -> 0x0A19 -> 0x0A1D -> 0x0A49 -> 0x0A21 -> 0x09D5 -> 0x09DD -> 0x09E5 -> 0x0A0D -> 0x09ED -> 0x09BD -> 0x0A69 -> 0x0A7A -> 0x0A7E -> 0x0D6E -> 0x0DD4 -> 0x0E16 -> 0x0E54 -> 0x0EC8 -> 0x0F07 -> 0x0F17 -> 0x0F7B -> 0x1032 -> 0x104E -> 0x105F -> 0x10AC -> 0x1163 -> 0x1192 -> 0x1263 -> 0x1287 -> 0x1384 -> 0x13E9 -> 0x1592 -> 0x15B2 -> 0x1606 -> 0x1619 -> 0x162C -> 0x16F2 -> 0x1731 -> 0x176B -> 0x1775 -> 0x18DC -> 0x18EE -> 0x1924 -> 0x1939 -> 0x1993 -> 0x1B7D -> 0x1B9F -> 0x1BE5 -> 0x1C59 -> 0x1C71 -> 0x1CA8 -> 0x1CB2 -> 0x1CDC -> 0x1CF6 -> 0x1D19 -> 0x1D3A -> 0x1D4D -> 0x1D67 -> 0x1D9E -> 0x1DC6 -> 0x1DE5 -> 0x1E6C -> 0x1E9B -> 0x1ECF -> 0x1F11 -> 0x1F60 -> 0x1F8A -> 0x1F97 -> 0x1FBE -> 0x1FD4 -> 0x1FDF -> 0x20D2 -> 0x2156 -> 0x21DD -> 0x21F0 -> 0x220A -> 0x2252 -> 0x2290 -> 0x22C3 -> 0x22E2 -> 0x2335 -> 0x234F -> 0x23A9 -> 0x23E4 -> 0x23FF -> 0x253C -> 0x254A -> 0x2595 -> 0x26FE -> 0x270B -> 0x27F1 -> 0x2816 -> 0x2867 -> 0x287E -> 0x28BF -> 0x28D6 -> 0x28FB -> 0x2922 -> 0x2978 -> 0x29EE -> 0x2A04 -> 0x2A1B -> 0x2A47 -> 0x2A73 -> 0x2A9F -> 0x2BA8 -> 0x2BFA -> 0x2C24 -> 0x2C4E -> 0x2C71 -> 0x2C81 -> 0x2C9B -> 0x2CCD -> 0x2D2D -> 0x2D59 -> 0x2D88 -> 0x2DD0 -> 0x2DFA -> 0x2E36 -> 0x2E41 -> 0x2ED1 -> 0x2EEF -> 0x2F08 -> 0x2F1E -> 0x2F69 -> 0x2F7E -> 0x3074 -> 0x309D -> 0x30AA -> 0x30C8 -> 0x30DE -> 0x30E9 -> 0x3106 -> 0x3110 -> 0x3122 -> 0x3142 -> 0x3394 -> 0x3404 -> 0x342B -> 0x3531 -> 0x354F -> 0x3573 -> 0x35E4 -> 0x35F2 -> 0x3614 -> 0x3671 -> 0x36A6 -> 0x36BB -> 0x374F -> 0x37B6 -> 0x381D -> 0x383D -> 0x3850 -> 0x3863 -> 0x386F -> 0x388D -> 0x3897 -> 0x38A3 -> 0x38C3 -> 0x38D7 -> 0x38F4 -> 0x38FE -> 0x390A -> 0x3922 -> 0x3932 -> 0x393C -> 0x3A02 -> 0x3A27 -> 0x3A7D -> 0x3BE0 -> 0x3CAA -> 0x3CE9 -> 0x3D25 -> 0x3D73 -> 0x3DA5 -> 0x3DC4 -> 0x3E36 -> 0x3E8B -> 0x3F8B -> 0x3FF3 -> 0x4021 -> 0x402E -> 0x4038 -> 0x4061 -> 0x40EB -> 0x412E -> 0x41B9 -> 0x41C3 -> 0x41CD -> 0x4317 -> 0x4346 -> 0x439E -> 0x43EC -> 0x4455 -> 0x4478 -> 0x4486 -> 0x44B8 -> 0x44D6 -> 0x44EF -> 0x452C -> 0x4583 -> 0x4618 -> 0x462C -> 0x4660 -> 0x467C -> 0x469A -> 0x479D -> 0x47BD -> 0x490A -> 0x4A4A -> 0x4A71 -> 0x4AB9 -> 0x4ADC -> 0x4AF6 -> 0x4B10 -> 0x4D11 -> 0x4D1B -> 0x4D55 -> 0x4D66 -> 0x4D82 -> 0x4DA5 -> 0x4E12 -> 0x4E20 -> 0x4F9A -> 0x5069 -> 0x5095 -> 0x512E -> 0x518A -> 0x51B6 -> 0x524B -> 0x5296 -> 0x5315 -> 0x5497 -> 0x54A4 -> 0x54D0 -> 0x556B -> 0x55A9 -> 0x55BD -> 0x55D3 -> 0x55E9 -> 0x55FF -> 0x5623 -> 0x5634 -> 0x5684 -> 0x568E -> 0x56C3 -> 0x56E7 -> 0x56FE -> 0x5708 -> 0x572B -> 0x5738 -> 0x5760 -> 0x5776 -> 0x5785 -> 0x57A7 -> 0x57C6 -> 0x57D3 -> 0x57F9 -> 0x5835 -> 0x584B -> 0x5861 -> 0x5877 -> 0x59C8 -> 0x5A54 -> 0x5A6B -> 0x5AA6 -> 0x5B15 -> 0x5B4E -> 0x5B89 -> 0x5BAA -> 0x5C04 -> 0x5C4B -> 0x5CA5 -> 0x5CCF -> 0x5CF7 -> 0x5D09 -> 0x5D2F -> 0x5D4F -> 0x5D5C -> 0x5D7A -> 0x5D8A -> 0x5DA0 -> 0x5DB3 -> 0x5DC1 -> 0x5DE4 -> 0x5E06 -> 0x5E4F -> 0x5EB6 -> 0x5ED9 -> 0x5EEB -> 0x5F5B -> 0x6055 -> 0x6070 -> 0x6096 -> 0x60C6 -> 0x60F8 -> 0x613F -> 0x625A -> 0x6278 -> 0x62A2 -> 0x62C4 -> 0x6301 -> 0x6353 -> 0x639D -> 0x6435 -> 0x6459 -> 0x6628 -> 0x663A -> 0x667A -> 0x670A -> 0x67A4 -> 0x67F3 -> 0x6815 -> 0x685B -> 0x6864 -> 0x688A -> 0x68C0 -> 0x68D5 -> 0x696B -> 0x697A -> 0x69E5 -> 0x6A4E -> 0x6ADD -> 0x6B07 -> 0x6B24 -> 0x6B34 -> 0x6B51 -> 0x6B7C -> 0x6B8B -> 0x6C0B -> 0x6C47 -> 0x6C50 -> 0x6C76 -> 0x6CAC -> 0x6CCD -> 0x6CD7 -> 0x6E6E -> 0x6E7B -> 0x6E83 -> 0x6E8B

################################################################################
# CALLS FAR DENTRO DE 76:18F4
################################################################################

################################################################################
# TODAS LAS RELOCACIONES INTERNAS HACIA SEGMENTO 80
################################################################################

SEGMENTO FUENTE : 38
REGISTRO        : 0
INICIO CADENA   : 0x0017
CADENA:
0x0017 -> 0x003F -> 0x01C4 -> 0x01D9 -> 0x0209 -> 0x0328 -> 0x03B3 -> 0x03BF -> 0x03D5 -> 0x040B -> 0x0444 -> 0x0666 -> 0x0671 -> 0x07A8 -> 0x088C -> 0x089C -> 0x08AD -> 0x099C -> 0x0A5A -> 0x0A93 -> 0x0AEC -> 0x0B3F -> 0x0B89 -> 0x0C20 -> 0x0C6E -> 0x0D09 -> 0x0D64 -> 0x0DC8 -> 0x0E8C -> 0x1013 -> 0x1084 -> 0x12A6 -> 0x12CA -> 0x13F9 -> 0x141B -> 0x1443 -> 0x14B5 -> 0x14CA -> 0x14DB -> 0x14F0 -> 0x150F -> 0x1536 -> 0x1769 -> 0x1773 -> 0x182D -> 0x190F -> 0x1A44 -> 0x1B01 -> 0x1B54 -> 0x1BAF -> 0x1BD4 -> 0x1BE9 -> 0x1BFA -> 0x1C0F -> 0x1C29 -> 0x1EEF -> 0x1F19 -> 0x216C -> 0x2176 -> 0x2230 -> 0x2312 -> 0x2447 -> 0x250D -> 0x2562 -> 0x259B -> 0x263B -> 0x268B -> 0x2711 -> 0x2792 -> 0x28A1 -> 0x28BC -> 0x29B6 -> 0x29D3 -> 0x29F5 -> 0x2A17 -> 0x2A45 -> 0x2A9E -> 0x2B74 -> 0x2B8A -> 0x2BC2 -> 0x2BD4 -> 0x2D44 -> 0x2EE9 -> 0x2F68 -> 0x2F83 -> 0x2FE6 -> 0x3113 -> 0x3129 -> 0x3141 -> 0x3193 -> 0x31A2 -> 0x31B1 -> 0x320F -> 0x3226 -> 0x3289 -> 0x3297 -> 0x32DA -> 0x3392 -> 0x362B -> 0x3636 -> 0x363B -> 0x364F -> 0x3654 -> 0x3659 -> 0x368D -> 0x36A0 -> 0x36B3 -> 0x36B8 -> 0x36BD -> 0x36DB -> 0x36E0 -> 0x36E5 -> 0x373C -> 0x3755 -> 0x376E -> 0x377C -> 0x378A -> 0x378F -> 0x3794 -> 0x37AC -> 0x37B1 -> 0x37C1 -> 0x37CC -> 0x37D1 -> 0x3805 -> 0x3818 -> 0x382B -> 0x3830 -> 0x3835 -> 0x387C -> 0x389B -> 0x38E3 -> 0x38F1 -> 0x38FF -> 0x3904 -> 0x3909 -> 0x3921 -> 0x3926 -> 0x3936 -> 0x3941 -> 0x3946 -> 0x3976 -> 0x397B -> 0x3980 -> 0x3992 -> 0x3997 -> 0x399C -> 0x39AE -> 0x39B3 -> 0x39B8 -> 0x39DD -> 0x39F1 -> 0x39FF -> 0x3A0D -> 0x3A12 -> 0x3A17 -> 0x3A3C -> 0x3A50 -> 0x3A5E -> 0x3A6C -> 0x3A71 -> 0x3A76 -> 0x3A9B -> 0x3AAF -> 0x3ABD -> 0x3ACB -> 0x3AD0 -> 0x3AD5 -> 0x3AED -> 0x3AF2 -> 0x3B13 -> 0x3B67 -> 0x3B7E -> 0x3BC0 -> 0x3C22 -> 0x3C61 -> 0x3CEC -> 0x3D66 -> 0x3D7C -> 0x3D8A -> 0x3D8F -> 0x3DCB -> 0x3DE2 -> 0x3E0B -> 0x3E2C -> 0x3E78 -> 0x3E84 -> 0x3E91 -> 0x3E9E -> 0x3EAB -> 0x3EB7 -> 0x3EC4 -> 0x3ED1 -> 0x3EDE -> 0x3EEA -> 0x3EF7 -> 0x3EFC -> 0x3F01 -> 0x3F1A -> 0x3F1F -> 0x3F78 -> 0x3F8C -> 0x3F97 -> 0x3F9C -> 0x4104 -> 0x4112 -> 0x4120 -> 0x4125 -> 0x412A -> 0x418B -> 0x4190 -> 0x41B5

SEGMENTO FUENTE : 61
REGISTRO        : 0
INICIO CADENA   : 0x000B
CADENA:
0x000B -> 0x00A2 -> 0x00F7 -> 0x0121 -> 0x01A5 -> 0x01B8 -> 0x01D1 -> 0x01E4 -> 0x01EE -> 0x0207 -> 0x021A -> 0x0224 -> 0x022E -> 0x0247 -> 0x025A -> 0x0274 -> 0x02AD -> 0x02C3 -> 0x02DD -> 0x0316 -> 0x032C -> 0x0346 -> 0x037F -> 0x03D3 -> 0x042A -> 0x0444 -> 0x046A -> 0x0480 -> 0x049A -> 0x04C0 -> 0x04D6 -> 0x04F0 -> 0x0516 -> 0x0552 -> 0x05A9 -> 0x05C3 -> 0x05D9 -> 0x05F3 -> 0x0609 -> 0x0623 -> 0x063C -> 0x0650 -> 0x0664 -> 0x0687 -> 0x06FF -> 0x07AF -> 0x07C4 -> 0x07D9 -> 0x07F1 -> 0x07FB -> 0x08B8 -> 0x0919 -> 0x0944 -> 0x0950 -> 0x0AA9 -> 0x0ABE -> 0x0AD3 -> 0x0B04 -> 0x0D49 -> 0x0D8B -> 0x0DF6 -> 0x0E68 -> 0x0EC3 -> 0x0EE2 -> 0x0F49 -> 0x0F55 -> 0x0F78 -> 0x0F9B -> 0x0FEB -> 0x0FFD -> 0x1007 -> 0x10FA -> 0x110C -> 0x1116 -> 0x11EC -> 0x125E -> 0x12A7 -> 0x162A -> 0x1743 -> 0x1772 -> 0x17BA -> 0x17D4 -> 0x17DC -> 0x17EE -> 0x1808 -> 0x1810 -> 0x18EB -> 0x1944 -> 0x195E -> 0x1966 -> 0x1978 -> 0x1992 -> 0x199A -> 0x1B24 -> 0x1B5B -> 0x1C2F -> 0x1C3F -> 0x1C64 -> 0x1C89 -> 0x1CAE -> 0x1CD3 -> 0x1CF8 -> 0x1D1D -> 0x1D42 -> 0x1D67 -> 0x1D8C -> 0x1DB1 -> 0x1DD6 -> 0x1DFB -> 0x1E20 -> 0x1E45 -> 0x1E6A -> 0x1E8F -> 0x1EB4 -> 0x1ED9 -> 0x1EFE -> 0x1F23 -> 0x1F48 -> 0x1F6D -> 0x1F92 -> 0x1FB7 -> 0x1FDC -> 0x2001 -> 0x2044 -> 0x276C -> 0x2776 -> 0x282F -> 0x29A4 -> 0x29D9 -> 0x29E3 -> 0x2A0E -> 0x2A18 -> 0x2AF1 -> 0x2DE4 -> 0x2E13 -> 0x2E1E -> 0x2E23 -> 0x2E58 -> 0x2E82 -> 0x2E99 -> 0x2EB0 -> 0x2EC0 -> 0x2ED4 -> 0x2ED9 -> 0x2EE0 -> 0x2EE5 -> 0x2EEA -> 0x2F09 -> 0x2F1F -> 0x2F2A -> 0x2F31 -> 0x2F36 -> 0x2F3B -> 0x2F4D -> 0x2F63 -> 0x2F6E -> 0x2F75 -> 0x2F7A -> 0x2F7F -> 0x2F91 -> 0x2FA7 -> 0x2FB2 -> 0x2FB9 -> 0x2FBE -> 0x2FC3 -> 0x2FD5 -> 0x2FEB -> 0x2FF6 -> 0x2FFD -> 0x3002 -> 0x3007 -> 0x3019 -> 0x302F -> 0x303A -> 0x3041 -> 0x3046 -> 0x304B -> 0x3056 -> 0x305B -> 0x307C -> 0x308D -> 0x30D7 -> 0x30D3 -> 0x30CF -> 0x3100 -> 0x30FC -> 0x30F8 -> 0x30F4 -> 0x31C3 -> 0x3241 -> 0x323D -> 0x3239 -> 0x3235 -> 0x3265 -> 0x3272 -> 0x3309 -> 0x338B -> 0x3387 -> 0x3383 -> 0x3431 -> 0x3515 -> 0x3522 -> 0x3584 -> 0x35A4 -> 0x35AF -> 0x3CC1 -> 0x3E20 -> 0x3E38 -> 0x3E4A -> 0x3E4F -> 0x3EA6 -> 0x3F51 -> 0x3F67 -> 0x3F89 -> 0x3FA8

################################################################################
# RELOCACIONES QUE AFECTAN DIRECTAMENTE 80:0444
################################################################################

################################################################################
# CONTEXTO BINARIO DE 80:0444
################################################################################

80:0404 - 80:04C4
0404: 06 20 25 CB 83 3E 20 25 00 75 01 CB B8 00 00 E9
0414: 4A FC 8B F4 36 8E 44 02 26 3B 55 02 7F 07 7C 14
0424: 26 3B 05 72 0F 26 3B 55 06 7C 08 7F 07 26 3B 45
0434: 04 77 01 CB B8 04 00 E9 22 FC B8 05 00 E9 1C FC
0444: 05 00 04 72 19 2B C4 73 15 F7 D8 36 3B 06 0A 00
0454: 72 0C 36 3B 06 0C 00 73 04 36 A3 0C 00 CB B8 CA
0464: 00 E9 27 FC C6 06 44 25 01 2E 80 3E AF 04 CD 74
0474: 3D 55 8B EC 83 EC 0A 50 DB 7E F6 DD 06 84 25 9B
0484: DC 36 8C 25 9B DC 0E 8C 25 9B DC 2E 84 25 9B DC
0494: 1E 94 25 9B 9B DF E0 9B 25 00 01 C1 E8 07 48 A2
04A4: 44 25 DB 6E F6 58 8B E5 5D EB 03 9B D9 C9 80 3E
04B4: 44 25 00 7E 04 9B DE F9 CB 74 A9 55 8B EC 83 EC

================================================================================
FIN DE LA INVESTIGACION
================================================================================
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Perfecto. Ahora sí tenemos una corrección importante y bastante información nueva.


## 1. Los `CALL FAR` de `76:18A9` ya quedaron resueltos


La cadena de relocación nos da:


| CALL | Destino real |
| --- | --- |
| `76:18D9` | **76:18F4** |
| `76:18DF` | **78:066E** |
| `76:18EB` | **76:1BEB** |


Es decir, la rutina `76:18A9` hace exactamente:


```
76:18A9
   │
   ├──→ 76:18F4
   │
   ├──→ 78:066E
   │
   └──→ 76:1BEB
```


Esto es mucho más útil que el resultado anterior.


---


## 2. Hay una corrección importante sobre `76:18F4`


El desensamblado que obtuvimos demuestra que **`76:18F4` termina antes de `76:1921`**.


Por tanto:


```
76:18F4 ───────────────┐
                       │
                       └── termina aproximadamente en 1914/1920

76:1921 = OTRA RUTINA
```


Los `CALL FAR` que aparecen en el segundo bloque de tu salida:


```
asmpush bp
mov bp,sp
...
lcall ...
...
lretw $6
```


**no pertenecen a `76:18F4`**. Son el comienzo de la siguiente rutina, `76:1921`.


Eso corrige una interpretación anterior nuestra.


---


# 3. `80:0444` también queda mucho más claro


El código es:


```
asm80:0444
    add ax,0400h
    jb  ...
    sub ax,sp
    jae ...
    neg ax
    cmp ss:[000A],ax
    jb  ...
    cmp ss:[000C],ax
    jae ...
    mov ss:[000C],ax
    retf
```


Esto **no tiene apariencia de código de ARMÓN**.


Es prácticamente seguro que estamos ante una **rutina de soporte del runtime de Borland/Delphi**, relacionada con comprobación/gestión del espacio de pila.


Y eso explica perfectamente que `CalcularClick` haga:


```
asmmov ax,8
call far 80:0444
```


antes de comenzar su trabajo real.


Por tanto, podemos eliminar `80:0444` de nuestra búsqueda de la lógica astrológica.


La estructura real de `CalcularClick` comienza después de esa comprobación:


```
asmCalcularClick:

    mov ax,8
    call 80:0444          ; runtime / stack check

    sub sp,8

    ...
    call 76:18F4

    ...
    call 76:18A9
```


---


# 4. Hay otro detalle muy importante


La enorme cadena que apareció, por ejemplo:


```
0x00AC -> 0x0094 -> 0x00CF -> ...
...
0x18DC -> 0x18EE -> ...
```


**no significa que el programa ejecute todas esas direcciones en secuencia**.


Es simplemente la **cadena de posiciones que utilizan una misma tabla de relocación**.


Es decir:


```
cadena de relocación ≠ flujo de ejecución
```


Esto es fundamental para no interpretar erróneamente el ejecutable.


---


# 5. Lo verdaderamente interesante ahora


Tenemos:


```
CalcularClick
     │
     ├──→ 80:0444
     │       └── runtime de pila
     │
     ├──→ 76:18F4
     │
     └──→ 76:18A9
             │
             ├──→ 76:18F4
             ├──→ 78:066E
             └──→ 76:1BEB
```


Y `76:18F4` tiene una estructura especialmente interesante:


```
asmenter 10,0

lea  di,[bp-10]
push ss
push di

les  di,[bp+6]
push es
push di

les  di,es:[di]

call far es:[di+34h]

add  sp,4

mov  ax,[bp-4]
mov  [bp-2],ax
mov  ax,[bp-2]

leave
lret 4
```


La secuencia:


```
asmles di,[bp+6]
...
les di,es:[di]
call far es:[di+34h]
```


es particularmente significativa.


Está siguiendo un **puntero a objeto y después accediendo a una entrada de su VMT (Virtual Method Table)**.


La expresión conceptual es aproximadamente:


```
objeto
   │
   └──→ VMT
          │
          └── offset +34h
                  │
                  └── método virtual
```


Esto encaja extraordinariamente bien con código generado por Delphi orientado a objetos.


---


# 6. Y `18A9` hace algo muy parecido


`76:18A9`:


```
asmenter 10,0

lea  di,[bp-10]
push ss
push di

les di,[bp+6]
push es
push di

les di,es:[di]

call far es:[di+34h]

add sp,4

mov ax,[bp-6]
mov [bp-2],ax
mov ax,[bp-2]

leave
lret 4
```


Es decir, ambos procedimientos utilizan:


```
asmcall far es:[di+34h]
```


La diferencia está en **qué posición de los 10 bytes temporales recuperan después**:


`18A9`:


```
asmmov ax,[bp-6]
```


`18F4`:


```
asmmov ax,[bp-4]
```


Esto merece estudiarse porque probablemente nos está mostrando cómo Delphi está manejando un tipo de dato de más de 16 bits mediante un parámetro oculto.


---


# 7. Ahora debemos dejar de mirar relocaciones y mirar los tres procedimientos


El siguiente paso correcto es analizar:


```
76:18F4
76:1BEB
78:066E
```


especialmente:


```
76:1BEB
78:066E
```


porque son los dos destinos que todavía no conocemos.


Ejecuta **un solo bloque**:


```
Bashcd /storage/emulated/0/Download/armon

python3 - <<'PY'
from pathlib import Path
import struct
import subprocess

data = Path("ARMON.EXE").read_bytes()

NE = 0x100
SEG_COUNT = struct.unpack_from("<H", data, NE + 0x1C)[0]
SEG_TABLE = NE + struct.unpack_from("<H", data, NE + 0x22)[0]
SHIFT = struct.unpack_from("<H", data, NE + 0x32)[0]

segments = {}

for n in range(1, SEG_COUNT + 1):
    p = SEG_TABLE + (n - 1) * 8

    sector = struct.unpack_from("<H", data, p)[0]
    length = struct.unpack_from("<H", data, p + 2)[0]
    flags  = struct.unpack_from("<H", data, p + 4)[0]

    size = 0x10000 if length == 0 else length
    fileoff = sector << SHIFT

    segments[n] = {
        "fileoff": fileoff,
        "size": size,
        "flags": flags
    }


def disasm(seg, start, end):

    s = segments[seg]

    blob = data[
        s["fileoff"] + start:
        s["fileoff"] + end
    ]

    print()
    print("=" * 80)
    print(f"SEGMENTO {seg}:{start:04X} - {seg}:{end:04X}")
    print("=" * 80)

    print()
    print("BYTES:")

    for p in range(0, len(blob), 16):

        row = blob[p:p+16]

        print(
            f"{start+p:04X}: "
            + " ".join(f"{x:02X}" for x in row)
        )

    print()
    print("DESENSAMBLADO:")
    print()

    text = " ".join(f"0x{x:02X}" for x in blob)

    r = subprocess.run(
        [
            "llvm-mc",
            "--disassemble",
            "--triple=i386-unknown-none-code16",
            "--x86-asm-syntax=intel"
        ],
        input=text + "\n",
        text=True,
        capture_output=True
    )

    print(r.stdout)

    if r.stderr:
        print("AVISOS:")
        print(r.stderr)


# ============================================================
# 1. 76:18F4
# ============================================================

disasm(
    76,
    0x18F4,
    0x1921
)

# ============================================================
# 2. 76:1BEB
#
# Mostrar bastante contexto para detectar el verdadero
# comienzo y final de la rutina.
# ============================================================

disasm(
    76,
    0x1B90,
    0x1C60
)

# ============================================================
# 3. 78:066E
#
# También mostramos contexto alrededor de la entrada.
# ============================================================

disasm(
    78,
    0x0620,
    0x0710
)

# ============================================================
# 4. Buscar prólogos Delphi cercanos
# ============================================================

print()
print("=" * 80)
print("PROLOGOS 55 8B EC CERCA DE LOS DESTINOS")
print("=" * 80)

for seg, target in [
    (76, 0x1BEB),
    (78, 0x066E)
]:

    s = segments[seg]

    inicio = max(0, target - 0x100)
    fin = min(s["size"] - 2, target + 0x20)

    print()
    print(f"SEGMENTO {seg}, destino {target:04X}")

    for off in range(inicio, fin):

        b0 = data[s["fileoff"] + off]
        b1 = data[s["fileoff"] + off + 1]
        b2 = data[s["fileoff"] + off + 2]

        if (
            b0 == 0x55 and
            b1 == 0x8B and
            b2 == 0xEC
        ):
            print(
                f"  posible procedimiento: "
                f"{seg}:{off:04X}"
            )

print()
print("=" * 80)
print("FIN")
print("=" * 80)
PY
```


Con ese resultado podremos hacer algo mucho más interesante: **seguir la cadena desde `CalcularClick` hasta las rutinas que realmente realizan el cálculo**, descartando primero todo el código de soporte de Delphi.


Y aquí ya aparece una pista fuerte: `80:0444` era infraestructura del runtime, mientras que `76:18A9 → 76:18F4 → 78:066E / 76:1BEB` parece entrar en la parte de objetos/tipos de datos que sí puede estar relacionada con los cálculos de ARMÓN.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> from pathlib import Path
> import struct
> import subprocess
>
> data = Path("ARMON.EXE").read_bytes()
>
> NE = 0x100
> SEG_COUNT = struct.unpack_from("<H", data, NE + 0x1C)[0]
> SEG_TABLE = NE + struct.unpack_from("<H", data, NE + 0x22)[0]
> SHIFT = struct.unpack_from("<H", data, NE + 0x32)[0]
>
> segments = {}
>
> for n in range(1, SEG_COUNT + 1):
>     p = SEG_TABLE + (n - 1) * 8
>
>     sector = struct.unpack_from("<H", data, p)[0]
>     length = struct.unpack_from("<H", data, p + 2)[0]
>     flags  = struct.unpack_from("<H", data, p + 4)[0]
>
>     size = 0x10000 if length == 0 else length
>     fileoff = sector << SHIFT
>
>     segments[n] = {
>         "fileoff": fileoff,
>         "size": size,
>         "flags": flags
>     }
>
>
> def disasm(seg, start, end):
>
>     s = segments[seg]
>
>     blob = data[
>         s["fileoff"] + start:
>         s["fileoff"] + end
>     ]
>
>     print()
>     print("=" * 80)
>     print(f"SEGMENTO {seg}:{start:04X} - {seg}:{end:04X}")
>     print("=" * 80)
>
>     print()
>     print("BYTES:")
>
>     for p in range(0, len(blob), 16):
>
>         row = blob[p:p+16]
>
>         print(
>             f"{start+p:04X}: "
>             + " ".join(f"{x:02X}" for x in row)
>         )
>
>     print()
>     print("DESENSAMBLADO:")
>     print()
>
>     text = " ".join(f"0x{x:02X}" for x in blob)
>
>     r = subprocess.run(
>         [
>             "llvm-mc",
>             "--disassemble",
>             "--triple=i386-unknown-none-code16",
>             "--x86-asm-syntax=intel"
>         ],
>         input=text + "\n",
>         text=True,
>         capture_output=True
>     )
>
>     print(r.stdout)
>
>     if r.stderr:
>         print("AVISOS:")
>         print(r.stderr)
>
>
> # ============================================================
> # 1. 76:18F4
> # ============================================================
>
> disasm(
>     76,
>     0x18F4,
>     0x1921
> )
>
> # ============================================================
> # 2. 76:1BEB
> #                                                    o
> # comienzo y final de la rutina. detectar el verdadero
> # ============================================================
>
> disasm(
>     76,
>     0x1B90,
>     0x1C60
> )
>
> # ============================================================
> # 3. 78:066E
> #
> # También mostramos contexto alrededor de la entrada.
> # ============================================================
>
> disasm(
>     78,
>     0x0620,
>     0x0710
> )
>
> # ============================================================
> # 4. Buscar prólogos Delphi cercanos
> # ============================================================
>
> print()
> print("=" * 80)
> print("PROLOGOS 55 8B EC CERCA DE LOS DESTINOS")
> print("=" * 80)
>
> for seg, target in [
>     (76, 0x1BEB),
>     (78, 0x066E)
> ]:
>
>     s = segments[seg]
>
>     inicio = max(0, target - 0x100)
>     fin = min(s["size"] - 2, target + 0x20)
>
>     print()
>     print(f"SEGMENTO {seg}, destino {target:04X}")
>
>     for off in range(inicio, fin):
>
>         b0 = data[s["fileoff"] + off]
>         b1 = data[s["fileoff"] + off + 1]
>         b2 = data[s["fileoff"] + off + 2]
>
>         if (
>             b0 == 0x55 and
>             b1 == 0x8B and
>             b2 == 0xEC
>         ):
>             print(
>                 f"  posible procedimiento: "
>                 f"{seg}:{off:04X}"
>             )
>
> print()
> print("=" * 80)
> print("FIN")
> print("=" * 80)
> PY

================================================================================
SEGMENTO 76:18F4 - 76:1921
================================================================================

BYTES:
18F4: C8 0A 00 00 8D 7E F6 16 57 C4 7E 06 06 57 26 C4
1904: 3D 26 FF 5D 34 83 C4 04 8B 46 FC 89 46 FE 8B 46
1914: FE C9 CA 04 00 55 89 E5 C4 7E 06 06 57

DESENSAMBLADO:

        enter   $10, $0
        leaw    -10(%bp), %di
        pushw   %ss
        pushw   %di
        lesw    6(%bp), %di
        pushw   %es
        pushw   %di
        lesw    %es:(%di), %di
        lcallw  *%es:52(%di)
        addw    $4, %sp
        movw    -4(%bp), %ax
        movw    %ax, -2(%bp)
        movw    -2(%bp), %ax
        leave
        lretw   $4
        pushw   %bp
        movw    %sp, %bp
        lesw    6(%bp), %di
        pushw   %es
        pushw   %di


================================================================================
SEGMENTO 76:1B90 - 76:1C60
================================================================================

BYTES:
1B90: 06 74 2B 26 FF 75 06 26 FF 75 04 BF 99 03 B8 E5
1BA0: 1B 50 57 9A F8 24 0A 1C 08 C0 74 12 C4 7E 06 26
1BB0: C4 7D 04 26 F6 45 18 01 74 04 B0 00 EB 02 B0 01
1BC0: 88 46 FF C4 7E 0A 06 57 C4 7E 06 06 57 9A 5D 4F
1BD0: 5A 20 80 7E FF 00 74 0F C4 7E 0A 06 57 C4 7E 06
1BE0: 06 57 9A 8C 1D 59 1C C9 CA 08 00 C8 10 00 00 8D
1BF0: 7E F0 16 57 C4 7E 06 06 57 26 C4 3D 26 FF 5D 34
1C00: 8D 7E F8 16 57 6A 08 9A A3 18 8E 1C C4 7E 06 26
1C10: FF 75 1E 26 FF 75 20 26 8B 45 22 2B 46 FC 03 46
1C20: 0A 50 26 8B 45 24 2B 46 FE 03 46 0C 50 06 57 26
1C30: C4 3D 26 FF 5D 4C C9 CA 08 00 55 89 E5 C4 7E 06
1C40: 26 8B 45 1A 26 0B 45 1C 74 11 FF 76 08 FF 76 06
1C50: 26 C4 7D 1A 06 57 9A C9 38 71 1C 8B 46 0A 0B 46

DESENSAMBLADO:

        pushw   %es
        je      43
        pushw   %es:6(%di)
        pushw   %es:4(%di)
        movw    $921, %di                       # imm = 0x399
        movw    $7141, %ax                      # imm = 0x1BE5
        pushw   %ax
        pushw   %di
        lcallw  $7178, $9464                    # imm = 0x1C0A
                                        # imm = 0x24F8
        orb     %al, %al
        je      18
        lesw    6(%bp), %di
        lesw    %es:4(%di), %di
        testb   $1, %es:24(%di)
        je      4
        movb    $0, %al
        jmp     2
        movb    $1, %al
        movb    %al, -1(%bp)
        lesw    10(%bp), %di
        pushw   %es
        pushw   %di
        lesw    6(%bp), %di
        pushw   %es
        pushw   %di
        lcallw  $8282, $20317                   # imm = 0x205A
                                        # imm = 0x4F5D
        cmpb    $0, -1(%bp)
        je      15
        lesw    10(%bp), %di
        pushw   %es
        pushw   %di
        lesw    6(%bp), %di
        pushw   %es
        pushw   %di
        lcallw  $7257, $7564                    # imm = 0x1C59
                                        # imm = 0x1D8C
        leave
        lretw   $8
        enter   $16, $0
        leaw    -16(%bp), %di
        pushw   %ss
        pushw   %di
        lesw    6(%bp), %di
        pushw   %es
        pushw   %di
        lesw    %es:(%di), %di
        lcallw  *%es:52(%di)
        leaw    -8(%bp), %di
        pushw   %ss
        pushw   %di
        pushw   $8
        lcallw  $7310, $6307                    # imm = 0x1C8E
                                        # imm = 0x18A3
        lesw    6(%bp), %di
        pushw   %es:30(%di)
        pushw   %es:32(%di)
        movw    %es:34(%di), %ax
        subw    -4(%bp), %ax
        addw    10(%bp), %ax
        pushw   %ax
        movw    %es:36(%di), %ax
        subw    -2(%bp), %ax
        addw    12(%bp), %ax
        pushw   %ax
        pushw   %es
        pushw   %di
        lesw    %es:(%di), %di
        lcallw  *%es:76(%di)
        leave
        lretw   $8
        pushw   %bp
        movw    %sp, %bp
        lesw    6(%bp), %di
        movw    %es:26(%di), %ax
        orw     %es:28(%di), %ax
        je      17
        pushw   8(%bp)
        pushw   6(%bp)
        lesw    %es:26(%di), %di
        pushw   %es
        pushw   %di
        lcallw  $7281, $14537                   # imm = 0x1C71
                                        # imm = 0x38C9
        movw    10(%bp), %ax

AVISOS:
<stdin>:1:1031: warning: invalid instruction encoding
0x06 0x74 0x2B 0x26 0xFF 0x75 0x06 0x26 0xFF 0x75 0x04 0xBF 0x99 0x03 0xB8 0xE5 0x1B 0x50 0x57 0x9A 0xF8 0x24 0x0A 0x1C 0x08 0xC0 0x74 0x12 0xC4 0x7E 0x06 0x26 0xC4 0x7D 0x04 0x26 0xF6 0x45 0x18 0x01 0x74 0x04 0xB0 0x00 0xEB 0x02 0xB0 0x01 0x88 0x46 0xFF 0xC4 0x7E 0x0A 0x06 0x57 0xC4 0x7E 0x06 0x06 0x57 0x9A 0x5D 0x4F 0x5A 0x20 0x80 0x7E 0xFF 0x00 0x74 0x0F 0xC4 0x7E 0x0A 0x06 0x57 0xC4 0x7E 0x06 0x06 0x57 0x9A 0x8C 0x1D 0x59 0x1C 0xC9 0xCA 0x08 0x00 0xC8 0x10 0x00 0x00 0x8D 0x7E 0xF0 0x16 0x57 0xC4 0x7E 0x06 0x06 0x57 0x26 0xC4 0x3D 0x26 0xFF 0x5D 0x34 0x8D 0x7E 0xF8 0x16 0x57 0x6A 0x08 0x9A 0xA3 0x18 0x8E 0x1C 0xC4 0x7E 0x06 0x26 0xFF 0x75 0x1E 0x26 0xFF 0x75 0x20 0x26 0x8B 0x45 0x22 0x2B 0x46 0xFC 0x03 0x46 0x0A 0x50 0x26 0x8B 0x45 0x24 0x2B 0x46 0xFE 0x03 0x46 0x0C 0x50 0x06 0x57 0x26 0xC4 0x3D 0x26 0xFF 0x5D 0x4C 0xC9 0xCA 0x08 0x00 0x55 0x89 0xE5 0xC4 0x7E 0x06 0x26 0x8B 0x45 0x1A 0x26 0x0B 0x45 0x1C 0x74 0x11 0xFF 0x76 0x08 0xFF 0x76 0x06 0x26 0xC4 0x7D 0x1A 0x06 0x57 0x9A 0xC9 0x38 0x71 0x1C 0x8B 0x46 0x0A 0x0B 0x46
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      ^


================================================================================
SEGMENTO 78:0620 - 78:0710
================================================================================

BYTES:
0620: 74 DA 05 27 06 E9 02 37 06 02 00 07 43 6C 61 73
0630: 73 65 73 02 00 A9 05 3B 06 41 4F 27 07 1C 00 FE
0640: FF 00 00 00 00 00 80 00 00 00 80 00 00 04 4E 61
0650: 6D 65 98 25 35 07 0C 00 FF FF 0C 00 FF FF 01 00
0660: 00 00 00 80 00 00 00 00 01 00 03 54 61 67 C8 04
0670: 00 00 8B 46 08 89 46 FC 8B 46 06 89 46 FE 8B 46
0680: FC 8B 56 FE C9 CA 04 00 C8 04 00 00 C4 7E 0E 8B
0690: 46 0C 26 89 05 8B 46 0A 26 89 45 02 8B 46 08 26
06A0: 89 45 04 8B 46 06 26 89 45 06 C9 CA 08 00 C8 04
06B0: 00 00 C4 7E 0E 8B 46 0C 26 89 05 8B 46 0A 26 89
06C0: 45 02 8B 46 0C 03 46 08 26 89 45 04 8B 46 0A 03
06D0: 46 06 26 89 45 06 C9 CA 08 00 55 89 E5 31 C0 31
06E0: D2 C4 7E 04 26 8B 7D E2 09 FF 74 06 26 8B 45 02
06F0: 8C C2 C9 C2 04 00 C8 08 01 00 8D BE F8 FE 16 57
0700: 68 07 F0 C4 7E 04 89 F8 8C C2 89 46 F8 89 56 FA

DESENSAMBLADO:

        je      -38
        addw    $1575, %ax                      # imm = 0x627
        jmp     14082
        pushw   %es
        addb    (%bx,%si), %al
        popw    %es
        incw    %bx
        insb    %dx, %es:(%di)
        popaw
        jae     115
        jae     2
        addb    %ch, 15109(%bx,%di)
        pushw   %es
        incw    %cx
        decw    %di
        daa
        popw    %es
        sbbb    $0, %al
        addb    %al, (%bx,%si)
        addb    %al, (%bx,%si)
        addb    %al, (%bx,%si)
        addb    %al, (%bx,%si)
        addb    $78, %al
        popaw
        insw    %dx, %es:(%di)
        cwtl
        andw    $1845, %ax                      # imm = 0x735
        orb     $0, %al
        orb     $0, %al
        addw    %ax, (%bx,%si)
        addb    %al, (%bx,%si)
        addb    %al, (%bx,%si)
        addb    %al, (%bx,%si)
        addw    %ax, (%bx,%si)
        addw    97(%si), %dx
        addr32          enter   $4, $0
        movw    8(%bp), %ax
        movw    %ax, -4(%bp)
        movw    6(%bp), %ax
        movw    %ax, -2(%bp)
        movw    -4(%bp), %ax
        movw    -2(%bp), %dx
        leave
        lretw   $4
        enter   $4, $0
        lesw    14(%bp), %di
        movw    12(%bp), %ax
        movw    %ax, %es:(%di)
        movw    10(%bp), %ax
        movw    %ax, %es:2(%di)
        movw    8(%bp), %ax
        movw    %ax, %es:4(%di)
        movw    6(%bp), %ax
        movw    %ax, %es:6(%di)
        leave
        lretw   $8
        enter   $4, $0
        lesw    14(%bp), %di
        movw    12(%bp), %ax
        movw    %ax, %es:(%di)
        movw    10(%bp), %ax
        movw    %ax, %es:2(%di)
        movw    12(%bp), %ax
        addw    8(%bp), %ax
        movw    %ax, %es:4(%di)
        movw    10(%bp), %ax
        addw    6(%bp), %ax
        movw    %ax, %es:6(%di)
        leave
        lretw   $8
        pushw   %bp
        movw    %sp, %bp
        xorw    %ax, %ax
        xorw    %dx, %dx
        lesw    4(%bp), %di
        movw    %es:-30(%di), %di
        orw     %di, %di
        je      6
        movw    %es:2(%di), %ax
        movw    %es, %dx
        leave
        retw    $4
        enter   $264, $0                        # imm = 0x108
        leaw    -264(%bp), %di
        pushw   %ss
        pushw   %di
        pushw   $61447                          # imm = 0xF007
        lesw    4(%bp), %di
        movw    %di, %ax
        movw    %es, %dx
        movw    %ax, -8(%bp)
        movw    %dx, -6(%bp)

AVISOS:
<stdin>:1:156: warning: invalid instruction encoding
0x74 0xDA 0x05 0x27 0x06 0xE9 0x02 0x37 0x06 0x02 0x00 0x07 0x43 0x6C 0x61 0x73 0x73 0x65 0x73 0x02 0x00 0xA9 0x05 0x3B 0x06 0x41 0x4F 0x27 0x07 0x1C 0x00 0xFE 0xFF 0x00 0x00 0x00 0x00 0x00 0x80 0x00 0x00 0x00 0x80 0x00 0x00 0x04 0x4E 0x61 0x6D 0x65 0x98 0x25 0x35 0x07 0x0C 0x00 0xFF 0xFF 0x0C 0x00 0xFF 0xFF 0x01 0x00 0x00 0x00 0x00 0x80 0x00 0x00 0x00 0x00 0x01 0x00 0x03 0x54 0x61 0x67 0xC8 0x04 0x00 0x00 0x8B 0x46 0x08 0x89 0x46 0xFC 0x8B 0x46 0x06 0x89 0x46 0xFE 0x8B 0x46 0xFC 0x8B 0x56 0xFE 0xC9 0xCA 0x04 0x00 0xC8 0x04 0x00 0x00 0xC4 0x7E 0x0E 0x8B 0x46 0x0C 0x26 0x89 0x05 0x8B 0x46 0x0A 0x26 0x89 0x45 0x02 0x8B 0x46 0x08 0x26 0x89 0x45 0x04 0x8B 0x46 0x06 0x26 0x89 0x45 0x06 0xC9 0xCA 0x08 0x00 0xC8 0x04 0x00 0x00 0xC4 0x7E 0x0E 0x8B 0x46 0x0C 0x26 0x89 0x05 0x8B 0x46 0x0A 0x26 0x89 0x45 0x02 0x8B 0x46 0x0C 0x03 0x46 0x08 0x26 0x89 0x45 0x04 0x8B 0x46 0x0A 0x03 0x46 0x06 0x26 0x89 0x45 0x06 0xC9 0xCA 0x08 0x00 0x55 0x89 0xE5 0x31 0xC0 0x31 0xD2 0xC4 0x7E 0x04 0x26 0x8B 0x7D 0xE2 0x09 0xFF 0x74 0x06 0x26 0x8B 0x45 0x02 0x8C 0xC2 0xC9 0xC2 0x04 0x00 0xC8 0x08 0x01 0x00 0x8D 0xBE 0xF8 0xFE 0x16 0x57 0x68 0x07 0xF0 0xC4 0x7E 0x04 0x89 0xF8 0x8C 0xC2 0x89 0x46 0xF8 0x89 0x56 0xFA
                                                                                                                                                           ^
<stdin>:1:281: warning: invalid instruction encoding
0x74 0xDA 0x05 0x27 0x06 0xE9 0x02 0x37 0x06 0x02 0x00 0x07 0x43 0x6C 0x61 0x73 0x73 0x65 0x73 0x02 0x00 0xA9 0x05 0x3B 0x06 0x41 0x4F 0x27 0x07 0x1C 0x00 0xFE 0xFF 0x00 0x00 0x00 0x00 0x00 0x80 0x00 0x00 0x00 0x80 0x00 0x00 0x04 0x4E 0x61 0x6D 0x65 0x98 0x25 0x35 0x07 0x0C 0x00 0xFF 0xFF 0x0C 0x00 0xFF 0xFF 0x01 0x00 0x00 0x00 0x00 0x80 0x00 0x00 0x00 0x00 0x01 0x00 0x03 0x54 0x61 0x67 0xC8 0x04 0x00 0x00 0x8B 0x46 0x08 0x89 0x46 0xFC 0x8B 0x46 0x06 0x89 0x46 0xFE 0x8B 0x46 0xFC 0x8B 0x56 0xFE 0xC9 0xCA 0x04 0x00 0xC8 0x04 0x00 0x00 0xC4 0x7E 0x0E 0x8B 0x46 0x0C 0x26 0x89 0x05 0x8B 0x46 0x0A 0x26 0x89 0x45 0x02 0x8B 0x46 0x08 0x26 0x89 0x45 0x04 0x8B 0x46 0x06 0x26 0x89 0x45 0x06 0xC9 0xCA 0x08 0x00 0xC8 0x04 0x00 0x00 0xC4 0x7E 0x0E 0x8B 0x46 0x0C 0x26 0x89 0x05 0x8B 0x46 0x0A 0x26 0x89 0x45 0x02 0x8B 0x46 0x0C 0x03 0x46 0x08 0x26 0x89 0x45 0x04 0x8B 0x46 0x0A 0x03 0x46 0x06 0x26 0x89 0x45 0x06 0xC9 0xCA 0x08 0x00 0x55 0x89 0xE5 0x31 0xC0 0x31 0xD2 0xC4 0x7E 0x04 0x26 0x8B 0x7D 0xE2 0x09 0xFF 0x74 0x06 0x26 0x8B 0x45 0x02 0x8C 0xC2 0xC9 0xC2 0x04 0x00 0xC8 0x08 0x01 0x00 0x8D 0xBE 0xF8 0xFE 0x16 0x57 0x68 0x07 0xF0 0xC4 0x7E 0x04 0x89 0xF8 0x8C 0xC2 0x89 0x46 0xF8 0x89 0x56 0xFA
                                                                                                                                                                                                                                                                                        ^
<stdin>:1:301: warning: invalid instruction encoding
0x74 0xDA 0x05 0x27 0x06 0xE9 0x02 0x37 0x06 0x02 0x00 0x07 0x43 0x6C 0x61 0x73 0x73 0x65 0x73 0x02 0x00 0xA9 0x05 0x3B 0x06 0x41 0x4F 0x27 0x07 0x1C 0x00 0xFE 0xFF 0x00 0x00 0x00 0x00 0x00 0x80 0x00 0x00 0x00 0x80 0x00 0x00 0x04 0x4E 0x61 0x6D 0x65 0x98 0x25 0x35 0x07 0x0C 0x00 0xFF 0xFF 0x0C 0x00 0xFF 0xFF 0x01 0x00 0x00 0x00 0x00 0x80 0x00 0x00 0x00 0x00 0x01 0x00 0x03 0x54 0x61 0x67 0xC8 0x04 0x00 0x00 0x8B 0x46 0x08 0x89 0x46 0xFC 0x8B 0x46 0x06 0x89 0x46 0xFE 0x8B 0x46 0xFC 0x8B 0x56 0xFE 0xC9 0xCA 0x04 0x00 0xC8 0x04 0x00 0x00 0xC4 0x7E 0x0E 0x8B 0x46 0x0C 0x26 0x89 0x05 0x8B 0x46 0x0A 0x26 0x89 0x45 0x02 0x8B 0x46 0x08 0x26 0x89 0x45 0x04 0x8B 0x46 0x06 0x26 0x89 0x45 0x06 0xC9 0xCA 0x08 0x00 0xC8 0x04 0x00 0x00 0xC4 0x7E 0x0E 0x8B 0x46 0x0C 0x26 0x89 0x05 0x8B 0x46 0x0A 0x26 0x89 0x45 0x02 0x8B 0x46 0x0C 0x03 0x46 0x08 0x26 0x89 0x45 0x04 0x8B 0x46 0x0A 0x03 0x46 0x06 0x26 0x89 0x45 0x06 0xC9 0xCA 0x08 0x00 0x55 0x89 0xE5 0x31 0xC0 0x31 0xD2 0xC4 0x7E 0x04 0x26 0x8B 0x7D 0xE2 0x09 0xFF 0x74 0x06 0x26 0x8B 0x45 0x02 0x8C 0xC2 0xC9 0xC2 0x04 0x00 0xC8 0x08 0x01 0x00 0x8D 0xBE 0xF8 0xFE 0x16 0x57 0x68 0x07 0xF0 0xC4 0x7E 0x04 0x89 0xF8 0x8C 0xC2 0x89 0x46 0xF8 0x89 0x56 0xFA
                                                                                                                                                                                                                                                                                                            ^


================================================================================
PROLOGOS 55 8B EC CERCA DE LOS DESTINOS
================================================================================

SEGMENTO 76, destino 1BEB

SEGMENTO 78, destino 066E

================================================================================
FIN
================================================================================
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Sí. Esta salida aclara bastante la estructura.


### Lo que acabamos de confirmar


1. **`76:18F4` sí es entrada válida de una rutina.**

Comienza con:


```
C8 0A 00 00
```


que es:


```
asmenter 10,0
```


En Delphi 16-bit no todas las rutinas empiezan con `55 89 E5`; también puede usar `ENTER`.


1. **`76:1BEB` también es una entrada válida.**


Ahí aparece:


```
1BEB: C8 10 00 00
```


→ `enter 16,0`.


Por tanto, la búsqueda exclusiva de `55 89 E5` estaba perdiendo rutinas Delphi perfectamente válidas.


1. **`78:066E` también es una entrada válida.**


```
066E: C8 04 00 00
```


→ `enter 4,0`.


Además, vemos algo muy interesante: antes de `066E` hay claramente **datos/metadatos Delphi** (`Classes`, `Name`, `Tag`, etc.), y a partir de `066E` empieza código. Eso confirma que estamos atravesando correctamente la mezcla de RTTI/datos/código propia del ejecutable Delphi.


1. Por tanto, las tres llamadas que encontramos en el segmento 76 son coherentes:


```
76:18D9 → offset 18F4
76:18DF → offset 066E
76:18EB → offset 1BEB
```


Lo que **todavía no sabemos** es a qué segmentos reales apuntan sus operandos segmentados, porque los valores que vemos (`18EE`, `192D`, `1924`) siguen siendo cadenas de relocalización.


### El siguiente paso es el importante


Ahora debemos resolver las relocalizaciones **por cada segmento**, no reutilizar la tabla de relocalizaciones del segmento 3 para analizar el segmento 76.


Ejecuta este único bloque. Va a resolver simultáneamente las llamadas de `CalcularClick` y las tres llamadas del segmento 76:


```
Bashcd /storage/emulated/0/Download/armon

python3 - <<'PY'
from pathlib import Path
import struct

data = Path("ARMON.EXE").read_bytes()

NE = 0x100

SEG_COUNT = struct.unpack_from("<H", data, NE + 0x1C)[0]
SEG_TABLE = NE + struct.unpack_from("<H", data, NE + 0x22)[0]
SHIFT = struct.unpack_from("<H", data, NE + 0x32)[0]

segments = {}

for n in range(1, SEG_COUNT + 1):

    p = SEG_TABLE + (n - 1) * 8

    sector = struct.unpack_from("<H", data, p)[0]
    length = struct.unpack_from("<H", data, p + 2)[0]
    flags = struct.unpack_from("<H", data, p + 4)[0]

    size = 0x10000 if length == 0 else length
    fileoff = sector << SHIFT

    segments[n] = {
        "fileoff": fileoff,
        "size": size,
        "flags": flags
    }


# ============================================================
# Construye las cadenas de relocación DE UN SEGMENTO
# ============================================================

def relocation_chains(segnum):

    s = segments[segnum]

    if not (s["flags"] & 0x0100):
        return []

    rel_base = s["fileoff"] + s["size"]

    count = struct.unpack_from("<H", data, rel_base)[0]

    records = []

    for i in range(count):

        p = rel_base + 2 + i * 8

        source_type = data[p]
        flags = data[p + 1]

        source = struct.unpack_from("<H", data, p + 2)[0]

        target = data[p + 4:p + 8]

        target_word1 = struct.unpack_from("<H", data, p + 4)[0]
        target_word2 = struct.unpack_from("<H", data, p + 6)[0]

        records.append({
            "index": i,
            "type": source_type,
            "flags": flags,
            "source": source,
            "target": target,
            "target1": target_word1,
            "target2": target_word2
        })

    return records


# ============================================================
# Obtiene todas las posiciones pertenecientes a cada cadena
# ============================================================

def build_source_map(segnum):

    records = relocation_chains(segnum)

    source_map = {}

    for r in records:

        first = r["source"]

        # Cada cadena comienza en source y los WORD siguientes
        # contienen el siguiente desplazamiento de la cadena.

        pos = first
        visited = set()

        while pos != 0xFFFF and pos not in visited:

            visited.add(pos)

            source_map[pos] = r

            if pos + 2 > segments[segnum]["size"]:
                break

            filepos = (
                segments[segnum]["fileoff"] +
                pos
            )

            nxt = struct.unpack_from("<H", data, filepos)[0]

            pos = nxt

    return source_map


# ============================================================
# Resolver un WORD que contiene el selector de segmento
# ============================================================

def resolve_segment(segnum, source):

    source_map = build_source_map(segnum)

    r = source_map.get(source)

    if r is None:
        return None

    # Para relocaciones internas tipo 02:
    #
    # target1 = número de segmento
    #
    # El segundo WORD pertenece al target union y no es
    # necesario para determinar el segmento.

    if r["type"] == 2 and (r["flags"] & 3) == 0:
        return r["target1"], r

    return None


# ============================================================
# Mostrar resolución
# ============================================================

def show_call(segnum, call_offset, target_offset, seg_source):

    result = resolve_segment(segnum, seg_source)

    print()
    print("-" * 80)

    print(
        f"CALL  {segnum}:{call_offset:04X}"
        f"  -> offset {target_offset:04X}"
    )

    print(
        f"WORD segmento localizado en "
        f"{segnum}:{seg_source:04X}"
    )

    if result is None:

        print("NO SE ENCONTRO RELOCALIZACION INTERNA.")

        # Mostrar qué relocaciones existen cerca
        records = relocation_chains(segnum)

        cercanas = []

        for r in records:

            d = abs(r["source"] - seg_source)

            if d <= 0x30:
                cercanas.append(r)

        if cercanas:

            print()
            print("RELOCALIZACIONES CERCANAS:")

            for r in cercanas:

                print(
                    f"  rec {r['index']:3d} "
                    f"type={r['type']:02X} "
                    f"flags={r['flags']:02X} "
                    f"source={r['source']:04X} "
                    f"target="
                    f"{r['target'].hex(' ')}"
                )

        return

    target_seg, r = result

    print()
    print("RESUELTO:")

    print(
        f"  segmento destino = {target_seg}"
    )

    print(
        f"  destino completo = "
        f"{target_seg}:{target_offset:04X}"
    )

    print(
        f"  relocación       = "
        f"registro {r['index']}"
    )

    print(
        f"  tipo             = {r['type']:02X}"
    )

    print(
        f"  flags            = {r['flags']:02X}"
    )

    print(
        f"  cadena inicial   = "
        f"{r['source']:04X}"
    )


# ============================================================
# 1. CalcularClick — segmento 3
# ============================================================

print()
print("=" * 80)
print("CALCULARCLICK — SEGMENTO 3")
print("=" * 80)

show_call(
    3,
    0x1DC9,
    0x0444,
    0x1DCC
)

show_call(
    3,
    0x1DD6,
    0x18F4,
    0x1DD9
)

show_call(
    3,
    0x1DE8,
    0x18A9,
    0x1DEB
)


# ============================================================
# 2. Rutinas del segmento 76
# ============================================================

print()
print("=" * 80)
print("LLAMADAS — SEGMENTO 76")
print("=" * 80)

show_call(
    76,
    0x18D9,
    0x18F4,
    0x18DC
)

show_call(
    76,
    0x18DF,
    0x066E,
    0x18E2
)

show_call(
    76,
    0x18EB,
    0x1BEB,
    0x18EE
)


# ============================================================
# 3. Información básica de los segmentos destino
# ============================================================

print()
print("=" * 80)
print("SEGMENTOS DESTINO")
print("=" * 80)

# Estos son los offsets que estamos investigando.
# No asumimos todavía qué segmento corresponde.

for seg in range(1, SEG_COUNT + 1):

    s = segments[seg]

    if seg in (76, 78, 80):

        print(
            f"segmento {seg:2d}  "
            f"file={s['fileoff']:06X}  "
            f"size={s['size']:04X}  "
            f"flags={s['flags']:04X}"
        )


print()
print("=" * 80)
print("FIN")
print("=" * 80)

PY
```


La clave ahora es que el resultado nos dará algo del tipo:


```
CALL  3:1DC9 -> offset 0444
segmento destino = XX
destino completo = XX:0444
```


y, separadamente:


```
CALL 76:18DF -> offset 066E
segmento destino = YY
destino completo = YY:066E
```


Con eso podremos establecer **qué código común está utilizando realmente `CalcularClick`** y, más importante, empezar a reconstruir la relación entre las rutinas Delphi originales y sus unidades/módulos compilados.


Además, ya no debemos preocuparnos porque `80:0444` no tenga prólogo `55 89 E5`: **un destino de `CALL FAR` puede ser una entrada interna a una rutina/thunk y no necesariamente el comienzo del procedimiento Delphi**.

## Usuario

~ $ cd /storage/emulated/0/Download/armon
.../Download/armon $
.../Download/armon $ python3 - <<'PY'
> from pathlib import Path
> import struct
>
> data = Path("ARMON.EXE").read_bytes()
>
> NE = 0x100
>
> SEG_COUNT = struct.unpack_from("<H", data, NE + 0x1C)[0]
> SEG_TABLE = NE + struct.unpack_from("<H", data, NE + 0x22)[0]
> SHIFT = struct.unpack_from("<H", data, NE + 0x32)[0]
>
> segments = {}
>
> for n in range(1, SEG_COUNT + 1):
>
>     p = SEG_TABLE + (n - 1) * 8
>
>     sector = struct.unpack_from("<H", data, p)[0]
>     length = struct.unpack_from("<H", data, p + 2)[0]
>     flags = struct.unpack_from("<H", data, p + 4)[0]
>
>     size = 0x10000 if length == 0 else length
>     fileoff = sector << SHIFT
>
>     segments[n] = {
>         "fileoff": fileoff,
>         "size": size,
>         "flags": flags
>     }
>
>
cation_chains(segnum)

        cercanas = []

        for r in records:

            d = abs(r["source"] - seg_source)

            if d <= 0x30:
                cercanas.append(r)

        if cercanas:

            print()
            print("RELOCALIZACIONES CERCANAS:")

            for r in cercanas:

                print(
                    f"  rec {r['index']:3d} "
                    f"type={r['type']:02X} "
                    f"flags={r['flags']:02X} "
                    f"source={r['source']:04X} "
                    f"target="
                    f"{r['target'].hex(' ')}"
                )

        return

    target_seg, r = result

    print()
    print("RESUELTO:")

    print(
        f"  segmento destino = {target_seg}"
    )

    print(
        f"  destino completo = "
        f"{target_seg}:{target_offset:04X}"
    )

    print(
        f"  relocación > # ============================================================
> # Construye las cadenas de relocación DE UN SEGMENTO
> # ============================================================
flags']:02X}"
    >
> def relocation_chains(segnum):
>
 >     s = segments[segnum]
>
>     if not (s["flags"] & 0x0100):
>         return []
>
=>     rel_base = s["fileoff"] + s["size"]
>
>     count = struct.unpack_from("<H", data, rel_base)[0]
>
>     records = []
>
>     for i in range(count):
>
>         p = rel_base + 2 + i * 8
>
>         source_type = data[p]
>         flags = data[p + 1]
>
>         source = struct.unpack_from("<H", data, p + 2)[0]
>
>         target = data[p + 4:p + 8]
>
>         target_word1 = struct.unpack_from("<H", data, p + 4)[0]
>         target_word2 = struct.unpack_from("<H", data, p + 6)[0]
>
>         records.append({
>             "index": i,
>             "type": source_type,
>             "flags": flags,
>             "source": source,
>             "target": target,
>             "target1": target_word1,
>             "target2": target_word2
>         })
>
 >     return records
>
>
> # ============================================================
> # Obtiene todas las posiciones pertenecientes a cada cadena
> # ============================================================
>
=> def build_source_map(segnum):
>
=>     records = relocation_chains(segnum)
>
G>     source_map = {}
>
n>     for r in records:
>
o>         first = r["source"]
>
v>         # Cada cadena comienza en source y los WORD siguientes
>         # contienen el siguiente desplazamiento de la cadena.
1, SEG_COUNT + 1):

    s = segments[seg]

   >
>         pos = first
>         visited = set()
>
 >         while pos != 0xFFFF and pos not in visited:
>
'>             visited.add(pos)
>
      >             source_map[pos] = r
>
>             if pos + 2 > segments[segnum]["size"]:
>                 break
>
>             filepos = (
>                 segments[segnum]["fileoff"] +
>                 pos
>             )
>
>             nxt = struct.unpack_from("<H", data, filepos)[0]
>
>             pos = nxt
>
>     return source_map
>
>
> # ============================================================
> # Resolver un WORD que contiene el selector de segmento
> # ============================================================
>
> def resolve_segment(segnum, source):
>
>     source_map = build_source_map(segnum)
>
>     r = source_map.get(source)
>
>     if r is None:
>         return None
>
>     # Para relocaciones internas tipo 02:
>     #
>     # target1 = número de segmento
>     #
>     # El segundo WORD pertenece al target union y no es
>     # necesario para determinar el segmento.
>
>     if r["type"] == 2 and (r["flags"] & 3) == 0:
>         return r["target1"], r
>
>     return None
>
>
> # ============================================================
> # Mostrar resolución
> # ============================================================
>
> def show_call(segnum, call_offset, target_offset, seg_source):
>
>     result = resolve_segment(segnum, seg_source)
>
>     print()
>     print("-" * 80)
>
>     print(
>         f"CALL  {segnum}:{call_offset:04X}"
>         f"  -> offset {target_offset:04X}"
>     )
>
>     print(
>         f"WORD segmento localizado en "
>         f"{segnum}:{seg_source:04X}"
>     )
>
>     if result is None:
>
>         print("NO SE ENCONTRO RELOCALIZACION INTERNA.")
>
>         # Mostrar qué relocaciones existen cerca
>         records = relocation_chains(segnum)
>
>         cercanas = []
>
>         for r in records:
>
>             d = abs(r["source"] - seg_source)
>
>             if d <= 0x30:
>                 cercanas.append(r)
>
>         if cercanas:
>
>             print()
>             print("RELOCALIZACIONES CERCANAS:")
>
>             for r in cercanas:
>
>                 print(
>                     f"  rec {r['index']:3d} "
>                     f"type={r['type']:02X} "
>                     f"flags={r['flags']:02X} "
>                     f"source={r['source']:04X} "
>                     f"target="
>                     f"{r['target'].hex(' ')}"
>                 )
>
>         return
>
>     target_seg, r = result
>
>     print()
>     print("RESUELTO:")
>
>     print(
>         f"  segmento destino = {target_seg}"
>     )
>
>     print(
>         f"  destino completo = "
>         f"{target_seg}:{target_offset:04X}"
>     )
>
>     print(
>         f"  relocación       = "
>         f"registro {r['index']}"
>     )
>
>     print(
>         f"  tipo             = {r['type']:02X}"
>     )
>
>     print(
>         f"  flags            = {r['flags']:02X}"
>     )
>
>     print(
>         f"  cadena inicial   = "
>         f"{r['source']:04X}"
>     )
>
>
> # ============================================================
> # 1. CalcularClick — segmento 3
> # ============================================================
>
> print()
> print("=" * 80)
> print("CALCULARCLICK — SEGMENTO 3")
> print("=" * 80)
>
> show_call(
>     3,
>     0x1DC9,
>     0x0444,
>     0x1DCC
> )
>
> show_call(
>     3,
>     0x1DD6,
>     0x18F4,
>     0x1DD9
> )
>
> show_call(
>     3,
>     0x1DE8,
>     0x18A9,
>     0x1DEB
> )
>
>
> # ============================================================
> # 2. Rutinas del segmento 76
> # ============================================================
>
> print()
> print("=" * 80)
> print("LLAMADAS — SEGMENTO 76")
> print("=" * 80)
>
> show_call(
>     76,
>     0x18D9,
>     0x18F4,
>     0x18DC
> )
>
> show_call(
>     76,
>     0x18DF,
>     0x066E,
>     0x18E2
> )
>
> show_call(
>     76,
>     0x18EB,
>     0x1BEB,
>     0x18EE
> )
>
>
> # ============================================================
> # 3. Información ásica de los segmentos destino
> # ============================================================
>
> print()
> print("=" * 80)
> print("SEGMENTOS DESTINO")
> print("=" * 80)
>
> # Estos son los offsets que estamos investigando.
> # No asumimos todavía qé segmento corresponde.
>
> for seg in range(1, SEG_COUNT + 1):
>
>     s = segments[seg]
>
>     if seg in (76, 78, 80):
>
>         print(
>             f"segmento {seg:2d}  "
>             f"file={s['fileoff']:06X}  "
>             f"size={s['size']:04X}  "
>             f"flags={s['flags']:04X}"
>         )
>
>
> print()
> print("=" * 80)
> print("FIN")
> print("=" * 80)
>
> PY

================================================================================
CALCULARCLICK — SEGMENTO 3
================================================================================

--------------------------------------------------------------------------------
CALL  3:1DC9  -> offset 0444
WORD segmento localizado en 3:1DCC

RESUELTO:
  segmento destino = 80
  destino completo = 80:0444
  relocación       = registro 3
  tipo             = 02
  flags            = 00
  cadena inicial   = 001C

--------------------------------------------------------------------------------
CALL  3:1DD6  -> offset 18F4
WORD segmento localizado en 3:1DD9

RESUELTO:
  segmento destino = 76
  destino completo = 76:18F4
  relocación       = registro 1
  tipo             = 02
  flags            = 00
  cadena inicial   = 0078

--------------------------------------------------------------------------------
CALL  3:1DE8  -> offset 18A9
WORD segmento localizado en 3:1DEB

RESUELTO:
  segmento destino = 76
  destino completo = 76:18A9
  relocación       = registro 1
  tipo             = 02
  flags            = 00
  cadena inicial   = 0078

================================================================================
LLAMADAS — SEGMENTO 76
================================================================================

--------------------------------------------------------------------------------
CALL  76:18D9  -> offset 18F4
WORD segmento localizado en 76:18DC

RESUELTO:
  segmento destino = 76
  destino completo = 76:18F4
  relocación       = registro 2
  tipo             = 02
  flags            = 00
  cadena inicial   = 00AC

--------------------------------------------------------------------------------
CALL  76:18DF  -> offset 066E
WORD segmento localizado en 76:18E2

RESUELTO:
  segmento destino = 78
  destino completo = 78:066E
  relocación       = registro 4
  tipo             = 02
  flags            = 00
  cadena inicial   = 0098

--------------------------------------------------------------------------------
CALL  76:18EB  -> offset 1BEB
WORD segmento localizado en 76:18EE

RESUELTO:
  segmento destino = 76
  destino completo = 76:1BEB
  relocación       = registro 2
  tipo             = 02
  flags            = 00
  cadena inicial   = 00AC

================================================================================
SEGMENTOS DESTINO
================================================================================
segmento 76  file=01BAC0  size=6EB3  flags=1D50
segmento 78  file=02A9C0  size=51C4  flags=1D50
segmento 80  file=032A00  size=261A  flags=0D50

================================================================================
FIN
================================================================================
.../Download/armon $ bash ~/rotar_sesion.sh

## Asistente

Perfecto. **Ahora sí tenemos una resolución fiable de las llamadas FAR**, porque cada llamada fue resuelta utilizando la tabla de relocaciones de su propio segmento.


### Resultado confirmado


La cadena principal de `CalcularClick` queda:


```
SEGMENTO 3 — CalcularClick @ 1DC3

3:1DC9  CALL FAR  80:0444
3:1DD6  CALL FAR  76:18F4
3:1DE8  CALL FAR  76:18A9
```


Y el segmento 76 contiene:


```
76:18D9  CALL FAR  76:18F4
76:18DF  CALL FAR  78:066E
76:18EB  CALL FAR  76:1BEB
```


Esto es muy valioso porque ya podemos construir un pequeño **grafo real de llamadas**:


```
                 ┌──────────────┐
                 │ CalcularClick│
                 │    3:1DC3   │
                 └──────┬───────┘
                        │
             ┌──────────┼──────────┐
             │          │          │
             ▼          ▼          ▼
          80:0444    76:18F4   76:18A9
                        │
                        │
                 ┌──────┴──────┐
                 │             │
                 ▼             ▼
             76:18F4       76:1BEB
                 │
                 ▼
              78:066E
```


Hay además una observación importante:


**76:18A9 y 76:18F4 pertenecen al mismo módulo/segmento y son reutilizadas desde `CalcularClick`.** Eso empieza a parecer mucho más una biblioteca/rutinas comunes que código específico del evento `CalcularClick`.


### Y podemos identificar qué hacen algunas


Por el código que ya desensamblamos:


**78:066E**:


```
asmpush bp
mov  bp,sp
xor  ax,ax
xor  dx,dx
les  di,[bp+4]
mov  di,es:[di-1Eh]
or   di,di
je   ...
mov  ax,es:[di+2]
mov  dx,es
...
ret 4
```


Esto parece una rutina que recibe un puntero/objeto y **extrae un valor de 32 bits (DX:AX)** desde una estructura. Es decir, tiene aspecto de getter/acceso a datos, no de lógica astrológica directamente.


**76:1BEB**, en cambio, trabaja con estructuras pasadas por puntero y realiza varias llamadas y cálculos sobre sus campos. Esa tiene bastante más posibilidades de ser una rutina de nivel superior.


---


## El siguiente objetivo ya no debería ser seguir llamadas arbitrariamente


Ahora podemos hacer algo mucho más potente:


**reconstruir automáticamente las rutinas Delphi del segmento 3 y sus dependencias.**


Partiremos de:


```
TFORM1
   ↓
CalcularClick
   ↓
rutinas llamadas
   ↓
rutinas llamadas por esas rutinas
   ↓
...
```


y etiquetaremos cada nodo como:


- procedimiento Delphi
- función
- getter/setter
- rutina RTL/VCL
- rutina propia del programa
- llamada importada
- dato/RTTI


Además, podemos aprovechar los nombres que ya recuperamos:


```
CalcularClick
FormResize
UseHelioClick
UseSiderClick
FormCreate
AhoraClick
DHGoHarmClick
ArmonicoChange
...
```


para comenzar a reconstruir **el código Pascal original a nivel semántico**, no simplemente ensamblador.


El próximo paso que haría es localizar **todas las llamadas FAR de `CalcularClick`, resolver automáticamente sus segmentos y offsets y desensamblar cada destino con suficiente contexto para determinar sus límites reales**. Esto nos permitirá empezar a traducir `CalcularClick` a pseudocódigo Delphi.

## Usuario

Continua

## Asistente

Sí. Con la resolución anterior ya podemos pasar al **siguiente nivel: reconstruir el árbol de llamadas de `CalcularClick` automáticamente**.


La idea es no volver a adivinar segmentos. El programa hará esto:


1. Toma `3:1DC3` (`CalcularClick`).
2. Busca sus `CALL FAR`.
3. Resuelve cada selector mediante las **relocaciones reales**.
4. Entra en cada rutina encontrada.
5. Busca las siguientes llamadas FAR.
6. Repite varias generaciones.
7. Marca las rutinas ya visitadas para evitar ciclos.
8. Muestra el comienzo de cada rutina y sus llamadas.


Esto nos permitirá empezar a separar **código propio de ARMÓN** de las rutinas auxiliares de Delphi/VCL.


Ejecuta **todo este bloque completo**:


```
Bashcd /storage/emulated/0/Download/armon

python3 - <<'PY'
from pathlib import Path
import struct

data = Path("ARMON.EXE").read_bytes()

NE = 0x100

SEG_COUNT = struct.unpack_from("<H", data, NE + 0x1C)[0]
SEG_TABLE = NE + struct.unpack_from("<H", data, NE + 0x22)[0]
SHIFT = struct.unpack_from("<H", data, NE + 0x32)[0]

# ============================================================
# SEGMENTOS
# ============================================================

segments = {}

for n in range(1, SEG_COUNT + 1):
    p = SEG_TABLE + (n - 1) * 8

    sector = struct.unpack_from("<H", data, p)[0]
    length = struct.unpack_from("<H", data, p + 2)[0]
    flags = struct.unpack_from("<H", data, p + 4)[0]

    size = 0x10000 if length == 0 else length
    fileoff = sector << SHIFT

    segments[n] = {
        "fileoff": fileoff,
        "size": size,
        "flags": flags
    }


# ============================================================
# RELOCALIZACIONES
# ============================================================

def relocation_records(segnum):

    s = segments[segnum]

    if not (s["flags"] & 0x0100):
        return []

    rel_base = s["fileoff"] + s["size"]

    count = struct.unpack_from("<H", data, rel_base)[0]

    records = []

    for i in range(count):

        p = rel_base + 2 + i * 8

        source_type = data[p]
        flags = data[p + 1]
        source = struct.unpack_from("<H", data, p + 2)[0]

        target = data[p + 4:p + 8]

        target1 = struct.unpack_from("<H", data, p + 4)[0]
        target2 = struct.unpack_from("<H", data, p + 6)[0]

        records.append({
            "index": i,
            "type": source_type,
            "flags": flags,
            "source": source,
            "target": target,
            "target1": target1,
            "target2": target2
        })

    return records


# ============================================================
# MAPA DE CADENAS DE RELOCALIZACION
# ============================================================

source_maps = {}


def build_source_map(segnum):

    if segnum in source_maps:
        return source_maps[segnum]

    records = relocation_records(segnum)

    source_map = {}

    for r in records:

        pos = r["source"]
        visited = set()

        while pos != 0xFFFF and pos not in visited:

            visited.add(pos)

            source_map[pos] = r

            if pos + 2 > segments[segnum]["size"]:
                break

            filepos = segments[segnum]["fileoff"] + pos

            nxt = struct.unpack_from("<H", data, filepos)[0]

            pos = nxt

    source_maps[segnum] = source_map

    return source_map


# ============================================================
# RESOLVER SELECTOR DE SEGMENTO
# ============================================================

def resolve_segment(segnum, source):

    m = build_source_map(segnum)

    r = m.get(source)

    if r is None:
        return None

    # Relocacion interna:
    # type 02 + target kind 00
    if r["type"] == 2 and (r["flags"] & 3) == 0:
        return r["target1"]

    return None


# ============================================================
# LEER BYTES
# ============================================================

def get_bytes(segnum, offset, n=32):

    s = segments[segnum]

    start = s["fileoff"] + offset

    end = min(
        start + n,
        s["fileoff"] + s["size"]
    )

    return data[start:end]


# ============================================================
# DETECTAR CALL FAR DIRECTO
#
# 9A oo oo ss ss
#
# El WORD del segmento está sujeto a relocación.
# ============================================================

def direct_far_calls(segnum, start, end):

    s = segments[segnum]

    end = min(end, s["size"])

    result = []

    p = start

    while p + 5 <= end:

        filepos = s["fileoff"] + p

        opcode = data[filepos]

        if opcode == 0x9A:

            offset = struct.unpack_from(
                "<H",
                data,
                filepos + 1
            )[0]

            seg_source = p + 3

            target_seg = resolve_segment(
                segnum,
                seg_source
            )

            result.append({
                "call": p,
                "offset": offset,
                "seg_source": seg_source,
                "segment": target_seg
            })

            p += 5

        else:

            p += 1

    return result


# ============================================================
# DETECTAR FINAL DE PROCEDIMIENTO
#
# C3 = RET
# CB = RETF
# C2 xx xx = RET n
# CA xx xx = RETF n
#
# Para una primera aproximación usamos el primer RETF.
# ============================================================

def approximate_end(segnum, start, maxlen=0x1000):

    s = segments[segnum]

    end = min(
        start + maxlen,
        s["size"]
    )

    p = start

    while p < end:

        op = data[s["fileoff"] + p]

        if op in (0xCB, 0xCA):

            if op == 0xCA and p + 3 <= end:
                return p + 3

            return p + 1

        p += 1

    return min(start + maxlen, s["size"])


# ============================================================
# RECONOCER PROLOGO
# ============================================================

def prologue(segnum, offset):

    b = get_bytes(segnum, offset, 8)

    if b.startswith(b"\x55\x89\xE5"):
        return "PUSH BP / MOV BP,SP"

    if len(b) >= 4 and b[0] == 0xC8 and b[3] == 0x00:
        n = b[1] | (b[2] << 8)
        return f"ENTER {n}"

    if b.startswith(b"\x55\x8B\xEC"):
        return "PUSH BP / MOV BP,SP"

    return "sin prologo Delphi evidente"


# ============================================================
# HEURISTICA PARA MOSTRAR BYTES
# ============================================================

def hexbytes(segnum, offset, n=20):

    return " ".join(
        f"{x:02X}"
        for x in get_bytes(segnum, offset, n)
    )


# ============================================================
# GRAFO DE LLAMADAS
# ============================================================

visited = set()
queue = [(3, 0x1DC3, "CalcularClick")]

MAX_NODES = 60
MAX_DEPTH = 5

print()
print("=" * 80)
print("GRAFO DE LLAMADAS FAR — ARMÓN / DELPHI")
print("=" * 80)

print()
print("Punto inicial:")
print("  3:1DC3  CalcularClick")

nodes = 0

while queue and nodes < MAX_NODES:

    segnum, offset, name = queue.pop(0)

    key = (segnum, offset)

    if key in visited:
        continue

    visited.add(key)
    nodes += 1

    print()
    print("-" * 80)

    print(
        f"RUTINA {nodes}: "
        f"{segnum}:{offset:04X}"
    )

    print(
        f"  nombre/contexto : {name}"
    )

    print(
        f"  prologo         : "
        f"{prologue(segnum, offset)}"
    )

    print(
        f"  bytes           : "
        f"{hexbytes(segnum, offset)}"
    )

    end = approximate_end(segnum, offset)

    print(
        f"  fin aproximado  : "
        f"{segnum}:{end:04X}"
    )

    calls = direct_far_calls(
        segnum,
        offset,
        end
    )

    if not calls:

        print("  CALL FAR        : ninguno")

        continue

    print("  CALL FAR:")

    for c in calls:

        if c["segment"] is None:

            print(
                f"    {segnum}:{c['call']:04X} "
                f"-> ???:{c['offset']:04X} "
                f"(relocacion no interna)"
            )

            continue

        dst = c["segment"]

        print(
            f"    {segnum}:{c['call']:04X} "
            f"-> {dst}:{c['offset']:04X}"
        )

        # Evitar que el grafo crezca indefinidamente.
        if len(visited) < MAX_NODES:

            queue.append(
                (
                    dst,
                    c["offset"],
                    f"desde {segnum}:{offset:04X}"
                )
            )


# ============================================================
# RESUMEN
# ============================================================

print()
print("=" * 80)
print("NODOS DESCUBIERTOS")
print("=" * 80)

for i, (seg, off) in enumerate(
    sorted(visited),
    1
):

    print(
        f"{i:3d}. "
        f"{seg}:{off:04X}  "
        f"{prologue(seg, off)}"
    )

print()
print("=" * 80)
print(f"TOTAL: {len(visited)} rutinas/entradas")
print("=" * 80)

PY
```


### Qué espero que nos revele


Lo especialmente importante será comprobar qué ocurre después de estas tres entradas ya confirmadas:


```
80:0444
76:18F4
76:18A9
```


y después:


```
76:18F4
 ├── ...
 ├── ...
 └── ...

76:18A9
 ├── ...
 └── ...

80:0444
 ├── ...
 └── ...
```


Si el resultado es razonablemente limpio, el siguiente paso será **mucho más interesante que seguir desensamblando a mano**: podremos tomar cada rutina y reconstruir su lógica en pseudocódigo tipo Delphi:


```
procedure CalcularClick;
begin
    ...
    valor1 := ...
    valor2 := ...
    ...
end;
```


y, donde sea posible, relacionar las variables con los componentes reales de `TFORM1` que ya recuperamos del `TFORM1` RCDATA.


**Pégame la salida completa de este bloque.** Con ella seguimos desde el grafo real, sin volver a hacer suposiciones sobre los segmentos.
