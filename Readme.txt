READ_ME.txt

这里盘点几个在 PE 系统里最常用的 Windows 自带命令行工具。在 PE 里按 Win + R 键，输入 cmd 就能打开命令行，关键时刻用这些命令能解决大问题。
一、 DISM（系统映像部署与维护）

这个工具不仅能用来装系统，还能用来给离线的系统注入驱动，功能非常多。
1. 把系统镜像释放到 C 盘

不管是 WIM 还是 ESD 格式的镜像，都可以用这条命令解压到目标分区（比如 C 盘）：
dism /Apply-Image /ImageFile:D:\sources\install.wim /Index:1 /ApplyDir:C:\

    /ImageFile: 后面填你镜像的实际路径。

    /Index:1 代表释放镜像里的第一个版本（比如专业版），可以用 dism /Get-WimInfo /WimFile:D:\sources\install.wim 来提前查看所有版本对应的编号。

    /ApplyDir: 后面填你要安装的目标盘符。

2. 检查系统映像健康状态

如果原系统损坏了，可以用这些命令来扫描和修复：

    只扫描不修复（看看有没有坏）：
    dism /Image:C:\ /Cleanup-Image /CheckHealth

    深度扫描并尝试修复（需要联网或指定干净源）：
    dism /Image:C:\ /Cleanup-Image /RestoreHealth

3. 给装好的系统注入驱动

如果重装系统后发现没有网卡驱动，或者蓝屏提示缺少某种驱动，可以直接在 PE 里把驱动打进 C 盘的系统里：
dism /Image:C:\ /Add-Driver /Driver:E:\Drivers /Recurse

    /Driver: 后面填你存放驱动解压文件的文件夹路径。

    /Recurse 代表自动查找子文件夹里的所有驱动。

二、 BCDBOOT（引导文件创建与修复）

现在绝大多数电脑都是 UEFI 启动模式了，如果系统装好了却开不了机，通常就是少了这个引导文件。
1. 修复或创建 UEFI 引导（最常用）

如果你的硬盘是 GPT 格式，且有一个专门的 EFI 分区（假设盘符被分配为了 S 盘），用这条命令把 C 盘系统的引导文件写进 EFI 分区：
bcdboot C:\Windows /s S: /f UEFI /l zh-cn

    /s S: 指定引导文件写到哪个分区（S 是你的 EFI 分区盘符）。

    /f UEFI 明确指定只创建 UEFI 格式的引导。

    /l zh-cn 把启动菜单和报错信息设置为简体中文。

2. 修复传统的 BIOS (Legacy) 引导

如果老电脑用的依然是 MBR 分区：
bcdboot C:\Windows /s C: /f BIOS /l zh-cn

    传统引导可以直接把引导文件写在 C 盘自己身上。

三、 DISKPART（高级磁盘管理）

这个工具专门用来处理硬盘分区。下面是全套流程，涵盖了转换 GPT、创建 EFI/MSR 分区以及调整分区大小。
1. 全盘清空并创建完整的 UEFI 结构分区（适合新硬盘或重装）

输入 diskpart 进入工具，然后依次敲下面的命令：

    list disk （看清楚你的目标硬盘是编号几，比如是 0 ）。

    select disk 0 （选定这块盘）。

    clean （清空全盘，数据全丢，看准了再弄）。

    convert gpt （把磁盘改成 GPT 格式，这是 UEFI 启动的基础）。

    create partition efi size=100 （创建 100MB 的 EFI 引导分区）。

    format quick fs=fat32 label="EFI" （快速格式化为 FAT32，并命名为 EFI）。

    assign letter=S （临时分配盘符为 S，方便刚才提到的 bcdboot 写引导）。

    create partition msr size=16 （创建 16MB 的 MSR 微软保留分区，装 Win10/11 必备）。

    create partition primary （把剩下的所有空间都给主分区，也就是 C 盘）。

    format quick fs=ntfs label="System" （快速格式化为主分区）。

    assign letter=C （分配盘符为 C）。

2. 调整磁盘大小（无损压缩或扩展）

如果你不想清空全盘，只是想调整现有分区的大小：

    把某个分区变小（压缩）：

        select disk 0

        list partition （看一下分区的编号）。

        select partition 3 （选定你要缩小的分区，比如现在的 D 盘）。

        shrink desired=10240 （把这个分区减少 10GB，单位是 MB，留出空闲空间）。

    把某个分区变大（扩展）：

        注意：扩容的前提是，这个分区的紧邻右侧必须有一块没有分配的空闲空间。

        select partition 2 （选定你要放大的分区，比如 C 盘）。

        extend （把刚才压缩出来的、或者右侧所有的空闲空间都吞掉，加到这个分区里）。

3. 退出工具

操作完了输入 exit 就能退回到普通的命令行。
四、 避坑小提示

    盘符可能会变： 进到 PE 之后，原本的 C 盘可能会变成 D 盘或者 E 盘，操作之前先打开“此电脑”或者用 diskpart 的 list volume 确认一下你要弄的盘到底现在是哪个盘符。

    别拔 U 盘： 在 PE 运行期间，特别是正在装系统或者传输文件的时候，千万别拔掉 PE 启动盘，否则系统会直接死机或报错。
