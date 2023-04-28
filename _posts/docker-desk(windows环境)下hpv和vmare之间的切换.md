​                                            docker-desk(windows环境)下hpv和vmare之间的切换

```
bcdedit /set hypervisorlaunchtype auto  -- 启动hpv
bcdedit /set hypervisorlaunchtype off  -- 关闭hpv
```

然后重启电脑