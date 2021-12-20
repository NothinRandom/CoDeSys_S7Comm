# CoDeSys_S7Comm
**What?**

CoDeSys_S7Comm is a CoDeSys 3.5.16.0 library that allows your CoDeSys controller (IPC) to communicate with Siemens S7 programmable logic controller (PLC).

**Why?**

Previously, a [CoDeSys EtherNet/IP library (CoDeSys_EIP)](https://github.com/NothinRandom/CoDeSys_EIP) was developed in order for CoDeSys IPC to be able to communicate with various EtherNet/IP capable devices via explicit messaging.  This means, we still have at least two other major vendors out there: Siemens and Mitsubishi.  STEP 7 Communication can be used to read/write to the Siemens PLCs (see `Examples/Siemens`).

For the control engineers out there, you might already know that writing PLC code is not as flexible as writing higher level languages such as Python/Java/etc, where you can create variables with virtually any data type on the fly; thus, this library was heavily modified to accomodate control requirements.  It is written to operate asynchronously (non-blocking) to avoid watchdog alerts, which means you make the call and be notified when data has been read/written succesfully.  If you need to read multiple variables in one scan, then you can create a lower priority task and place the calls into a WHILE loop or use each scan cycle to generate a for loop (see `Examples/Siemens`).  At least 95% of the library leverages pointers for efficiency, so it might not be straight forward to digest at first.  The documentation / comments is not too bad, but feel free to raise issues if needed.

### Getting started
Create an function block instance in your CoDeSys program, and specify the PLC's IP and port.  Then create some variables:
```
VAR
    _PLC        : CoDeSys_S7Comm.Device(sIpAddress:='192.168.1.219',
                                        uiPort:=102,
                                        bRack:=0,
                                        bSlot:=0);

    _bReadDb    : BOOL;     // create var for write (e.g. _bWriteDb)
    _bReadDb1   : BYTE;     // create var for write 
    _siReadDb   : SINT;     // create var for write 
    _usiReadDb  : USINT;    // create var for write 
    _iReadDb    : INT;      // create var for write 
    _uiReadDb   : UINT;     // create var for write 
    _diReadDb   : DINT;     // create var for write 
    _udiReadDb  : UDINT;    // create var for write 
    _liReadDb   : LINT;     // create var for write 
    _uliReadDb  : ULINT;    // create var for write 
    _wReadDb    : WORD;     // create var for write 
    _dwReadDb   : DWORD;    // create var for write 
    _rReadDb    : REAL;     // create var for write 
    _lrReadDb   : LREAL;    // create var for write 
    _stReadDb   : stString; // create var for write 

END_VAR
```

In your code, toggle `bEnable` of _PLC to `TRUE`.  There is optional `bAutoReconnect` to re-establish session if terminated from idling.

#### Data alignment:
Siemens is 0 bytes aligned, so make sure you specify CoDeSys STRUCTs with `{attribute 'pack_mode' := '0'}`.  Read the [CoDeSys pack mode](https://help.codesys.com/webapp/_cds_pragma_attribute_pack_mode;product=codesys;version=3.5.16.0).

#### Reading Data Block
**NOTE**:
* Possible arguments for `bReadDB`:
    * `uiIndex` (UINT) [**required**]: Data block index number.
    * `udiOffset` (UDINT) [**required**]: Address offset of the data block
    * `eDataSize` (ENUM) [**required**]: Data type size (e.g. BOOL has size of 1 BYTE).  Choose _STRUCT when reading STRING or custom STRUCT
    * `pbBuffer` (POINTER TO BYTE) [**required**]: Pointer to the output buffer.
    * `uiBufferSize` (UINT): Size of the output buffer (pbBuffer).
        * Default: `0`
    * `uiElements` (UINT): Elements to be requested.
        * **Note**: Not implemented.
        * Default: `0`
    * `psId` (POINTER TO STRING): Pointer to caller id [e.g. psId:=ADR('Read#1: ')].
        * Useful for troubleshooting. If you incorrectly declare your data type , a `Data Type Inconsistent` would typically be returned.  By declaring `psId:=ADR('Read#1: ')`, sError will return `'Read#1: Data Type Inconsistent'` to let you know of the error.
* `bReadDB` returns TRUE on successful read.

Below reads a PLC's variable with data type of **BOOL** at address offset 0 of data block #2 and writes to a CoDeSys variable with data type **BOOL** called `_bReadDb`.
```
_PLC.bReadDB(uiIndex:=2,
            udiOffset:=0,
            eDataSize:=CoDeSys_S7Comm.eDataType._BOOL,
            pbBuffer:=ADR(_bReadDb),
            uiBufferSize:=SIZEOF(_bReadDb),
            psId:=ADR('Read#0: '));
```

*
*
*

Below reads a PLC's variable with data type of **UDINT** at address offset 12 of data block #2 and writes to a CoDeSys variable with data type **UDINT** called `_udiReadDb`.
```
_PLC.bReadDB(uiIndex:=2,
            udiOffset:=12,
            eDataSize:=CoDeSys_S7Comm.eDataType._UDINT,
            pbBuffer:=ADR(_udiReadDb),
            uiBufferSize:=SIZEOF(_udiReadDb),
            psId:=ADR('Read#7: '));
```

*
*
*

Below reads a PLC's variable with data type **LREAL** at address offset 42 of data block #2 and writes to a CoDeSys variable with data type **LREAL** called `_lrReadDb`.
```
_PLC.bReadDB(uiIndex:=2,
            udiOffset:=42,
            eDataSize:=CoDeSys_S7Comm.eDataType._LREAL,
            pbBuffer:=ADR(_lrReadDb),
            uiBufferSize:=SIZEOF(_lrReadDb),
            psId:=ADR('Read#13: '));
```
Below reads a PLC's variable with data type **STRING** at address offset 42 of data block #2 and writes to a CoDeSys variable with data type **STRUCT** called `_stReadDb`.
**Note**: Keep.
```
_PLC.bReadDB(uiIndex:=2,
            udiOffset:=50,
            eDataSize:=CoDeSys_S7Comm.eDataType._STRING,
            pbBuffer:=ADR(_stReadDb),
            uiBufferSize:=SIZEOF(_stReadDb),
            psId:=ADR('Read#14: '));
```

### Writing Data Block
**NOTE**:
* Siemens PLCs use Big Endian, so you will need to most likely swap bytes when reading a custom STRUCT.  Known data types have been swapped for you already.
* Possible arguments for `bWriteDB`:
    * `uiIndex` (UINT) [**required**]: Data block index number.
        * You can also point to a STRING instead [e.g. psTag:=ADR(_sMyTestString)].
    * `udiOffset` (UINT) [**required**]: Address offset of the data block
    * `eDataSize` (ENUM) [**required**]: Data type size (e.g. BOOL has size of 1 BYTE).  Choose _STRUCT when writing custom STRUCT
    * `pbBuffer` (POINTER TO BYTE) [**required**]: Pointer to the output buffer.
    * `uiBufferSize` (UINT): Size of the output buffer (pbBuffer).
        * Default: `0`
    * `uiElements` (UINT): Elements to be requested.
        * **Note**: Not implemented.
        * Default: `0`
    * `psId` (POINTER TO STRING): Pointer to caller id [e.g. psId:=ADR('Write#1: ')].
        * Useful for troubleshooting. If you incorrectly declare your data type , a `Data Type Inconsistent` would typically be returned.  By declaring `psId:=ADR('Write#1: ')`, sError will return `'Write#1: Data Type Inconsistent'` to let you know of the error.
* `bWriteDB` returns TRUE on successful write

Below writes a CoDeSys variable with data type **BOOL** called `_bWriteDb` to a PLC's variable with data type of **BOOL** at address offset 0 of data block #2.
```
_PLC.bWriteDB(uiIndex:=2,
            udiOffset:=0,
            eDataSize:=CoDeSys_S7Comm.eDataType._BOOL,
            pbBuffer:=ADR(_bWriteDb),
            uiBufferSize:=SIZEOF(_bWriteDb),
            psId:=ADR('Write#0: '));
```
*
*
*

Below writes a CoDeSys variable with data type **UDINT** called `_udiWriteDb` to a PLC's variable with data type **UDINT** at address offset 12 of data block #2.
```
 _PLC.bWriteDB(uiIndex:=2,
            udiOffset:=12,
            eDataSize:=CoDeSys_S7Comm.eDataType._UDINT,
            pbBuffer:=ADR(_udiWriteDb),
            uiBufferSize:=SIZEOF(_udiWriteDb),
            psId:=ADR('Write#7: '));
```
*
*
*

Below writes a CoDeSys variable with data type **LREAL** called `_lrReadDb` to a PLC's variable with data type **LREAL** at address offset 42 of data block #2.
```
_PLC.bWriteDB(uiIndex:=2,
            udiOffset:=42,
            eDataSize:=CoDeSys_S7Comm.eDataType._LREAL,
            pbBuffer:=ADR(_lrWriteDb),
            uiBufferSize:=SIZEOF(_lrWriteDb),
            psId:=ADR('Write#13: '));
```
Below writes a CoDeSys variable with data type **STRUCT** called `_stWriteDb`to a PLC's variable with data type **STRING** at address offset 52 of data block #2.
**NOTE:** Writing to a PLC string must follow the format of a STRUCT made up of maxLen (BYTE), actualLen (BYTE), and a STRING.  These will be set during write!  See `Examples/Siemens` folder for more details
```
_PLC.bWriteDB(uiIndex:=2,
            udiOffset:=50,
            eDataSize:=CoDeSys_S7Comm.eDataType._STRING,
            pbBuffer:=ADR(_stWriteDb),
            uiBufferSize:=SIZEOF(_stWriteDb),
            psId:=ADR('Write#14: '));
```

### But does it work?
Yes... 60% of the time, it works every time.  Testing was done using a Raspberry Pi 3, with CoDeSys 3.5.16.0 runtime installed, to communicate with a Siemens `CPU 1517F-3 PN/DP` PLC.  You will need to install SysTime and OSCAT Basic (this is for time formatting).

### Current features
#### Get CPU Info
`bGetCpuInfo()` (BOOL) is automatically called after TCP connection is established to return device info. You could scan your network for other Siemens PLCs.  
* **Examples:**
    * Retrieve single parameter as UINT using: `_uiOemId := _PLC.uiOemId;`
        * **Output:** `0`
    * Retrieve single parameter as STRING using: `_sOemId := _PLC.sOemId;`
        * **Output:** `'0x0000'`
    * Retrieve entire STRUCT using: `_stDevice := _PLC.stCpuInfo;`
        * Requires STRUCT variable: `_stDevice : CoDeSys_S7Comm.stCpuInfo;`
        * **Output:**
            * sSystemName: `'S71500/ET200MP station_1'`
            * sModuleName: `'PLC_1'`
            * uiPlantId: `16#2020`
            * sPlantId: `'0x2020'`
            * sCopyright: `'Original Siemens Equipment'`
            * sSerialNumber: `'S C-J3P979902017'`
            * sCpuType: `'CPU 1517F-3 PN/DP'`
            * sMcSerialNumber: `'SMC_3b64ee0208  '`
            * uiManufacturerId: `16#002A`
            * sManufacturerId: `'0x002A'`
            * uiProfileId: `16#F600`
            * sProfileId: `'0xF600'`
            * uiProfileSpec: `16#0001`
            * sProfileSpec: `'0x0001'`
            * sOemCopyright: `''`
            * uiOemId: `16#0000`
            * sOemId: `'0x0000'`
            * udiOemAddId: `16#00000000`
            * sOemAddId: `'0x00000000'`
            * sLocationId: `'Test Lab'`

#### Get/Set PLC time
Currently under development and will be released in the next revision.
<!--
`bGetPlcTime()` (BOOL) requests the current PLC time.  The function can handle 64b time up to nanoseconds resolution, but the PLC's accuracy is only available at the microseconds.
* **Example:**
    * Retrieve time as STRING: `_sPlcTime := _PLC.sPlcTime;`
        * **Output:** `'LDT#2020-07-10-01:05:59.409036000'`
    * Retrieve time in microseconds as ULINT: `_uliPlcTime := _PLC.uliPlcTime;`
        * **Output:** `1593815478238754`

`bSetPlcTime(ULINT)` (BOOL) sets the PLC time.
* **Examples:**
    * Synchronize PLC's time to IPC's time: `bSetPlcTime()`
    * Set a PLC time to `Friday, July 3, 2020 10:31:18 PM GMT` with seconds level accuracy in microseconds: `bSetPlcTime(1593815478000000)`
    * **NOTE:** Look at built-in `Timestamp` function block
-->

### Useful parameters (SET/GET)
**NOTE:** There are a lot more, so dive into library to see what works best for you
* `bAutoReconnect` (BOOL) re-establishes session if disconnected.
    * Default: `FALSE`
    * Example set: `_PLC.bAutoReconnect := TRUE;`
    * Example get: `_bReconnect := _PLC.bAutoReconnect;`
* `bHexPrefix` (BOOL) attaches the '0x' prefix to the strings from CPU info STRUCT.
    * Default: `TRUE`
* `bRack` (BYTE) specifies the module rack.
    * Default: `0`
* `bSlot` (BYTE) specifies the module slot.
    * Default: `0`
* `sDeviceIp` (STRING) allows you to change device IP from the one specified initially.
    * Default: `'11.200.0.10'`
    * Toggle bEnable to update settings
* `uiDevicePort` (UINT) allows you to change device port from the one specified initially.
    * Default: `102`
    * Toggle bEnable to update settings
* `udiTcpWriteTimeout` (UDINT) specifies the maximum time it should take for the TCP client write to finish.
    * Default: `200000` microseconds
* `uiCommWriteTimeout` (UINT) specifies the maximum time it should take for a write request to finish.
    * Default: `200` milliseconds
* `udiTcpClientTimeOut` (UDINT) specifies the TCP client timeout.
    * Default: `500000` microseconds
* `udiTcpClientRetry` (UDINT) specifies the auto reconnect interval for `bAutoReconnect`.
    * Default: `5000` milliseconds

### Useful variable lists:
* `gvcParameters` holds the default values from above, so adjust values to fit your needs.

### Useful methods:
* `bDisconnect()` (BOOL) sends a disconnect session request. 
    * **NOTE:** If `bAutoReconnect` is set to TRUE, session will be re-established within the specified `udiTcpClientRetry` period.
* `bResetFault()` (BOOL) call this to ACK read/write fault flags.