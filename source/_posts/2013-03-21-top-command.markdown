---
layout: post
title: "top命令指南"
date: 2013-03-21 10:33
comments: true
categories: Linux Shell
---

####top 字段信息

- PID：进程的ID  
- USER：进程所有者
- PR：进程的优先级别，越小越优先被执行
- NInice：值
- VIRT：进程占用的虚拟内存
- RES：进程占用的物理内存
- SHR：进程使用的共享内存
- S：进程的状态。S表示休眠，R表示正在运行，Z表示僵死状态，N表示该进程优先值为负数
- %CPU：进程占用CPU的使用率
- %MEM：进程使用的物理内存和总内存的百分比
- TIME+：该进程启动后占用的总的CPU时间，即占用CPU使用时间的累加值。
- COMMAND：进程启动命令名称

#### top命令操作指令

【头部信息】

	1. l 切换uptime负载信息
	2. t 切换 task/cpu 信息
	3. m 切换内存信息

【显示】

	1. 按数字1 显示每个物理cpu 的负载G情况
	2. c 显示完整的命令行参数
	3. H(Shift + h)显示线程信息
	4. f 调整显示的字段 
	5. o 调整显示列的顺序
	6. S(Shift + s)累计模式，会把已完成或退出的子进程占用的CPU时间累计到父进程的MITE+

【高亮】     

	1. b 打开或关闭高亮
	2. x 打开或关闭排序列高亮
	3. y 打开或关闭行高亮

【控制】

	1.  h 显示帮助信息
	2.  q 退出top命令
	3.  <Space> 立即刷新
	4.  k 杀死某个进程 pid 
	5.  r 修改进程的renice 值
	6.  s 改变刷的时间间隔
	7.  W（Shift + w）保存对top的设置到文件~/.toprc,下次启动将自动调用toprc 文件的设置。


【窗口】

	1. A(Shift + a)切换单窗口与多窗口显示
	2. G(shift + g) 切换显示的 窗口（field group）,1 = Def 默认，2= Job,3=Mem,4 =usr 用户。 或在多窗口模式下，选择当前窗口。
	3. a,w 在多窗口模式时，循环列表， a = Forward ,w = Backward.
	4. - 减号 打开或关闭当前窗口

【排序】

	1. P(Shift + p) 按cpu占用排序
	2. M(Shitf + m) 按内存占用排序
	3. N(Shift + n)按进程id (pid) 进行排序
	4. T(Shitf + t)按进程生存时间排序MP<<<
	5. R(Shift + r) 倒序，改变排序方式
	6. <（Shift + ,） 或 >(Shift + .) 改变排序字段
	7. F (Shift + f) 或 O(Shift + o) 选择排序的字段

【过滤】     

	1. u 过滤用户
	2. g 过滤组
	3. i 不显示睡眠进程与僵尸进程，只显示正在运行的进程 


#####top命令行参数

top -hv | -bcisSHM -d delay -n iterations [-u user | -U user] -p pid [,pid ...]

-h 显示帮助

-b 批处理模式  
-c 显示程序名称或命令行参数  
-d delay 指定刷新时间  
-H 显示线程  
-i  Idle process toggle   
-M 按（k/M/G）的方式来显示内存信息 

-u user 指定用户  
-p pid 指定进程id  
