## 创建`vmware_start.bat`
```batch
@echo off
set VMRun="D:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe"

:: 预设参数变量
set VM_PATH="E:\Users\59507\Documents\Virtual Machines\ubuntu-22.04.3\ubuntu-22.04.3.vmx"
set OPTS=-T ws start %VM_PATH% nogui

:: 最终执行一行指令
%VMRun% %OPTS%

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

## 创建`vmware_start.vbs`
```basic
Dim ws
Set ws = Wscript.CreateObject("Wscript.Shell")
ws.run "vmware_start.bat",vbhide
Wscript.quit
```

