# GitHubPit1_lineFeed

https://github.com/cssmagic/blog/issues/22

GitHub 第一坑：换行符自动转换

源起

一直想在 GitHub 上发布项目、参与项目，但 Git 这货比较难学啊。买了一本《Git 权威指南》，翻了几页，妈呀，那叫一个复杂，又是 Cygwin 又是命令行的，吓得我不敢学了。

终于某天发现 GitHub 还有一个 Windows 客户端，试了一下还挺好用。不需要掌握太多的 Git 原理和命令，也可以在 GitHub 上麻溜建项目了，甚是欢喜。可是好景不长，第一次参与开源项目就出洋相了。

经过

小心翼翼地 Fork 了朴灵大大 (@JacksonTian) 的 EventProxy 项目，本地改好提交，同步到服务器，怀着激动的心情发出 Pull Request……这时发现问题了。我发现 diff 图表显示的更新并不仅是我修改的那几行，而是整个文件都显示为已修改。（下图为示意图）

first-trap-on-github-autocrlf-diff

这看起来很奇怪啊，于是赶紧撤回 Pull Request，自己闷头找原因。

初步定位是文件的换行符问题，因为我发现本地的文件是 Windows 换行符，但很显然大家现在做项目都是用 UNIX 换行符啊。这是一大疑点，于是在反复对比 Web 端和本地的各个文件、各个版本之后，基本定位到了问题所在。

背景

在各操作系统下，文本文件所使用的换行符是不一样的。UNIX/Linux 使用的是 0x0A（LF），早期的 Mac OS 使用的是 0x0D（CR），后来的 OS X 在更换内核后与 UNIX 保持一致了。但 DOS/Windows 一直使用 0x0D0A（CRLF）作为换行符。（不知道 Bill Gates 是怎么想的，双向兼容？）

这种不统一确实对跨平台的文件交换带来麻烦。虽然靠谱的文本编辑器和 IDE 都支持这几种换行符，但文件在保存时总要有一个固定的标准啊，比如跨平台协作的项目源码，到底保存为哪种风格的换行符呢？

Git 作为一个源码版本控制系统，以一种（我看起来）有点越俎代庖、自作聪明的态度，对这个问题提供了一个“解决方案”。

Git 由大名鼎鼎的 Linus 开发，最初只可运行于 *nix 系统，因此推荐只将 UNIX 风格的换行符保存入库。但它也考虑到了跨平台协作的场景，并且提供了一个“换行符自动转换”功能。

安装好 GitHub 的 Windows 客户端之后，这个功能默认处于“自动模式”。当你在签出文件时，Git 试图将 UNIX 换行符（LF）替换为 Windows 的换行符（CRLF）；当你在提交文件时，它又试图将 CRLF 替换为 LF。

（看明白了吗？一个版本控制系统会在你不知不觉的情况下修改你的文件。这 TM 简直酷毙了，对吧？）

缺陷

Git 的“换行符自动转换”功能听起来似乎很智能、很贴心，因为它试图一方面保持仓库内文件的一致性（UNIX 风格），一方面又保证本地文件的兼容性（Windows 风格）。但遗憾的是，这个功能是有 bug 的，而且在短期内都不太可能会修正。

问题具体表现在，如果你手头的这个文件是一个 包含中文字符的 UTF-8 文件，那么这个“换行符自动转换”功能 在提交时是不工作的（但签出时的转换处理没有问题）。我猜测可能这个功能模块在处理中文字符 + CRLF 这对组合时直接崩溃返回了。

这可能还不是唯一的触发场景（毕竟我没有太多精力陪它玩），但光这一个坑就已经足够了。

踩坑

这是一个相当大的坑，Windows 下的中文开发者几乎都会中招。举个例子，你在 Windows 下用默认状态的 Git 签出一个文件，写了一行中文注释（或者这个文件本来就包含中文），然后存盘提交……不经意间，你的文件就被毁掉了。

因为你提交到仓库的文件已经完全变成了 Windows 风格（签出时把 UNIX 风格转成了 Windows 风格但提交时并没有转换），每一行都有修改（参见本文开头的示意图），而这个修改又不可见（大多数 diff 工具很难清楚地显示出换行符），这最终导致谁也看不出你这次提交到底修改了什么。

这还没完。如果其他小伙伴发现了这个问题、又好心地把换行符改了回来，然后你又再次重演上面的悲剧，那么这个文件的编辑历史基本上就成为一个谜团了。

由于老外几乎不可能踩到这个坑，使得这个 bug 一直隐秘地存在着。但在网上随便搜一下，就会发现受害者绝对不止我一个，比如 这位大哥的遭遇 就要比我惨痛得多。

防范

首先，不要着急去整 Git，先整好自己。你的团队需要确立一个统一的换行符标准（推荐使用 UNIX 风格）。然后，团队的成员们需要分头做好准备工作——配置好自己的代码编辑器和 IDE，达到这两项要求：

在新建文件时默认使用团队统一的换行符标准
在打开文件时保持现有换行符格式不变（即不做自动转换）
这样一方面可以最大程度保证项目代码的规范一致，另一方面，即使现有代码中遗留了一些不规范的情况，也不会因为反复转换而导致混乱。（当然，作为一个强迫症患者，我还是祝愿所有的项目从一开始就步入严谨有序的轨道。）

接下来，我们就可以开始调教 Git 了。我的建议是， 完全关掉这个自作聪明的“换行符自动转换”功能。关闭之后，Git 就不会对你的换行符做任何手脚了，你可以完全自主地、可预期地控制自己的换行符风格。

下面主要针对不同的 Git 客户端，分别介绍一下操作方法。

Git for Windows

这货由 Git 官方出品，在安装时就会向你兜售“换行符自动转换”功能，估计大多数人在看完华丽丽的功能介绍之后会毫不犹豫地选择第一项（自动转换）。请千万抵挡住诱惑，选择最后一项（不做任何手脚）。

first-trap-on-github-autocrlf-git-install

如果你已经做出了错误的选择，也不需要重新安装，可以直接使用命令行来修改设置。很简单，直接打开这货自带的命令行工具 Git Bash，输入以下命令，再敲回车即可：

git config --global core.autocrlf false
first-trap-on-github-autocrlf-bash

TortoiseGit

很多从 TortoiseSVN 走过来的同学很可能会选用 TortoiseGit 作为主力客户端，那么也需要配置一下。在 Windows 资源管理器窗口中点击右键，选择“TortoiseGit → Settings → Git”，做如下设置。

first-trap-on-github-autocrlf-tortoisegit

（由于 TortoiseGit 实际上是基于 Git for Windows 的一个 GUI 外壳，你在上一节所做的设置会影响到上图这些选项的状态，它们可能直接就是你所需要的样子了。）

GitHub 的 Windows 客户端

它是今天的第二被告。这货很容易上手，很适合小白，我主要用它来一键克隆项目到本地。可能正是为了维护简洁易用的亲切形象，这货并没有像 TortoiseGit 那样提供丰富的选项（对“换行符自动转换”这样的细节功能完全讳莫如深啊，我这样的小白死了都不知道怎么死的……）。因此，我们需要手动修改一下它的配置。

GitHub 的 Windows 客户端实际上也是一个壳，它自带了一个便携版的 Git for Windows。这个便携版和你自己安装的 Git for Windows 是相互独立的，不过它们都会使用同一个配置文件（实际上就是当前用户主目录下的 .gitconfig 文件）。

所以如果你已经配置好了自己安装的 Git for Windows，那就不用操心什么了。但如果你的机器上只装过 GitHub 的 Windows 客户端，那么最简单的配置方法就是手工修改配置文件了。

修改 Git 的全局配置文件

进入当前用户的主目录（通常 XP 的用户目录是 C:\Documents and Settings\yourname，在 Vista 和 Win7 下是 C:\Users\yourname），用你最顺手的文本编辑器打开 .gitconfig 文件。

在 [core] 区段找到 autocrlf，将它的值改为 false。如果没找到，就在 [core] 区段中新增一行：（最终效果见图）

    autocrlf = false
first-trap-on-github-autocrlf-gitconfig

事实上上面介绍的所有命令行或图形界面的配置方法，最终效果都是一样的，因为本质上都是在修改这个配置文件。

还有

关掉了 Git 的“换行符自动转换”功能就万事大吉了吗？失去了它的“保护”，你心里会有点不踏实。你可能会问：如果我不小心在文件中混入了几个 Windows 回车该怎么办？这种意外可以防范吗？

事实上 Git 还真能帮你阻止这种失误。它提供了一个换行符检查功能（core.safecrlf），可以在提交时检查文件是否混用了不同风格的换行符。这个功能的选项如下：

false - 不做任何检查
warn - 在提交时检查并警告
true - 在提交时检查，如果发现混用则拒绝提交
我建议使用最严格的 true 选项。

和 core.autocrlf 一样，你可以通过命令行、图形界面、配置文件三种方法来修改这个选项。具体操作就不赘述了，大家自己举一反三吧。

最后

你可能还会问，如果我自己一不小心用编辑器把整个文件的换行符都转换成了另一种格式怎么办？还能预防吗？

这……我就真帮不了你了。所以还是建议大家在提交文件之前多留心文件状态：

first-trap-on-github-autocrlf-commit

如果发现变更行数过多，而且增减行数相同，就要警惕是不是出了意外状况。被图形界面惯坏的孩子往往缺乏耐心，对系统信息视而不见，看到按钮就点，容易让小疏忽酿成大事故。所以高手们青睐命令行，并不是没有道理的。

好了，小伙伴们，今天的《踩坑历险记》就到这儿，我们下集再见！祝大家编码愉快！

© Creative Commons BY-NC-ND 4.0   |   我要订阅   |   我要打赏
 👍 7
@perryyeh
perryyeh commented on Aug 18, 2013
试了下 找到之前没这个提示的原因了。 之前的git客户端是默认配置，没autocrlf = false 和safecrlf = true 的选项，所以提交的时候，git是自己的转换来提交的，倒也一直没处问题。 刚设置了下这2个选项，果然一下子就提示整个文件被替换了。
@jsw0528
jsw0528 commented on Aug 18, 2013
结论：不要用 windows 😄
@liuweifeng
liuweifeng commented on Aug 18, 2013
结论：不要用 windows 😄 哈哈~~
@JacksonTian
JacksonTian commented on Aug 18, 2013
结论：不要用 windows 😄 +1
@cssmagic
Owner
cssmagic commented on Aug 18, 2013
感谢朴灵 (@JacksonTian) 和米粽粽 (@myst729) 两位老师对本文的技术校审。
@myst729
myst729 commented on Aug 18, 2013
弄死楼上几个果粉 [挖鼻屎] Windows 用官方的命令行客户端 msysgit 毫无压力 XD
@SunLn
SunLn commented on Aug 18, 2013
同事用 window写代码、linux虚拟机提交代码曾经碰到过这种问题，看来这篇文章可以解决。
@cssmagic
Owner
cssmagic commented on Aug 19, 2013
转发转载记录：

[微博] 伯乐在线官方微博： //weibo.com/1670481425/A5uRqg6an
伯乐在线： //blog.jobbole.com/46200/
@cssmagic
Owner
cssmagic commented on Aug 19, 2013
转自《Git 权威指南》作者蒋鑫 (@jiangxin) 老师的微博： //weibo.com/2080831604/A5v5Rhehj

靠程序员设置 core.autocrlf 就是在以德治国，不靠谱。对于有跨平台需要的库，要提交一个 .gitattributes 文件，详见《Git权威指南》第40章40.3小节。第40章40.1小节字符集部分不适用于1.7.10版本后的msysGit，不要再设置 i18n.commitEncoding和i18n.logOutputEncoding了。
我回头翻一下书，补补课。
@lexdene
lexdene commented on Aug 20, 2013
Windows ?
23333333
@devan5
devan5 commented on Aug 21, 2013
你好, 我尝试了下, 并没有发现此问题.

我试着重现这个故事原委, 可是没有发现 autocrlf 自动转换失败

具体步骤是 :

git init 
git config core.autocrlf true
touch Test.js
notepad++ Test.js // 设定为 UTF8 编码, 档案格式为 window(CRLF), 并写下几行代码和注释, 包含中文, 按照楼主所说
git add .
git ci -m "init"

git config core.autocrlf false
rm Test.js
git co .
notepad++ Test.js // 发现文档格式已经是 LF, 说明自动转换成功
若有纰漏, 请批判纠正补充.

另外, 附注下我的环境:
git 客户端版本: 1.8.3
window7 64
@cssmagic
Owner
cssmagic commented on Aug 21, 2013
@devan5 谢谢回复。有空我再重现一下事故现场。
@yisibl
yisibl commented on Mar 3, 2014
弄死楼上几个果粉 [挖鼻屎] Windows 用官方的命令行客户端 msysgit 毫无压力 XD
 @jimichan jimichan referenced this issue in NLPchina/ansj_seg on Aug 29, 2014
 Closed
git autocrlf 配置问题 #146
@northyoung
northyoung commented on Oct 15, 2014
原来问题出在git的客户端配置上！
@wy-ei
wy-ei commented on Oct 23, 2014
其实命令行挺好用的 用一段时间就习惯了
@markyun
markyun commented on Oct 23, 2014
结论：最好不要用 windows 😄 +1
@Yannyezixin
Yannyezixin commented on Dec 2, 2014
结论：最好不要用 windows 😄 +1
@QianYuXiang
QianYuXiang commented on Mar 17, 2015
结论：最好不要用windows😄+1
 😄 1
@chenyongze
chenyongze commented on Sep 9, 2015
摒弃windows
@hax
hax commented on Sep 10, 2015
现在这个bug估计已经修好了。。。

话说用\r\n的也不止windows，比如按照http规范，协议里的换行都是\r\n。
 @alwaystest alwaystest referenced this issue in alwaystest/Blog on Sep 27, 2015
 Open
git 之 文件换行符 #3
@windqyoung
windqyoung commented on Dec 21, 2015
真是搞不懂, 无缘无故的为什么要改我的文件.
我把这选项全试过了, 都没用. 还是被改了.
这功能真恶心.
@momognu
momognu commented on Feb 1, 2016
不要用 windows...
我还不要用中文注释呢
@cssmagic
Owner
cssmagic commented on Feb 1, 2016
不要用 windows...
我还不要用中文注释呢
在公司项目中要求用纯英文写注释，估计坑会大得多。 😂
@anguiao
anguiao commented on Apr 15, 2016
额，事实上CR-LF才是正统。。打字机就是这么设计的。。不过纠结于这些也没什么意义了。。
 👍 2
@momognu
momognu commented on Apr 17, 2016
@anguiao 我还以为是微软一直坚持要用^M$来代表自己呢
@myst729
myst729 commented on Apr 17, 2016 • edited
@momognu

回想你看过的老电影里面使用打字机的场景，一行敲满后要做两件事：换行，回车。

这是两个不同的行为：

LF（换行，line feed）：将光标下移一行；
CR（回车，carriage return）：将光标移动到当前行的开头。
Typewriter

每输入一个字符，纸张托架会左移一小截（一个字符的宽度 + 字符间距，好在那时候打字机都是等宽字体 😃 ）。当托架移动到最左侧，即纸张右侧已经和敲针对齐时，会有响铃提示已经抵达行末，该另起一行了。

这时候拉动换行杆就会把纸往上卷一小截（行高，line height），然后把托架推到最右侧，使纸张左侧重新和敲针对齐（这个动作就叫回车，不知道“回车”译法出自何处，我猜测跟 carriage 另一个意思“马车”有关）。如果只换行不回车，那么第一行敲满以后，敲针始终在纸张右侧，没法继续输入；只回车不换行，所有的内容都敲到同一行里了。

后来一些改良的产品已经能让这两个动作连贯起来了，比如这个视频（Youtube），注意看视频快结束时他是怎么输入连续空行的。这个视频（Youtube）讲解更详细。

然而，不论设计怎样改进，在机械设备上操作再无缝，换行和回车仍然是两个不同的行为。
 👍 7
@momognu
momognu commented on Apr 18, 2016
@myst729 哦！我算是知道这段历史了，不过微软为什么在linux和mac都统一用单一$的今天，还一直坚持这个“仿古”行为？
@hax
hax commented on Apr 18, 2016
@momognu 为了保持兼容性。我前面也说过，实际上许多互联网协议里都要求换行是 CR LF。
 👍 1
@be5invis
be5invis commented on Jul 10, 2016 • edited
@momognu 因为当年 Multics 电脑机能强大，可以用驱动程序把 LF 转换成打印机的 CRLF（甚至是 CR-NUL-LF），而 PC 没有驱动的概念，因为跑不动
后来就懒得改了，那么多软件谁敢动他
@windqyoung
windqyoung commented on Jul 13, 2016
我在win10, 官方的git下设置 core.autocrlf = false, 依然会提示全被修改了, 后来改成 core.autocrlf = input 好了.

看起来把值 true, false, input 都试试就行.
 @xuguan xuguan referenced this issue in xuguan/myNetworkTools on Jul 14, 2016
 Closed
中文乱码问题 #1
@wuliupo
wuliupo commented on Nov 3, 2016 • edited
core.autocrlf = input 就是安装时候的第二项 @windqyoung
Git for Windows Install
@nimstill
nimstill commented on Jan 17
刚被坑过，thanks
