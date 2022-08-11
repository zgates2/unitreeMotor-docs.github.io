---
sort: 2
---

## 相关配置
### 查看串口名
&emsp;&emsp;将USB转RS-485转接口连接在上位机上时，上位机会为这个串口分配一个串口名。在Linux系统中，这个串口名一般是以“ttyUSB”开头，在Windows系统中，串口名往往以“COM”开头。
&emsp;&emsp;在Linux系统中，一切外接设备都是以文件形式存在的。USB转RS-485转接器也可以被视为/dev文件夹下的一个“文件”。打开任意一个终端窗口（在Ubuntu下快捷键为Ctrl+Alt+t组合键），运行如下命令：
```
cd /dev
ls | grep ttyUSB
```
&emsp;&emsp;其中cd /dev命令将当前文件夹切换为/dev， ls |grep ttyUSB命令显示当前文件夹下所有文件名包含ttyUSB的文件，其中的 | 符号就在键盘的回车键上方，按住Shift+\即可键入”|”字符。运行如上命令后，即可得到上位机当前连接的串口名。例如图3所示，当前上位机连接的串口名为ttyUSB0。考虑到串口所在的文件夹路径，其完整的串口名为/dev/ttyUSB0。