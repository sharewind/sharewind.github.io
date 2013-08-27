---
layout: post
title: "time in python"
date: 2013-03-28 10:41
comments: true
categories: Python
---

###datetime 模块

datetime.MINYEAR 1  date或datetime中所允许的年份  
datetime.MAXYEAR 9999 date或datetime中所允许的年份  

####类型  

- datetime.**date** 表示日期的类，属性：year,month,day.  
- datetime.**time** 表示时间的类,不依赖于具体的日期，假定每天有 24 * 60 * 60 秒，属性：hour,minute,second,miscrosecond, and tzinfo.  
- datetime.**datetime** 日期与时间的组合，属性：year,month,day，hour,minute,second,miscrosecond, and tzinfo.    
- datetime.**timedelta** 表示两个date,time or datetime之间的差。  
- datetime.**tzinfo** 抽象的时区信息类。在datetime 和 time 类中被用来提供一个可定制时区的概念。  

以上类型都是不可变的。


类的继承关系

	object  
	    timedelta  
	    tzinfo  
	    time  
	    date  
	        datetime  


**timedelta**  

- datetime.timedelta(days,seconds,microseconds,minutes,hours,weeks)  
- timedelta.total_seconds()  

year = datetime.timedelta(days=365)  
ten_year = year * 10

一天有多少秒  
datetime.timedelta(days=1).total_seconds()

明天  
datetime.date.today() + datetime.timedelta(days=1)


**date**  

类方法  

- datetime.date(year, month, day)  
- date.today()   This is equivalent to date.fromtimestamp(time.time()).  
- date.fromtimestamp(timestamp)  返回time.time() 构造timestamp 对应的日期。  
- date.fromordinal(ordinal) 0001-01-01年算的序数为1.

实例方法  

- date.replace(year, month, day)  
- date.isocalendar() 返回格式为（2013，03，28）这样的元组  
- date.isoformat()  ‘YYYY-MM-DD’. 示例, date(2002, 12, 4).isoformat() == '2002-12-04'.    
- date.ctime()   for example date(2002, 12, 4).ctime() == 'Wed Dec 4 00:00:00 2002'. 
- date.strftime(format) 返回指定格式的日期  
- date.weekday() 返回星期几，0是星期一。

支持的运算  
date2 = date1 + timedelta  
date2 = date1 - timedelate  
timedelta = date1 - date2  
date1 < date2  


**datetime**

类方法(返回datetime对象)  

- datetime.datetime(year, month, day[, hour[, minute[, second[, microsecond[, tzinfo]]]]])    
- datetime.today()  等同于 datetime.fromtimestamp(time.time())，tzinfo 为None.
- datetime.now([tz])     
- datetime.fromtimestamp(timestamp[, tz])  
- datetime.combine(date, time)  
- datetime.strptime(date_string, format)  将字符串转换为时间。

实例方法  

- datetime.date()  
- datetime.time()  
- datetime.replace([year[, month[, day[, hour[, minute[, second[, microsecond[, tzinfo]]]]]]]])  
- datetime.timetuple()  类似time.localtime(),等同于 time.struct_time((d.year, d.month, d.day, d.hour, d.minute, d.second, d.weekday(), yday, dst))  
- datetime.strftime(format)  

支持的运算  
datetime2 = datetime1 + timedelta  
datetime2 = datetime1 - timedelate  
timedelta = datetime1 - datetime2  
datetime1 < datetime2  


**time**

datetime.time([hour[, minute[, second[, microsecond[, tzinfo]]]]])  

实例方法  

- time.replace([hour[, minute[, second[, microsecond[, tzinfo]]]]])  
- time.isoformat() ISO 8601 format, HH:MM:SS.mmmmmm
- time.strftime(format)  


###time 模块

time 模块在Uninx平台上的时间纪元从1970年1月1日开始，截止点是2038年。该模块不能处理1970年前或是2038年后的时间。

**类**

time.struct_time
gmtime(), localtime(), and strptime() 时间序列的返回类型. 是一个命名元组接口: 值可以用索引或属性名进行访问.属性如下：

	Index	Attribute	Values
	0	tm_year	(for example, 1993)
	1	tm_mon	range [1, 12]
	2	tm_mday	range [1, 31]
	3	tm_hour	range [0, 23]
	4	tm_min	range [0, 59]
	5	tm_sec	range [0, 61]; see (2) in strftime() description
	6	tm_wday	range [0, 6], Monday is 0
	7	tm_yday	range [1, 366]
	8	tm_isdst	0, 1 or -1; see below

**函数**   

- time.time() 返回从纪元开始的秒数，是一个浮点数。  
- time.clock()在Unix上返回当前处理器的时间。

- time.gmtime([secs])  把秒数转化为GMT时间，返回time.struct_time。  
  gmtime(0) 返回unix时间，1970年1月1日。

- time.localtime([secs]) 将秒数转化为本地时间，返回 time.struct_time。   (tm_year=2013, tm_mon=3, tm_mday=28, tm_hour=14, tm_min=48, tm_sec=26, tm_wday=3, tm_yday=87, tm_isdst=0) 

- time.mktime(t) localtime()的逆过程，参数为 struct_time or full 9-tuple。返回时间截，一个类似time.time() 一样的浮点数。

- time.asctime([t]) 把  gmtime() 或  localtime() 返回的tunple 或struct_time 转化为本地时间字符串。如果未指定t,则使用 localtime() 。如：'Sun Jun 20 23:21:05 1993'。
- time.ctime([secs]) 把秒数表示为当前本地时间字符串。如果未指定参数，则使用time().'Thu Mar 28 14:43:37 2013'   

- time.sleep(secs)  暂停当前线程指定的秒数。参数可以是浮点数。



- time.strftime(format[, t]) 格式化时间为字符串, 把 gmtime() or localtime() 返回的tunple 或 struct_time  进行格式化。
- time.strptime(string[, format]) 将字符串转化为时间，返回struct_time

当前时间的两种方式   
time.strftime("%Y-%m-%d %H:%M:%S")  
datetime.now().strftime("%Y-%m-%d %H:%M:%S")


### calendar 模块

这个模块提供了一些有用的日历功能。

类方法  
calendar.Calendar([firstweekday]) firstweekday指定一周的第一天，默认0以周一作为一周的第一天，6把星期天作为一周的第一天。

实例方法  

- iterweekdays() 遍历week的每一天,从0-6
- itermonthdates(year, month)遍历month的每一天，返回2013-02-01的格式。并且以周为单位，会算上月份开始的那周一开始到月份结束的那周末。
- itermonthdays2(year, month) 遍历month的每一天，返回（几日，周几）的元组。
- itermonthdays(year, month) 遍历month的每一天，只返回几日。
- monthdatescalendar(year,month) 返回这个月的 datetime.date 的对象列表。

TextCalendar 生成文本日历
HTMLCalendar 生成HTML日历

静态方法

- calendar.isleap(year) 是否润年  
- calendar.leapdays(y1, y2) 从y1到y2（不包括y2）有几个润年  
- calendar.weekday(year, month, day) 返回周几，0是星期一。  
- calendar.monthrange(year, month) 返回当月第一天是周几，当月有多少天。  

### timeit 模块

timeit 是一个可以衡量代码块执行时间的工具

简单示例  

{% codeblock lang:python 监测程序执行的时间 %}

python -m timeit '"-".join(str(n) for n in range(100))'  
timeit.timeit('"-".join(str(n) for n in range(100))', number=10000)

{% endcodeblock %}

复杂示例 

{% codeblock lang:python  %}

	def test():
	    """Stupid test function"""
	    L = []
	    for i in range(100):
	        L.append(i)

	if __name__ == '__main__':
	    import timeit
		# 这里从当前的__main__模块中import了test方法
	    print(timeit.timeit("test()", setup="from __main__ import test"))

{% endcodeblock %}

### 计时

	def timer(func, reps, *pargs, **kargs):
		start = time.clock()
		for i in xrange(reps):
			ret = func(*pargs,**kargs)
		elapsed = time.clock() - start
		reutrn (elasped,ret)

对于windows系统 time.clock() 调用计时代码是最好的。time.time 在某些Unix平台上可能提供更好的解析。  
更加内容参考：《Python学习手册》


####参考资料

- [8.1. datetime — Basic date and time types](http://docs.python.org/2/library/datetime.html#module-datetime)
- [15.3. time — Time access and conversions](http://docs.python.org/2/library/time.html#time.time)
- [8.2. calendar — General calendar-related functions](http://docs.python.org/2/library/calendar.html#module-calendar)
- [timeit — Measure execution time of small code snippets](http://docs.python.org/2/library/timeit.html)