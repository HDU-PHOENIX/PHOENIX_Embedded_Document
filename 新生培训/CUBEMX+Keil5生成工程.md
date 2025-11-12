# CUBEMX + Keil5 生成工程
> 本文档介绍如何使用 CUBEMX 生成适用于 Keil5 的 STM32 工程， 通用步骤如下：

## 1. 选择微控制器型号
点击`ACCESS TO MCU SELECTOR`，可能会需要更新，等待即可，如果是提示未注册，可以看看群里的安装手册，本文档不再赘述，搜索芯片型号，选择对应芯片，双击打开
![](https://youke1.picui.cn/s1/2025/11/08/690f26601be3d.png)

## 2. 三个基础配置，全工程通用
### 2.1 时钟配置
选择`RCC`，配置`High Speed Clock`为`Crystal/Ceramic Resonator`
![](https://youke1.picui.cn/s1/2025/11/08/690f278f27555.png)
### 2.2 设置Debug接口
选择`SYS`，配置`Debug`为`Serial Wire`
![](https://youke1.picui.cn/s1/2025/11/08/690f27c856bde.png)
### 2.3 配置时钟树
选择`Clock Configuration`，把`HCLK`设置为`72MHz`，其他保持默认即可，注意检查时钟晶振是否是`8MHz`，且时钟来源与图中所示路径相同
![](https://youke1.picui.cn/s1/2025/11/08/690f289c2798e.png)

## 3. 生成工程
### 3.1 工程设置
选择`Project Manager`填写工程名称，选择要生成的工程路径，选择工具链为`MDK-ARM`!!!
![](https://youke1.picui.cn/s1/2025/11/08/690f298090535.png)
### 3.2 代码配置
选择`Code Generator`，勾选`Copy only the necessary files`和`Generate peripheral initialization as a pair of .c/.h files per peripheral`，最后点击`Generate`按钮即可生成工程
![](https://youke1.picui.cn/s1/2025/11/08/690f2a2caef31.png)

## 4. Keil5配置
### 4.1 打开工程
在生成工程后点击`Open Project`按钮即可打开 Keil5 软件，或者找到对应的工程路径下，找到`MDK-ARM`文件夹，双击`xxx.uvprojx`文件即可打开工程
### 4.2 检查cubemx配置
点击`Build`，如果显示0error，0warning，说明配置正确，可以进行后续开发
### 4.3 配置编译器
点击魔术棒，在弹出的窗口中选择`Target`选项卡，配置`ARM Compiler`为自己安装的版本。如果不清楚自己哪一个能用，可以全部尝试一下，直到编译通过为止
![](https://youke1.picui.cn/s1/2025/11/08/690f2bf2bbafb.png)
### 4.4 配置编译选项
依旧在魔术棒中，选择`C/C++`选项卡，设置`Optimization`为`Level 0 (-O0)`，以便于调试
### 4.5 配置链接器选项
还是在魔术棒中，选择`Debug`选项卡，配置烧录器为自己使用的，一般是`ST-Link `，点击`Settings`按钮，选择`Port`为`SW`，把`Flash Download`中的`Reset and Run`选项勾选上
![](https://youke1.picui.cn/s1/2025/11/08/690f2cb9c58eb.png)

## 5. 添加文件
### 5.1 添加源文件
点击“三色块”，新建一个组，然后点击`Add Files`,把需要添加的源文件添加进去即可
![](https://youke1.picui.cn/s1/2025/11/08/690f2e26f0162.png)
### 5.2 添加头文件路径
点击魔术棒，选择`C/C++`选项卡，在`Include Paths`中添加头文件路径即可
![](https://youke1.picui.cn/s1/2025/11/08/690f2ef761af9.png)

## 6. 使用VsCode写Keil5工程
> 为什么要到vscode写Keil5工程：因为Keil5自带的编辑器太过简陋，且代码提示路边一条，而VsCode作为一款强大的代码编辑器，拥有丰富的插件生态和强大的功能，可以极大提升开发效率和体验
### 6.1 下载vscode
下载地址：https://code.visualstudio.com/
### 6.2 安装基础插件
打开vscode，点击左侧扩展图标，搜索`C/C++`，搜索`Chinese(Simplified)`，安装即可
### 6.3 安装keil5插件
搜索`Keil`，安装`Keil Assiatant`插件
![](https://youke1.picui.cn/s1/2025/11/08/690f412a5f6e4.png)
### 6.4 配置插件
进入插件配置界面，根据自己的安装路径，配置`ARM Path`，配置完成后重启vscode
### 6.5 打开工程
点击主界面下的`KEIL UVSION PROJECT`,点击右边的加号，在跳出来的的窗口里选择对应工程的`xxx.uvprojx`文件，双击即可打开工程。

<strong style="font-size: 2em;">恭喜你，你已经成功配置好了 CubeMX + Keil5 + Vscode 的开发环境，可以愉快地进行 STM32 开发了！</strong>
> Copyright (c) 2025 PHOENIX
> 作者： 何文轩 <wenx_public@163.com>
> 说明：本文件版权由杭州电子科技大学PHOENIX实验室持有