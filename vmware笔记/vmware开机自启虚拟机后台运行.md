## 创建`vmware_start.bat`
```batch
@echo off

set VMRun="C:\Program Files\VMware\VMware Workstation\vmrun.exe"

%VMRun% ^
    -T ws ^
    start ^
        "C:\Users\59507\Documents\Virtual Machines\Ubuntu-22.04.5\Ubuntu-22.04.5.vmx" ^
    nogui

```

## 创建`vmware_start.vbs`
```basic
Dim ws
Set ws = Wscript.CreateObject("Wscript.Shell")
ws.run "vmware_start.bat",vbhide
Wscript.quit
```

