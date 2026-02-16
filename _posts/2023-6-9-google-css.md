---
title: 谷歌 CSS、HTML 书写规范简要记录
date: 2023-6-9 16:39:00 +0800 # 2022-01-01 13:14:15 +0800 只写日期也行；不写秒也行；这样也行 2022-03-09T00:55:42+08:00
categories: [CSS]
tags: [CSS,HTML]     # TAG names should always be lowercase

# 以下默认false
math: true
mermaid: true
pin: false
---

#### 1.常规

##### 1.1 基础书写风格

###### 1.1.1 网络协议
在使用各种图片，媒体资源，script 时，尽量采用 `https`网络协议，除了不支持`https`的资源。
##### 1.2 基础格式规范
###### 1.2.1 缩进
一次缩进两个空格，**不推荐使用tab键缩进**

```html
<ul>
  <li></li>
  <li></li>
</ul>
```

###### 1.2.2 大小写
只使用小写，所有的代码，包括标签名称，属性，属性值，css选择器等，标签内容（字符串除外)

###### 1.2.3 尾部空白 `trailing whitespace`
去除尾部空白，尾部空白是非必须的，同时可能导致错误。

##### 1.3 基础源规则（Meta)
###### 1.3.1 编码规则

尽量使用UTF-8编码，保证编辑器使用的是UTF-8编码器，没有字节标记顺序。

指定HTML模板的编码，不要指定样式表的编码（样式表默认使用UTF-8)

###### 1.3.2 Comments 注解

尽可能解释编码，使用注释去解释代码，

**What does it cover, what purpose does it serve, why is respective solution used or preferred?**

+ 它包含什么
+ 他是用来干什么的
+ 为什么需要使用这种写法

###### 1.3.3 Action Items （任务项）

使用 `TODO` 标注任务项和需要完成的事情。
使用括号形式加入添加人 `TODO(frank)`
使用冒号和内容，在TODO后面添加需要完成的任务项`TODO:action item`

#### 2 HTML
##### 2.1样式规则

+ 尽量使用 HTML5，尽管对于HTML标签来说是可以，但是尽量不要闭合空的标签。比如说`<br>` 而非 `<br/>`
+ 使用HTML尽量符合它的语意 <u>Using HTML according to its purpose is important for accessibility, reuse, and code efficiency reasons.</u>
+ 多媒体反馈，为多媒体提供可替代的内容 比如说添加 `alt`属性
+ 结构 和 表现 和 行为 分离 *Strictly keep structure (markup), presentation (styling), and behavior (scripting) apart, and try to keep the interaction between the three to an absolute minimum.*。严格的分开 css html 和 js ，同时尽量保证三者交互的最小化。
+ 不要使用实体引用 *Do not use entity references.*
  + 如果团队使用的是同一种编码格式，就不需要为一些特殊符号添加引用
+ 省略可选标签
+ 对于`type` 标签的使用
  + 针对 样式和 脚本文件，尽量不要使用 type值， 比如说省略 `type='text/css'`
  + 如果针对不是使用的JS脚本文件，需要添加type

##### 2.2 格式规则
+ 为每一段，每一个新的表格，新起一行，对每个子类元素进行缩进
+ 虽然HTML没有行数限制，但是最好打断太长的行。新行至少要有四个缩进（属性区域）

```html
<md-progress-circular md-mode="indeterminate" class="md-accent"
    ng-show="ctrl.loading" md-diameter="35">
</md-progress-circular>
<md-progress-circular
    md-mode="indeterminate"
    class="md-accent"
    ng-show="ctrl.loading"
    md-diameter="35">
</md-progress-circular>
<md-progress-circular md-mode="indeterminate"
                      class="md-accent"
                      ng-show="ctrl.loading"
                      md-diameter="35">
</md-progress-circular>
```

+ 在HTML内进行标签属性赋值时，尽量使用双引号。

#### 3.CSS

##### 3.1 CSS样式规范

+ 尽量使用有效，验证过的CSS样式。
+ 针对`ID`和 `Class` 的命名
+ 尽量使用有意义 或者通用的 命名
+ 尽量使用反映当前ID 或者 Class命名 目的的命名
+ *as short as possible but as long as necessary*

+ 避免在 元素选择器后 使用 Id 和 class选择器；除非有必要，不要混用元素选择器和 ID选择

+ 尽量使用简写形式（shorthand）

  ```css
    /* Not recommended */
    border-top-style: none;
    font-family: palatino, georgia, serif;
    font-size: 100%;
    line-height: 1.6;
    padding-bottom: 2em;
    padding-left: 1em;
    padding-right: 1em;
    padding-top: 0;
    /* Recommended */

    border-top:0;
    font:100%/1.6 palatino,georgia,serif;
    padding:0 1rem 2rem;
```

+ **0** 和 单位 `Units`。 在0的后面尽量省略单位
+ 使用小数点时，尽量省略前面的0 `font-size:.8rem`
+ 在表示颜色的时候，尽量使用三位16进制表示
+ 在为标签命名时，*使用连字符号而不是下划线*

##### 3.2 CSS格式规则

+ 声明顺序 **按照字母表的顺序来进行声明**，以便记忆和维护
+ 缩进所有的块内容，提高层次性、更容易理解
+ 每一个声明结束，都需要加上分好，保证连续性。
+ 属性`property` 和 `value`，之间，需要加入一个空格

  ``` js
    /* Not recommended */
    h3 {
      font-weight:bold;
    }
    /* Recommended */
    h3 {
      font-weight: bold;
    }
```

+ 选择器 和前面的括号之间，使用空格分开。
```css
    /* Not recommended: missing space */
    #video{
      margin-top: 1em;
    }

    /* Not recommended: unnecessary line break */
    #video
    {
      margin-top: 1em;
    }
    /* Recommended */
    #video {
      margin-top: 1em;
    }

```
+ 每一个选择器，每一个属性，都需要新开一行。
 ```css
    /* Not recommended */
    a:focus, a:active {
      position: relative; top: 1px;
    }
    /* Recommended */
    h1,
    h2,
    h3 {
      font-weight: normal;
      line-height: 1.2;
    }
```

+ CSS内，使用单引号，`url()`内，不需要加入引号。
+ 对样式进行分组管理，最好一组的CSS样式写到一块。

#### 4.CSS 属性编写顺序

根据属性重要性，从外往内书写，【同时，每个分组内部又要满足字母表顺序】

- Layout Properties (`position`, `float`, `clear`, `display`)
- Box Model Properties (`width`, `height`, `margin`, `padding`)
- Visual Properties (`color`, `background`, `border`,` box-shadow`)
- Typography Properties (`font-size`, `font-family`, `text-align`, `text-transform`)
- Misc Properties (`cursor`, `overflow`, `z-index`)
