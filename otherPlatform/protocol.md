---
sort: 1
---

## 相关移植配置
&emsp;&emsp;GO-M8010-6电机采用串口通信，通信标准为RS-485，波特率为4.0 Mbps。串口的数据位为8 bit，无奇偶校验位，停止位为1 bit。需要注意的是为了提高电机的通信频率，我们使用了4.0 Mbps这一很高的波特率，用户需要检查自己的硬件是否支持这么高的波特率。
如果您的硬件无法支持该波特率，可以使用Unitree提供的USB转RS-485模块。

## 电机运动控制指令格式
&emsp;&emsp;在控制电机时，我们会通过串口给电机发送一个长度为17字节的命令，之后电机会返回一个长度为16字节的信息。如果不给电机发送命令，那么电机也不会返回状态。给电机发送命令的格式如表1，电机返回的状态格式如表2所示。这两个表中详细介绍了收发报文中各个字节表示的含义，下面我们会解释其中的部分细节。<br>
&emsp;&emsp;首先是发送给电机命令中的第11、12字节，这两个字节表示了电机前馈力矩，显然前馈力矩是一个浮点数，即float型，一个float型浮点数需要占用4个字节。为了节约通信带宽，我们用2个字节来表示浮点数，我们使用的方法是移位操作，在此不对具体原理展开讲解。从应用的角度，读者可以认为我们对前馈力矩乘了256，之后赋值给一个2个字节的signed short int型，即带符号的短整形变量。在这个赋值过程中会强制取整，这样我们就可以只用两个字节来发送前馈力矩，当电机接收到这个数据后，只需要除以256，就能获得前馈力矩的数值。这种操作虽然会丧失一些精度，但是对于实际应用是完全足够的。另外需要注意的一点就是，对于一个长度为2字节的变量，一共有16个字符，即16位。其中1位用来表示正负，是符号位，所以只有15位用于表示数值的大小，这意味着命令中T数值不能大于215。考虑到我们曾经对原始数据乘了256，所以其绝对值存在上限：<br>
<center>
$$ \lvert \tau_{ff} \rvert<  \frac{2^{15}}{256} = 128 $$
</center>
&emsp;&emsp;同时需要注意的是，由于在赋值过程中存在强制取整，所以的数值越大，保存的小数精度越低。表1中变量$$\tau$$的说明里所说的$$x256$$倍描述指的就是上文中所说的乘256，其他变量中所说的某某倍描述也与之同理。<br>
&emsp;&emsp;并且，对于2个字节的  变量来说，它的低位在前，即第13字节，高位在后，即第14字节。在发送命令和接收状态的末尾，我们可以看到一个2字节的CRC_CCITT校验。在命令发送之前，我们会计算这些命令字节的发送前CRC校验值，并且和命令一起发送给电机。当电机收到命令之后，还会根据收到的命令计算发送后CRC校验值。<br>
&emsp;&emsp;如果数据传输过程中没有发生任何错误，那么发送前CRC校验值等于发送后CRC校验值。如果在数据传输过程中出现了数据错误，那么发送后CRC校验值的计算结果就与发送前CRC校验值不相等，电机就能够知道数据发生了损坏，从而帮助我们避免错误数据。读者可以直接参考Linux内核中关于crc_ccitt函数的源码。

<table>
    <tr>
        <td>类型(type)</td>
        <td>位(bit)</td>
        <td>符号</td>
        <td>说明</td>
        <td>Value</td>
    </tr>

    <tr>
        <td>
            包头<br>
            (2Byte)
        </td>
        <td>
            0-15<br>
            (2Byte)
        </td>
        <td>
            HEAD
        </td>
        <td>
            数据包头部
        </td>
        <td>
            0xFD 0xEE<br>
            注意:与来自主机方向的包头不同
        </td>
    </tr>

    <tr>
        <td rowspan="3">
            模式设置<br>
            (1Byte)
        </td>
        <td>
            16-19<br>
            （4bit）
        </td>
        <td>
            ID
        </td>
        <td>
            目标电机ID
        </td>
        <td>
            0,1,2,3 … 13,14<br>
            15.保留
        </td>
    </tr>

    <tr>
        <td>
            20-22<br>
            （3bit）
        </td>
        <td>
            STATUS
        </td>
        <td>
            电机工作模式
        </td>
        <td>
            0.锁定(Default)<br>
            1.FOC闭环<br>
            2.编码器校准<br>
            3-7.保留
        </td>
    </tr>

    <tr>
        <td>
            23<br>
            （1bit）
        </td>
        <td colspan="3">
            保留
        </td>
    </tr>

    <tr>
        <td rowspan="5">
            控制参数<br>
            (12byte)
        </td>
        <td>
            24-39<br>
            （2Byte）
        </td>
        <td>
            $$\tau_{set}$$
        </td>
        <td>
            期望电机转矩值
        </td>
        <td>
            例：
            $$t_{ff} = 0.75(N*m)$$ <br>
            $$\tau_{set} = t_{ff} *256 = 0.75*256 =192 $$ <br>
            $$注：\lvert t_{ff} \rvert \leqslant 127.99N \bullet m $$
        </td>
    </tr>

     <tr>
        <td>
            40-55<br>
            （2Byte）
        </td>
        <td>
            $$\omega_{set}$$
        </td>
        <td>
            期望电机速度
        </td>
        <td>
           例：
           $$\omega_{des} = 90 = \pi/2(rad/s)$$ <br>
           $$\omega_{set} = \omega_{des}/2 \pi * 256 = 128$$ <br>
           $$注：2\pi = 6.28rad/s = 60PRM$$ <br>
           $$\lvert \omega_{des} \leqslant 804.0rad/s \rvert $$ 
        </td>
    </tr>

     <tr>
        <td>
            56-87<br>
            （4Byte）
        </td>
        <td>
            $$\theta_{set}$$
        </td>
        <td>
            期望电机输出位置(多圈累加)
        </td>
        <td>
           例：
           $$\theta_{des} = \degree{90} = \frac{\pi}{2} = 1.57(rad)$$<br>
           $$\theta_{set} = \frac{\theta_{des}}{2\pi} * 32768 = 8187$$<br>
           $$注：2\pi = \degree{360} = 6.2831rad$$<br>
           $$\lvert \theta_{des} \rvert \leqslant 411774rad(65535圈)$$
        </td>
    </tr>

     <tr>
        <td>
            88-103<br>
            （2Byte）
        </td>
        <td>
            $$K_{pos}$$
        </td>
        <td>
            电机刚度系数/
            位置误差比例系数(多圈累加)
        </td>
        <td>
           例：
           $$k_p = 0.1$$<br>
           $$k_{pos} = k_p * 1280 = 128$$<br>
           $$注：0 \leqslant k_p \geqslant 25.599$$
        </td>
    </tr>

    <tr>
        <td>
            104-119<br>
            （2Byte）
        </td>
        <td>
            $$K_{spd}$$
        </td>
        <td>
            电机阻尼系数/
            速度误差比例系数
        </td>
        <td>
           例：
           $$k_w = 0.2$$<br>
           $$k_{spd} = k_w*1280 = 256$$<br>
           $$注：0 \leqslant k_w \geqslant 25.599$$
        </td>
    </tr>

    <tr>
        <td>
            校验部分<br>
            （2Byte）
        </td>
        <td>
            120-135<br>
            （2Byte）
        </td>
        <td>
            CRC16
        </td>
        <td>
           CRC16校验结果
        </td>
        <td>
           CRC16_CCITT多项式计算0-199位数据的结果
        </td>
    </tr>

      <tr>
        <td>
            共计：
        </td>
        <td colspan="4">
            17 Byte
        </td>
    </tr>

</table>
<center>
<div style="color:orange; border-bottom: 0.1px solid #d9d9d9;
display: inline-block;
color: #999;
padding: 1px;">表1 主机侧控制协议</div>
</center>
```note
为了保证标定效果，切换到编码器校准模式后，需要等待5s再进行通信
（期间不可以给电机发送任何数据包，否则会标定失败)
```
<br>
<br>

<table>
    <tr>
        <td>
            类型(type)
        </td>
        <td>
            位(bit)
        </td>
        <td>
            符号
        </td>
        <td>
            说明
        </td>
        <td>
            Value
        </td>
    </tr>

    <tr>
        <td>
            包头<br>
            (2Byte)
        </td>
        <td>
            0-15<br>
            (2Byte)
        </td>
        <td>
            HEAD
        </td>
        <td>
            数据包头部
        </td>
        <td>
            0xFD 0xEE<br>
            注意:与来自主机方向的包头不同
        </td>
    </tr>

    <tr>
        <td rowspan="3">
            模式信息<br>
            (1Byte)
        </td>
        <td>
            16-19<br>
            [4bit]
        </td>
        <td>
            ID
        </td>
        <td>
            目标电机ID
        </td>
        <td>
            0,1,2,3 … 13,14<br>
            15.保留
        </td>
    </tr>

    <tr>
        <td>
            20-22<br>
            [3bit]
        </td>
        <td>
            STATUS
        </td>
        <td>
            电机工作模式
        </td>
        <td>
            0.锁定(Default)<br>
            1.FOC闭环<br>
            2.编码器校准<br>
            3-7．保留
        </td>
    </tr>

    <tr>
        <td>
            23<br>
            [1bit]
        </td>
        <td colspan="3">
            保留
        </td>
    </tr>

    <tr>
        <td rowspan="7">
            反馈数据<br>
            （11Byte）
        </td>
        <td>
            24-39<br>
            (2Byte)
        </td>
        <td>
            $$\tau_{fbk}$$
        </td>
        <td>
            实际关节输出转矩
        </td>
        <td>
            $$\tau(N*m) = \frac{\tau_{fbk}}{256}$$
        </td>
    </tr>

    <tr>
        <td>
            40-55<br>
            (2Byte)
        </td>
        <td>
            $$\omega_{fbk}$$
        </td>
        <td>
            实际关节输出速度
        </td>
        <td>
            $$\omega(rad/s) = \frac{\omega_{fbk}}{256}*2\pi$$
        </td>
    </tr>

    <tr>
        <td>
            56-87<br>
            (4Byte)
        </td>
        <td>
            $$\theta_{fbk}$$
        </td>
        <td>
            实际关节输出位置(多圈累加)
        </td>
        <td>
            $$\theta(rad) = \frac{\theta_{fbk}}{32768}*2\pi$$
        </td>
    </tr>

    <tr>
        <td>
            88-95<br>
            (1Byte)
        </td>
        <td>
            TEMP
        </td>
        <td>
            电机温度
        </td>
        <td>
            unit: 摄氏度 (int8_t)<br>
            -128~127°C，90℃时触发温度保护，需要电机重新上电后才能控制。
        </td>
    </tr>

    <tr>
        <td>
            96-98<br>
            [3bit]
        </td>
        <td>
            MERROR
        </td>
        <td>
            电机错误标识
        </td>
        <td>
            0.正常<br>
            1.过热<br>
            2.过流<br>
            3.过压<br>
            4.编码器故障<br>
            5-7.保留
        </td>
    </tr>

    <tr>
        <td>
            99-110<br>
            [12bit]
        </td>
        <td>
            FORCE
        </td>
        <td>
            足端力
        </td>
        <td>
            12bit 原始数据  (0-4095)<br>
            物理量单位/范围，待确认
        </td>
    </tr>

    <tr>
        <td>
            111<br>
            [1bit]
        </td>
        <td colspan="3">
            保留
        </td>
    </tr>

     <tr>
        <td>
            校验部分<br>
            (2Byte)
        </td>
        <td>
            112-127<br>
            (2Byte)
        </td>
        <td>
            CRC16
        </td>
        <td>
            CRC16校验结果
        </td>
        <td>
            CRC16_CCITT多项式计算<br>0-111位数据的结果
        </td>
    </tr>

    <tr>
        <td>
            共计:
        </td>
        <td colspan="4">
            16 Byte
        </td>
    </tr>
</table>
<center>
<div style="color:orange; border-bottom: 0.1px solid #d9d9d9;
display: inline-block;
color: #999;
padding: 1px;">表2 电机端反馈数据</div>
</center>
```note
通讯协议内所有的数据类型都为整形，具体大小请参照上表“位(bit)”项描述。
请确保控制主机处理器平台为小端(LSB)模式。
```
```note
没有特殊说明，表中参数都为关节电机的电机转子侧，而非输出端。
	Go-M8010-6电机转子到输出端的传动减速比为: `6.33`
```