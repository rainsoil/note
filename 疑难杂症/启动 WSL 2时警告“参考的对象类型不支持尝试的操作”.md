# 启动 WSL 2时警告“参考的对象类型不支持尝试的操作”

出现图中所示错误的原因是 代理软件与 wsl2 的端口冲突。

在管理员身份模式下执行 netsh winsock reset ,可以重新启动 WSL。

此操作会导致代理软件（proxifier）无法使用，请谨慎操作。
Github Issue1
Github Issue2

使用 NoLsp.exe 下载链接

备用下载链接

使用管理员身份运行以下命令:

NoLsp.exe C:\Windows\system32\wsl.exe
1
参数为 wsl 的绝对路径（默认为 C:\Windows\system32\wsl.exe）

问题原因及解决方案的讨论见 Gihub Issue
————————————————
版权声明：本文为CSDN博主「yingming006の」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/MShow006/article/details/103774672