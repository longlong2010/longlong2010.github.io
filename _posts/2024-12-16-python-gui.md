---
title: 使用Python构建图形界面
layout: post
---

> 自从习惯了Web程序后，自己的PC图形界面程序开发基本还停留在Java上，使用Netbeans的图形界面构建工具搭配Swing实现图形界面程序的构建，并且一直感觉Java的图形界面编程的逻辑是非常清晰的——各种操作行为对应相应的Action类或是Listener接口。
>
> 之后也尝试过Qt或是其他的图形界面库，感觉都不是特别好用。如今已经好多年不用Java，而且Java目前的生态环境相比现在Python的流行也要差很多了，估计主要还是用在Android开发和比较老的电商系统上了吧。
> 
> 目前的需求是借助Python的Numpy和Matplotlib实现一些类型Matlab的功能，并且要封装成为一个可执行文件。

#### 1. Tk图形界面库

> Tkinter 模块(Tk 接口)是Python的标准Tk GUI工具包的接口 .Tkinter可以在大多数的Unix平台下使用,同样可以应用在Windows 和macOS系统里。Tk8.0的后续版本可以实现本地窗口风格,并良好地运行在绝大多数平台中。
>
> 其最大的优点就是作为了Python的常用标准库，直接就可以使用，而且使用上也相对简单，对于开发较为简单的图形界面程序比较友好，使用难度也相对较低。其运行的核心代码如下
>
```python
import tkinter;
>
if __name__ == '__main__':
    tk = tkinter.Tk(className='Main');
    tk.mainloop();
```

#### 2. Tk控件

> Tkinter的提供各种控件，如按钮，标签和文本框等共计15种。在使用时需要关联到控件的上层控件，实现整个控件的树形结构。如下面的代码实现了一个完整的菜单控件。控件使用add\_command方法实现绑定操作时执行的代码。
```python
menu = tkinter.Menu(tk);
m1 = tkinter.Menu(menu, tearoff=False);
m1.add_command(label='打开', command=load_data);
m1.add_command(label='保存', command=save_data);
m1.add_command(label='退出', command=on_exit);
menu.add_cascade(label='文件', menu=m1);
tk.config(menu=menu);
```
> 除了基本控件，Tkinter还提供了一些常用的组合控件，如各种类型的提示框，实现消息提示，文件选择等功能。
```python
#提示信息
tkinter.messagebox.showerror('警告', '数据格式错误！');
#选择要打开的文件
path = tkinter.filedialog.askopenfilename();
```

#### 3. 界面布局
> Tkinter控件有特定的几何状态管理方法，管理整个控件区域组织，以下是Tkinter公开的几何管理类：包、网格、位置。比较类似Java中的Layout。使用包(pack)可是实现控件大方位的指定（上下左右）。
>
```python
#放置在顶部
canvas.get_tk_widget().pack(side=tkinter.TOP);
#放置在底部
s2.pack(side=tkinter.BOTTOM);
```

#### 4. 动态调整控件属性
> 在图形界面程序中经常需要进行一定操作后，根据执行的结果动态调整某些控件的某些属性，Tk中则是通过控件的config方法，该方法接受一个Python字典结构，字典中的键值为需要调整的属性，值则为调整后的属性值。如以下代码将两个范围控件的最大值进行更新。
```python
fm = data.maxFreq() + 2;
s1.config({'to' : fm});
s2.config({'to' : fm});
```

#### 5. 在Tk中集成Matplotlib图表
> 由于需要使用Matplotlib绘制图表，并继承在图形界面中，Tk也支持将Matplotlib图表放置在界面中，可通过以下方法实现
```python
#创建一个Matplotlib绘图
figure = matplotlib.figure.Figure();
#利用绘图创建一个画布
canvas = matplotlib.backends.backend_tkagg.FigureCanvasTkAgg(figure, master=tk);
#将画布放置到界面中
canvas.get_tk_widget().pack(side=tkinter.TOP);
#在完成绘图后调用更新显示
canvas.draw();
```

#### 6. 打包程序

> 如今Python可以借助pyinstaller实现对源代码进行打包，形成一个可执行文件，如此就可以实现程序的分发，而不需要运行的计算机安装配置对应的Python环境，同时也能一定程度上隐藏源代码。打包的命令格式如下
```bash
pyinstaller -F -w -i <程序图标文件> [源代码文件]
```
> 其中-F表示生成一个exe文件，-w表示运行时不弹出命令提示符窗口，-i可以指定可执行文件的图标。
