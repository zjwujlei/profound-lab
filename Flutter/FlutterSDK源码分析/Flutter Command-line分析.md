Flutter Command-line分析
========================

### Flutter.sh/Flutter.bat

Flutter命令执行的就是Flutter.sh/Flutter.bat这个可执行文件，而这个可执行文件。我们在第一次执行Flutter命令时，终端会输出“Building flutter tool...”的日志。这个时候其实是基于"$FLUTTER_ROOT/packages/flutter_tools"中的dart代码在打snapshot包。因此需要注意的是，如果我们修改了相关实现，需要删除snapshot包后再次执行才能生效。

```
# FLUTTER_TOOL_ARGS isn't quoted below, because it is meant to be considered as
# separate space-separated args.
"$DART" --packages="$FLUTTER_TOOLS_DIR/.packages" $FLUTTER_TOOL_ARGS "$SNAPSHOT_PATH" "$@"
```
我们看到，在执行了Flutter命令后，最终是用DART.exe执行dart脚本，Flutter这个只是一个包装。我们甚至可以在flutter_tools中加入自定义的功能，用Flutter命令执行。


### flutter pub get




