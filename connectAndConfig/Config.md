---
sort: 2
---

## 相关配置
&emsp;&emsp;宇树提供了电机工具箱`Unitree MotorTools`让用户可以更方便地进行配置工作，如更改ID、电机固件升级等。解压 Unitree MotorTools工具箱，进入bin文件夹后可以看到一些可执行程序，这里提供了常见的一些对电机修改的工具。
<center>
<img src="./img/unitree_motortools.png" style="zoom:100%" alt=" 图片不见了。。。 "/>
<br>
<div style="color:orange; border-bottom: 0.1px solid #d9d9d9;
display: inline-block;
color: #999;
padding: 1px;">Unitree MotorTools工具箱</div>
</center>
<br>



### 查看串口名
&emsp;&emsp;将USB转RS-485转接口连接在上位机上时，上位机会为这个串口分配一个串口名。在Linux系统中，这个串口名一般是以“ttyUSB”开头，在Windows系统中，串口名往往以“COM”开头。
&emsp;&emsp;在Linux系统中，一切外接设备都是以文件形式存在的。USB转RS-485转接器也可以被视为/dev文件夹下的一个“文件”。打开任意一个终端窗口（在Ubuntu下快捷键为Ctrl+Alt+t组合键），运行如下命令：
```
cd /dev
ls | grep ttyUSB
```
&emsp;&emsp;其中cd /dev命令将当前文件夹切换为/dev， ls |grep ttyUSB命令显示当前文件夹下所有文件名包含ttyUSB的文件，其中的 | 符号就在键盘的回车键上方，按住Shift+\即可键入”|”字符。运行如上命令后，即可得到上位机当前连接的串口名。例如图3所示，当前上位机连接的串口名为ttyUSB0。考虑到串口所在的文件夹路径，其完整的串口名为/dev/ttyUSB0。
```note
串口名的序号和插入设备的顺序一致，对于连接多个设备这很有帮助。
```
