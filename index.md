## iOS崩溃分析

### 1、App崩溃的基础知识

* DSYM文件

  DSYM 是保存16进制函数地址映射信息的中转文件
  我们的分析原理是crash应用-》crash日志-》crash地址

* Mach-O文件

  DYSM是Mach-O格式。
  第一部分是header，hander定义了文件的基本信息，包括文件大小，文件类型，使用的平台等信息。
  其次是load commands，这一部分定义了详细的加载指令，指明如何加载到内存。从头文件定义可以看到，基础的load_command结构体只包含了cmd以及cmdsize，通过cmd类型，可以转义成不同类型的load command 结构体。
  最后的数据部分，包括了代码段，数据段，符号表等具体的二进制数据。

![截屏2020-12-22 下午3 47 18](https://user-images.githubusercontent.com/16996959/127108153-4e4282a6-1578-4595-af11-1beffecfd45f.png)

* 可执行文件的加载过程

**内核加载可执行文件（Mach-O）-》获得dyld的路径-》dyld加载动态库、符号绑定、runtime的初始化-》main**

  VM Address是编译后Image的起始位置，Load Address是在运行时加载到虚拟内存的起始位置，Slide是加载到内存的偏移，这个偏移值是一个随机值，每次运行都不相同，有下面公式：
**Load Address = VM Address + Slide**

  由于dsym符号表是编译时生成的地址，crash堆栈的地址是运行时地址，这个时候需要经过转换才能正确的符号化。crash日志里的符号地址被称为Stack Address，而编译后的符号地址被称为Symbol Address，他们的关系如下：
**Stack Address = Symbol Address + Slide**
  运行时地址 = 编译后地址 + 偏移量

  符号化就是通过Symbol Address到dsym文件中寻找对应符号信息的过程。

* App捕获异常流程

  我们使用SDK中**NSSetUncaughtExceptionHandler**函数来捕获异常处理，但功能有限。像内存访问错误，重复释放等Signal错误无法处理。崩溃收集统计函数应该只进行一次调用，如果用第三方的话也最好只用一个第三方，这样我们获取崩溃统计信息的途径也是唯一的。
  第三方统计工具并不是用的越多越好，使用多个崩溃收集第三方会导致NSSetUncaughtExceptionHandler()函数指针的恶意覆盖，导致有些第三方不能收到崩溃信息。
现在很多第三方崩溃收集工具为了确保自己能最大可能的收集到崩溃信息，会对NSSetUncaughtExceptionHandler()函数指针的恶意覆盖。因为这个函数是将函数地址当做参数传递，所以只要重复调用就会被覆盖，这样就不能保证崩溃收集的稳定性。
  我们解析崩溃信息时，看到崩溃堆栈只有main.m文件中的崩溃，并且可以确定不是因为main.m文件中的bug导致的崩溃，就基本可以确定是NSSetUncaughtExceptionHandler()函数指针被恶意覆盖。

You can use the [editor on GitHub](https://github.com/wuxinsheng/iOS/edit/gh-pages/index.md) to maintain and preview the content for your website in Markdown files.

Whenever you commit to this repository, GitHub Pages will run [Jekyll](https://jekyllrb.com/) to rebuild the pages in your site, from the content in your Markdown files.

### Markdown

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/wuxinsheng/iOS/settings/pages). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://docs.github.com/categories/github-pages-basics/) or [contact support](https://support.github.com/contact) and we’ll help you sort it out.
