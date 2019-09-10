---
title: maven记录
date: 2019-09-10 17:50:24
tags: [tool,maven]
categories: algorithm
---

maven使用过程中的各种记录
<!-- more -->

### 打包时排除文件
在`maven-jar-plugin`下`configuration`增加`excludes`，如下：
```
<project>
  ...
  <build>
    <plugins>
      ...
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <version>3.1.2</version>
        <configuration>
          <excludes>
            <exclude>**/service/**</exclude>
            <exclude>**/test/*</exclude>
          </excludes>
        </configuration>
      </plugin>
      ...
    </plugins>
  </build>
  ...
</project>
```
注意：`*`代表排除文件，`**`代表排除文件及子目录
