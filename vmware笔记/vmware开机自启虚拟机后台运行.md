```batch
@echo off

set VMRun="D:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe"
%VMRun% ^
-T ws ^
start ^
"E:\Users\59507\Documents\Virtual Machines\CentOS 7\CentOS 7.vmx" ^
nogui

%VMRun% ^
-T ws ^
start ^
"E:\Users\59507\Documents\Virtual Machines\ubuntu-22.04.3\ubuntu-22.04.3.vmx" ^
nogui

```

```basic
Dim ws
Set ws = Wscript.CreateObject("Wscript.Shell")
ws.run "vmware_start.bat",vbhide
Wscript.quit
```

