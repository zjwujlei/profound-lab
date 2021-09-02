使用frida工具dump dex文件
=======================

### firida工具安装

这里直接使用<a href="https://github.com/hluwa/FRIDA-DEXDump">FRIDA-DEXDump</a>

>Requires
>
>frida: pip install frida
>[optional] click pip install click
>Installation
>
>From pypi
>
>pip3 install frida-dexdump
>frida-dexdump -h
>From source
>
>git clone https://github.com/hluwa/FRIDA-DEXDump
>cd FRIDA-DEXDump/frida-dexdump
>python3 main.py -h

pip install frida命令即为安装frida。

在使用pip命令是碰到一个python版本的问题，这里做一个记录：
```
dyld: Library not loaded: /System/Library/Frameworks/CoreFoundation.framework/Versions/A/CoreFoundation
  Referenced from: /Library/Frameworks/Python.framework/Versions/3.6/Resources/Python.app/Contents/MacOS/Python
  Reason: image not found
[1]    15494 abort      pip install frida
```

看日志了解到，pip命令使用了3.6版本的python，依赖的CoreFoundation库没有。在对应路径下确实没有找到该文件。

我们正常配置下应该是pip使用的是python2，pip3使用python3。当下使用pip命令使用了3.6版本python就十分怪异。使用“which pip”查看发现pip命令是在${PYTHON_HOME}/Versions/3.6/bin/pip下。其实本机上是安装了2.7版本也按照了2.7版本对应的pip工具。这里删除3.6下的pip工具解决问题，应该是之前使用的时候误装了。

