---
title: IDEA相关
date: 2019-09-08 15:40:22
tags: [IDEA,tool]
categories: tool
---

idea使用过程中的各种记录
<!-- more -->

### 配置篇
1. reopen last projects on startup 设置启动工程
  1. 启动idea后更改设置：Settings -> Appearance & Behaviour -> System Settings -> Reopen last project on startup
  2. 直接修改idea文件：  
      1. 打开文件：/.IntelliJIdeaxxxx/config/options/ide.general.xml
      2. 修改或者添加：
      ```
      <application>
        <component name="GeneralSettings">
          <option name="reopenLastProject" value="false" />
        </component>
      </application>
      ```
