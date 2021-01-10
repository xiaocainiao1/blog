​                                                                    **mvn插件**

FAQ

1: 没有显式指定-P参数，profile webpack也会运行？

解决思路：当初以为是绑定了

```
<phase>generate-resources</phase>
```

mvn的时期，会主动运行，最后发现把这个注释掉还是会运行，经过仔细排查，发现是

```
<activation>
    <file>
        <missing>${basedir}/target/classes/static/app/main.bundle.js</missing>
    </file>
</activation>
```

在作妖，总结：还是对maven插件不是太了解

