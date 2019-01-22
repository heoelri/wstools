SYMPROXY

1. Install the debugging tools to c:\debuggers

The debugging tools are available via
https://developer.microsoft.com/en-us/windows/hardware/windows-driver-kit

2. Install the debugging tools to c:\debuggers

The debugging tools are available via https://developer.microsoft.com/en-us/windows/hardware/windows-driver-kit

3. Import the following reg file

```
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Symbol Server Proxy] 
"Available Settings"="Remove the 'x' prefix to use the setting"
"xNoInternetProxy"=dword:00000001
"xNoLongerIndexedAuthoritive"=dword:00000001
"xNoFilePointers"=dword:00000001
"xNoUncompress"=dword:00000001
"xNoCache"=dword:00000001
"xRequestTimeout"=dword:00000019
"xRetryAppHang"=dword:00000002
"xMissAgeTimeout"=dword:00015180
"xMissAgeCheck"=dword:00000e10
"xMissFileCache"=dword:00000001
"xMissFileThreads"=dword:00000010
"xFailureCount"=dword:00000004
"xFailurePeriod"=dword:00000078
"xFailureTimeout"=dword:0000002d
"xFailureBlackout"=dword:00000384
"Urifilter"=dword:00000017
"UriTiers"=dword:00000001
"MissTimeout"=dword:00000384
"NoInternetProxy"=dword:00000001

[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Symbol Server Proxy\Web Directories]

[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Symbol Server Proxy\Web Directories\Symbols]
"SymbolPath"="https://msdl.microsoft.com/download/symbols"
```

3. Install the iis role with isapi filters enabled.

```powershell
install-windowsfeature web-server, Web-ISAPI-Filter -includemanagementtools
```

4. Remove the default web site
5. Create a folder d:\symbols
6. Set file and share permissions on d:\symbols

Everyone:  Read Write On file and share
Grant Read\Write to the SymProxy App Pool user account (Domain\User) to the folder

7.  You need two files  to install the symproxy now. The files  are "install.cmd" and  "staticContentClear.xml"   
Generate these files with the content below and  copy them to  c:\debuggers\symproxy  
 
Install.cmd
 
```cmd 
@echo off
 
SET VirDirectory=%1
SET UserName=%2
SET Password=%3
 
::
::  SymProxy dll installation. 
::
 
:: The  files below are in the debugger subdirectory symproxy . Adjust pathes accordingly
 
copy symproxy.dll %windir%\system32\inetsrv
copy symproxy.man %windir%\system32\inetsrv
copy symsrv.dll %windir%\system32\inetsrv
 
lodctr.exe /m:%windir%\system32\inetsrv\symproxy.man
wevtutil.exe install-manifest %windir%\System32\inetsrv\symproxy.man
regedit.exe /s symproxy.reg
 
::
::  Web server Configuraiton
::
 
IF not exist %VirDirectory% mkdir %VirDirectory%
 
rem Make the 'Default Web Site'
%windir%\system32\inetsrv\appcmd.exe add site -site.name:"Default Web Site" -bindings:"http/*:80:" -physicalPath:C:\inetpub\wwwroot
 
rem Enabled Directory Browsing on the 'Default Web Site'
%windir%\system32\inetsrv\appcmd.exe set config "Default Web Site" -section:system.webServer/directoryBrowse /enabled:"True"
 
rem Make the 'SymProxy App Pool'
%windir%\system32\inetsrv\appcmd.exe add apppool -apppool.name:SymProxyAppPool -managedRuntimeVersion:
%windir%\system32\inetsrv\appcmd.exe set apppool -apppool.name:SymProxyAppPool -processModel.identityType:SpecificUser -processModel.userName:%UserName% -processModel.password:%Password% 
 
rem Make the 'Symbols' Virtual Directory and assign the 'SymProxy App Pool'
%windir%\system32\inetsrv\appcmd.exe add app -site.name:"Default Web Site" -path:/Symbols -physicalpath:%VirDirectory%
%windir%\system32\inetsrv\appcmd.exe set app -app.name:"Default Web Site/Symbols" -applicationPool:SymProxyAppPool
 
rem Disable 'web.config' for folders under virtual directories in the 'Default Web Site'
%windir%\system32\inetsrv\appcmd.exe set config -section:system.applicationHost/sites "/[name='Default Web Site'].virtualDirectoryDefaults.allowSubDirConfig:false
 
rem Add the 'SymProxy ISAPI Filter'
%windir%\system32\inetsrv\appcmd.exe set config -section:system.webServer/isapiFilters /+"[name='SymProxy',path='%windir%\system32\inetsrv\SymProxy.dll',enabled='True']
 
rem Clear the MIME Types on the 'Default Web Site'
%windir%\system32\inetsrv\appcmd.exe set config -in "Default Web Site" < staticContentClear.xml
 
rem Add * to the MIME Types of the 'Default Web Site'
%windir%\system32\inetsrv\appcmd.exe set config "Default Web Site" -section:staticContent /+"[fileExtension='.*',mimeType='application/octet-stream']"
``` 
  
StaticContentClear.xml is used by install.cmd to configure iis. It must look as follows: 

```xml
<?xml version="1.0" encoding="UTF-8"?>
<appcmd>
    <CONFIG CONFIG.SECTION="system.webServer/staticContent" path="MACHINE/WEBROOT/APPHOST">
        <system.webServer-staticContent>
            <clear />
        </system.webServer-staticContent>
    </CONFIG>
</appcmd>
``` 
 
8. Use install.cmd to configure iis as a symproxy
 
The Install.cmd script requires 3 parameters:
 * Virtual Directory path (e.g. D:\SymStore\Symbols )
 * Username (for the Application Pool)
 * Password (for the Application Pool)
 
To clear the MIME Type inheritance, an XML file is needed to drive the associated AppCmd.exe command. Place the staticContentClear.xml file shown below in the same folder as the Install.cmd script to achieve this result.
 
Example Install.Cmd parameter usage:
 
Open up an administrative command prompt.
```cmd 
cd  c:\debuggers\symproxy 
 
Install.cmd D:\Symbols CONTOSO\SymProxyService Pa$$word
```

where Domain\user is the name of the user you granted permissions on the folder.  This user will be configured as the app pool user. 
 
9. Start the debugger c:\debuggers\windbg 
10. Start notepad under the debugger 
 
11. Set symbol path to Http://localhost/symbols produces the following symbols loading output.

12. In the debugger command prompt type 

```
!sym noisy
.reload /f notepad.exe
```

when these commands return the proxy is running 
13. Use 

```powershell
get-windowsupdatelog -symbolserver  Http://localhost/symbols 
```

to decode the Windows Update Logs 

14. You might have to handle some issues with your local proxy. The experience is that you need a proxy set at system level, when a proxy is in your environment.

15.  optionally set  a symbols proxy  environment variable   like the following.

```cmd
set _nt_symbol_path=Http://localhost/symbols
```

where you need to replace Localhost by the computername of the proxy
