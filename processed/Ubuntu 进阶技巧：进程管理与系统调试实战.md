---
title: Ubuntu 进阶技巧：进程管理与系统调试实战
source: raw_20260712_211017_866066e2
source_url: https://www.zhihu.com/question/27764060/answer/1992497973939164469
created: '2026-07-12'
updated: '2026-07-12'
tags:
- Linux
- Ubuntu
- 命令行工具
- 系统调试
- 进程管理
- 后端运维
status: processed
note_type: content
related:
- 线程与进程的区别：火车与车厢的比喻
---

# Ubuntu 进阶技巧：进程管理与系统调试实战

## 核心结论
> 真正理解 Linux 的关键不在于花活，而在于掌握 [[procfs]]、[[strace]]、[[tmpfs]] 这类基础设施的工作方式。掌握 [[Linux任务控制]]、[[lsof]]、[[awk]] 等工具，能让排查问题的效率从只会看日志提升到直接观察系统行为的量级。

## 详细内容
## 核心结论
真正理解 Linux 的关键不在于花活，而在于掌握 [[procfs]]、[[strace]]、[[tmpfs]] 这类基础设施的工作方式。掌握 [[Linux任务控制]]、[[lsof]]、[[awk]] 等工具，能让排查问题的效率从只会看日志提升到直接观察系统行为的量级。

## 一、Linux 任务控制：不止 ctrl+c
多数人只熟悉 `ctrl+c` 终止进程和 `&` 放后台，但 Linux 的[[作业控制]]远不止于此：

- `ctrl+z`：将前台进程挂起（暂停而非终止）
- `fg`：将挂起任务恢复到前台
- `bg`：将挂起任务放到后台继续运行
- `jobs`：查看当前 shell 中的挂起与后台任务

需要注意，这些任务都绑定在当前 shell 上，终端关闭后任务也会死亡。解决方案：

- `nohup command &`：启动时就脱离终端
- `disown %1`：将已在跑的后台任务脱离终端

典型场景：SSH 到服务器跑长任务，中途想断开回家——`ctrl+z` 挂起、`bg` 扔后台、`disown` 脱离，第二天回来 `ps aux | grep xxx` 还能找到它。

## 二、/proc 与 /sys：内核的实时窗口
[[procfs]] 不是普通目录，而是内核暴露运行时状态的接口。常用排查点：

- `/proc/<pid>/fd/`：进程打开的所有[[文件描述符]]，排查读写哪些文件、连接哪些 socket
- `/proc/<pid>/maps`：进程[[内存映射]]，可看加载了哪些动态库
- `/proc/<pid>/environ`：进程环境变量，`cat /proc/<pid>/environ | tr '\0' '\n'` 即可查看
- `/proc/sys/`：内核参数，可直接读写，如临时调整 `fs/file-max`

## 三、strace：观察进程的真实行为
当日志看不出问题时，`strace` 能直接展示进程的所有[[系统调用]]。

```bash
strace -p <pid>                     # 附加到运行中进程
strace -e open,read,write -p <pid>  # 仅跟踪特定调用类型
strace -e network -p <pid>          # 跟踪网络相关
strace -c -p <pid>                  # 统计各类系统调用耗时
```

实战案例：服务卡在 starting...，`strace` 一挂发现 `connect()` 在等一个连不上的 Redis，定位到配置写错的地址。`strace -c` 还能直观判断进程时间花在 CPU 计算还是 IO 等待。

## 四、/dev/shm：基于内存的 tmpfs
[[tmpfs]] 是挂载在内存上的文件系统，读写速度比 SSD 快一个数量级，重启后数据消失，适合临时数据处理。

- 默认大小为内存的一半，可用 `mount -o remount,size=4G /dev/shm` 调整
- 注意：占的是真实内存，不要把大文件全部塞进去撑爆内存
- `/tmp` 在多数发行版也是 tmpfs，可用 `df -h /tmp` 确认

典型用法：处理 10GB 日志前 `cp` 到 `/dev/shm`，再用 `awk` 处理，比直接在机械盘上操作快一个数量级。

## 五、lsof：定位谁在占用
删除文件后磁盘空间没释放？端口被占不知道是谁？都用 [[lsof]]：

```bash
lsof | grep deleted      # 找出仍占用已删除文件的进程
lsof -i :8080            # 查找占用 8080 端口的进程
lsof -p <pid>            # 查看某进程打开的文件与网络连接
```

## 六、awk：按行处理、按列提取
awk 的精髓是模式匹配加列提取，记住这个 90% 场景够用：

```bash
awk '{print $1}' file.txt                  # 打印第一列
awk -F',' '{print $2}' file.csv            # 按逗号分隔取第二列
awk '$3 > 100 {print $0}' file.txt         # 条件筛选
awk '{sum += $1} END {print sum}' file.txt # 求和
```

经典日志统计一行搞定：

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -20
```

## 总结
这些不是奇技淫巧，是 Linux 真正的基础设施。会用 `strace` 的人与只会看日志的人，排查效率差几个档次。

## 相关笔记
- [[线程与进程的区别：火车与车厢的比喻]]