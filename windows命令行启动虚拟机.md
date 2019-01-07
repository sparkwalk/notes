### Windows命令行启动虚拟机

1. 启动虚拟机

   ```
   @
   echo "poweron the apple virtual machine..."
   
   @"D:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe" -T ws start "D:\VirtualMachines\apple\apple.vmx" nogui
   ```

2. 关闭虚拟机

   ```
   @echo "poweroff the apple virtual machine..."
   
   @"D:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe" -T ws stop "D:\VirtualMachines\apple\apple.vmx" nogui
   ```
