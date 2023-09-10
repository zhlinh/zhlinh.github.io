---
layout: article
title: "macOS Env Setup"
categories: programing
tags: [macOS, env]
toc: true
image:
    teaser: programing/2023-08-27-macOS-Env-Setup/teaser.jpg

date: 2023-08-27
---

## 1. 输入篇

### 1.1 输入法

右上角点击输入法，点击最底下的设置
在弹出的窗口中左下角添加简体中文全拼、双拼输入法或者五笔输入法

#### [☆小鹤双拼](https://www.flypy.com/)

**推荐小鹤双拼输入法**

- 没有五笔基础的推荐使用双拼输入法，可有效减少`按键次数`，即所有字最多两次按键。基本上一个下午就可上手，上手后强制使用一周即可熟练
- 双拼键位方案推荐[小鹤双拼](https://www.flypy.com/)，左右手比较均衡，没有使用逗号分号作为按键
- 如果对减少`重码`有需求的话，可以增加辅助码，甚至字形（即部首）的学习。如果全拼每分钟能打 60 个字，双拼能达到 80，双拼加辅助码 100，双拼加双形 120
- 个人觉得普通双拼应该就可以满足大多数人的需求了，辅助码使用频率实际上也挺低的，生僻字词才会用到，常用字词基本首选或首屏（输入法也会自动记录或者支持设置常用的组合）

### 1.2 快捷键

#### [☆恢复F1、F2作为标准功能键的功能](https://support.apple.com/zh-cn/HT204436)

**推荐恢复F1、F2作为标准功能键的功能**

- 如果是笔记本电脑，推荐恢复F1、F2等作为标准功能键的功能，否则默认会是调节亮度明暗的功能
- 实际上亮度调节等使用频率很低，可以在按下Fn键+F1时作为亮度调节使用。设置方法如下
1. 选取苹果菜单  >“系统设置”
2. 点按边栏中的“键盘”
3. 点按右侧的“键盘快捷键”按钮
4. 点按边栏中的“功能键”
5. 打开“将 F1、F2 等键用作标准功能键”

### 1.3 系统设置

1. 建议将程序坞Dock移动到左侧，默认是在底部，但实际上通常宽度不会被用完，内容基本都显示在中间，两侧通常都会有留白。系统设置-桌面和程序坞-屏幕中的位置，设置为左侧
2. 建议提高键盘光标的移动速度，否则在terminal中移动光标时会特别的慢，在系统设置-键盘中，将光标重复速率提高到最高，将延迟调到最短。如果还想更快，则可通过`defaults write NSGlobalDomain KeyRepeat -int 1`来进行设置，这里可以设置值为1，设置页面中设置的最快值为2

## 2. 全局搜索篇

#### [☆Alfred-推荐开机启动](https://www.alfredapp.com/)

- 安装之后，可以关闭系统自带的spotlight了，系统设置-键盘-快捷键-关闭spotlight的快捷键，后面在Alfred的设置中，就可以将`CMD+Space`设置给Alfred了
- 免费版有基础的功能，但还是建议购买高级版，因为剪切板历史功能真的太好用了，尝试过其他剪切板历史的应用都没有Alfred的好，推荐特性-剪切板历史的快捷键设置成 `CMD+Shift+V`，建议在剪切板历史设置中关闭Snippets的显示，否则首行会被Snippets占据，首次使用时需要打开Accessbility中的开关设置
- 推荐在更新中设置关闭自动更新，否则在唤醒时总会提示有更新
- 首次使用时会提醒授予Document文件夹访问的权限

## 3. 浏览器篇

#### [☆Chrome浏览器-推荐常驻Dock](https://www.google.com/chrome/)

**推荐Chrome浏览器**

- 同步和插件生态做的很好

插件推荐，这个实际上可以单独成篇，因为插件生态才是一个好产品的灵魂，Chrome如是，VSCode如是，Vim如是。可以查看这个列表[awesome-chome](https://github.com/xyNNN/awesome-chrome)。以下是一些装了又删，删了又装，目前留下的

- [Bitwarden - Free Password Manager](https://chrome.google.com/webstore/detail/bitwarden-free-password-m/nngceckbapebfimnlniiiahkandclblb) - 密码管理工具
- [Create Link](https://chrome.google.com/webstore/detail/create-link/gcmghdmnkfdbncmnmlkkglmnnhagajbm) - 复制链接为多种格式，比如可以直接复制为markdown的形式，而且会自动带上网页的标题，对于分享链接实在太友好了，快捷键建议设置为`CMD+Shitf+C`
- [easyScholar](https://chrome.google.com/webstore/detail/easyscholar/njgedjcccpcfmjecccaajkjiphpddfji) - 显示各种文献排名，并且提供翻译、文献收藏功能，助力科研
- [Zotero Connector](https://chrome.google.com/webstore/detail/zotero-connector/ekhagklcjbdpajgpjgmbionohlpdbjgc) - 保存文献到Zotero中，配合Zotero方便进行文献阅读和管理
- [Evernote Web Clipper](https://chrome.google.com/webstore/detail/evernote-web-clipper/pioclpoplcdbaefihamjohnefbikjilc) -Evernote的剪藏工具
- [Fatkun Batch Download Image](https://chrome.google.com/webstore/detail/fatkun-batch-download-ima/nnjjahlikiabnchcpehcpkdeckfgnohf) - 批量下载图片
- [Linkclump](https://chrome.google.com/webstore/detail/linkclump/lfpjkncokllnfokkgpkobnkbkmelfefj) - 批量打开链接，建议快捷键设置为`Shift+鼠标左键`选择区域
- [Markdown Preview Plus](https://chrome.google.com/webstore/detail/markdown-preview-plus/febilkbfcbhebfnokafefeacimjdckgl) - 浏览器显示markdown预览
- [Momentum](https://chrome.google.com/webstore/detail/momentum/laookkfknpbbblfpciffpaejjkokdgca) - 新标签页的壁纸，设置中可以关闭Focus和TODO功能
- [Dark Reader](https://chrome.google.com/webstore/detail/dark-reader/eimadpbcbfnmbkopoojfekhnkhdbieeh) - 暗黑模式，如果喜欢暗黑模式的可以安装这个插件，可以方便切换暗黑
- [Online Dictionary Helper](https://chrome.google.com/webstore/detail/online-dictionary-helper/lppjdajkacanlmpbbcdkccjkdbpllajb) - 划词翻译插件，目前最好用的了，柯林斯词典、剑桥词典、牛津词典，还支持anki
- [QR Code Generator](https://chrome.google.com/webstore/detail/qr-code-generator/afpbjjgbdimpioenaedcjgkaigggcdpp) - 生成当前页面的二维码，有时候需要手机打开网页或者浏览时比较实用，而且也可以修改二维码的内容
- [Tab Session Manager](https://chrome.google.com/webstore/detail/tab-session-manager/iaiomicjabeggjcfkbimgmglanimpnae) - 保存当前所有打开网页的记录，相当于是书签的集合。因为很多时候打开的网页没有看完（或者这一系列网页属于同一个组合），需要切换到另一个电脑，或者切换到下一次查看。之前还用过同类的[Session Manager](https://chrome.google.com/webstore/detail/session-manager/mghenlmbmjcpehccoangkdpagbcbkdpc)，但还是Tab Session Manager更好用。使用时建议关闭自动保存的功能，或者限制自动保存的个数为1个，否则会产生默认10个自动保存的Tab组合
- [uBlock Origin](https://chrome.google.com/webstore/detail/ublock-origin/cjpalhdlnbpafiamejdnhcphjbkeiagm) - 可以自定义网页中需要屏蔽的元素
- [Vimium](https://chrome.google.com/webstore/detail/vimium/dbepggeogbaibhgnhhndojpepiihcmeb) - vim模式的快捷键，使用后可以不用鼠标浏览网页，有vim经验的应该会觉得相当好用，设置中可以修改自定义按键，推荐[这个方案](https://gist.github.com/zhlinh/773825e7f71da977eff8ef44d3546aea)，将`f`绑定为查找后新标签页打开（原先`f`是当前页面打开），将`>`绑定为向后翻页，等等
- [Tampermonkey](https://chrome.google.com/webstore/detail/tampermonkey/dhdgffkkebhmkfjojejmpbldmpobfkfo) -这个就是大杀器了，会极大拓展插件的边界，看到另一个插件世界，脚本可以查看[Greasy Fork](https://greasyfork.org/zh-CN)，但需要注意鉴别，推荐使用最近更新的，经常更新的，用户数较多的，而且不建议使用集合体

## 4. 密码管理工具篇

#### [☆Bitwarden-推荐加入到浏览器插件中](https://chrome.google.com/webstore/detail/bitwarden-free-password-m/nngceckbapebfimnlniiiahkandclblb?hl=zh-TW)

**推荐Bitwarden密码管理工具**

- 自从LastPass在2021年收费之后（免费用户不可同时同步PC电脑和手机，即PC和手机二选一），Bitwarden应该是毫无疑问绝对好用的平替了，[代码开源](https://github.com/bitwarden)，也支持从其他密码管理平台导入数据，详见[从LastPass导入数据](https://bitwarden.com/help/import-from-lastpass/)
- 注意注册之后，一定要开启两步验证（2FA），两步验证工具推荐使用[Authy](https://authy.com/) 
- 曾经也试过[KeepPass工具](https://keepass.info/download.html)，有各个平台的实现，但之后实在是因为同步需要DIY，有点繁琐而弃坑了。当然可能也是因为自身不够硬核，因为这样似乎更安全一些，但Bitwarden也声明是[零知识加密-Zero knowledge encryption](https://bitwarden.com/help/bitwarden-security-white-paper/)，即端到端加密，后台无法查看数据，丢失Master密码后无法恢复，目前觉得基本够用了
- 此外如果对私有部署有执念的，Bitwarden还支持私有部署，直接从Docker中部署即可，详见[Bitwardenrs/server](https://bitwarden.com/help/install-on-premise-manual/)

#### [☆Authy-推荐手机安装](https://authy.com/)

**推荐Authy两步验证（2FA）工具**

- 两步验证（2FA）的工具，比Google Authenticator要好用的多，支持备份
- 推荐手机安装，不太建议安装到PC电脑上

## 5. 截图工具篇

#### [☆Snipaste-推荐开机启动](https://zh.snipaste.com/)

**推荐Snipase截图工具**

- 全平台支持，在Windows上无疑应该是最好用的了，在macOS上也是数一数二的
- 截图是`F1`，截取或者复制的内容上屏显示是`F3`，如果是笔记本电脑，可能需要先[恢复F1、F2作为标准功能键的功能](https://support.apple.com/zh-cn/HT204436)
- 首次使用按F1时会要求屏幕截取的权限
- F3上屏之后双击上屏的内容（或者按ESC键）是取消显示，右键可以显示工具栏，支持各种标注工具
- 可选择开机启动，打开设置-通用-系统开机时启动

#### [☆Shottr-推荐开机启动](https://shottr.cc/)

**推荐Shottr截图工具作为Snipaste的补充**

- 首次打开时会要求屏幕截取的权限
- 首推其OCR识别的功能，相当好用。注意需要在设置-高级中，设置OCR语言为Chinese，否则会识别不出中文。识别的内容会自动复制到剪切板中，顶部也会弹出识别内容的提示
- 此外还有滚动截图的功能，这个也很无敌，可以设置滚动的区域，首次使用时会要求Accessibility的权限，打开后看到除了Shottr之外还有一个AEServer是iMAC系统自带的（AEServer不用打开Accessibility），仅Shottr打开就好
- 个人习惯修改其默认的快捷键，设置-快捷键，取消全屏截图的快捷键，因为很少用到；区域截图修改为`Alt+F1`；滚动截屏修改为`Alt+F2`；QR识别修改为`Alt+F4`
- 可选择开机启动，打开设置-通用-系统开机时启动

## 6. 笔记篇

#### [☆MarkText-推荐常驻Dock](https://github.com/marktext/marktext)

**推荐MarkText作为简单Markdown的编写工具**

- 自从Typora收费之后，MarkText可以完全平替，也是所见即所得（WYSIWYG - What You See Is What You Get）全平台支持，开源，左侧SideBar的界面甚至还更喜欢一点，可能是更贴近VSCode的切换风格
- 如果是直接下载的安装包，打开后会提示是否放到废纸篓中，无法打开，需要先执行`sudo xattr -r -d com.apple.quarantine /Applications/MarkText.app`

#### [☆Evernote-推荐常驻Dock](https://evernote.com/blog/)

**推荐Evernote作为知识管理工具**

- 老牌的笔记应用，有[国际版](https://evernote.com/blog/)和[国内版](https://www.yinxiang.com/)，登陆国际版需要加/blog后缀，否则会被强制跳转到国内版。个人推荐国际版，因为可以有更丰富的API支持和联动，例如[IFTTT](https://ifttt.com/)的联动等，以及后续导入导出的考虑。但国际版的问题是官方没有Markdown的支持（Markdown支持需要配合VSCode的[EverMonkey](https://marketplace.visualstudio.com/items?itemName=michalyao.evermonkey)插件），国内版直接支持Markdown
- 注意注册之后，一定要开启两步验证（2FA），两步验证工具推荐使用[Authy](https://authy.com/) 
- 可作为第二大脑，剪藏和搜索都相当不错，用于知识管理。当然如果是简单的记录备忘，实际上可以直接使用[滴答清单](https://www.dida365.com/#q/all/tasks)

#### [☆Obsidian-推荐常驻Dock](https://obsidian.md/)

 **推荐Obsidian作为双向链接需求的工具**

- 如果对离线笔记应用有需求的，推荐Obsidian，官方也支持同步功能，设置中打开同步即可，也可以用WedDev实现同步。如果想迁移Evernote的笔记尝试可以借助[yarle](https://github.com/akosbalasko/yarle)
- 当然，其最重点的功能是支持**双向链接**，方便构建知识图谱，且插件生态相当丰富
- 支持vimBinding，简直程序员的福音，但启用vim之后有一个比较烦的就是中英文转换的问题，因为输入时是中文，但转换到normal模式时的操作又需要英文，希望可以有插件可以解决这一问题，Idea编辑器的vim插件就有支持normal下自动切换为英文的设置
- 至于说是否可以直接替代MarkText，可能离散的md文件，比如README.md还是倾向于直接使用MarkText打开来编辑，但需要存档笔记自然会使用Obsidian
- 插件和主题可以查看[awesome-obsidian](https://github.com/kmaasrud/awesome-obsidian)，主题目前比较喜欢[Shimmering Focus](https://github.com/chrisgrieser/shimmering-focus)，插件推荐[大纲视图-Quite Outline](https://github.com/guopenghui/obsidian-quiet-outline/tree/master)，确实比官方的要好用得多。设置中打开社区插件的开关，然后在社区插件中搜索安装即可，安装后使用`CMD+P`打开快捷命令模式，然后输入`Quite Outline`来显示大纲视图
- 插件建议一开始先不要安装太多，用一段时间后再按需安装，因为有些插件的使用还是有点复杂的，有的还涉及Canvas
- `CMD+P`可以打开快捷命令模式

#### [☆Notion-推荐常驻Dock](https://www.notion.so/)

**推荐Notion作为固定模版的写作工具**

- Notion的页面实在太好看了，而且有丰富的模版可以选择
- 注意注册之后，一定要开启两步验证（2FA），两步验证工具推荐使用[Authy](https://authy.com/) 
- 对笔记审美有需求的可以使用Notion来代替Evernote，官方直接支持导入，设置中选择导入可以选择Evernote，授权访问即可

## 7. GTD工具篇

#### [☆滴答清单-推荐常驻Dock](https://dida365.com/)

**推荐滴答清单作为GTD工具**

- 滴答清单无疑是最好用的GTD任务管理和待办事项工具了

- 滴答清单有国内版(dida365.com)和国际版(ticktick.com)，但目前选择了国内版，主要是艾宾浩斯重复提醒的需求，对于学习来说简直太好用了。另外国内版还有农历的功能

## 8. 压缩解压工具篇

#### [☆Bandizip-推荐安装](https://en.bandisoft.com/bandizip)

**推荐Bandizip作为压缩解压工具**

- Bandizip无广告，界面整洁，支持多种压缩格式

- 在Windows是免费的，在macOS上是收费的，可以买断，目前还没有找到功能相当的平替。目前来看还是很值得购买的

## 9. 网盘工具篇

- 网盘工具基本都大同小异
- [坚果云](ttps://www.jianguoyun.com/)算是其中比较特别的，按流量限制而不是按容量限制，而且支持WebDev，对于国内网盘工具来说来说简直太牛了
- 其他的就看买了哪家的会员了

## 10. 字体篇

#### [☆nerd-fonts-推荐安装](https://github.com/ryanoasis/nerd-fonts/tree/master)

- 因为ohmyzsh或者joshuto的主题大多需要nerd字体，所以此处主要推荐支持nerd字体的，当然实际上是在原字体的基础上增加了特殊符号的patch。

- 推荐使用[nerd-fonts](https://github.com/ryanoasis/nerd-fonts/tree/master)中的脚本进行安装，clone时记得一定要使用`git clone --depth 1 https://github.com/ryanoasis/nerd-fonts.git`指定层级为1进行下载，否则会特别慢。下载后可以使用`./install.sh <字体名>`来进行安装，推荐的字体有：
  
  1. [DejaVuSansMono]( https://github.com/ryanoasis/nerd-fonts/tree/master/patched-fonts/DejaVuSansMono )
  
  2. [DroidSansMono]( https://github.com/ryanoasis/nerd-fonts/tree/master/patched-fonts/DroidSansMono )
  
  3. [JetBrainsMono](https://github.com/ryanoasis/nerd-fonts/tree/master/patched-fonts/JetBrainsMono)
  
  4. [Noto](https://github.com/ryanoasis/nerd-fonts/tree/master/patched-fonts/Noto)

## 11. 终端篇

#### [☆iterm2-推荐常驻Dock](https://iterm2.com/)

- macOS上最好用的终端工具了
- 推荐使用[solarized](https://github.com/altercation/solarized)主题，项目中有一个iterm2-colors-solarized文件夹是给iterm2的，下载后，点击其中的`Solarized Dark.itermcolors`和`Solarized Light.itermcolors`安装，安装完成后，在Profile-Colors-Color Presets选择相应的Solarized即可
- 建议增大默认窗口的大小，在Profile-Window中，将列设置为120列，行设置为35
- 建议更换为Nerd字体，并增大字体的大小，在Profile-Text中更换Nerd系列的字体，并将字体大小设置为17

#### [☆brew-推荐安装](https://brew.sh/)

- macOS上的包管理工具，安装命令是`/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)" `
- 安装完成后，可以使用`brew install xxx`来安装对应的命令行工具

#### [☆ohmyzsh-推荐安装](https://ohmyz.sh/)

- 目前macOS已经默认是zsh了，ohmyzsh基本上也是必备安装的了，安装命令是`sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"`，执行前需要保证已经安装了[Xcode](https://developer.apple.com/xcode/)，安装完Xcode后还需要打开Xcode，首次打开时会提醒安装必要的组件
- 推荐使用[.zshrc配置](https://gist.github.com/zhlinh/8e9a0b3b22d758fd24d16e3cdc772de4)，注意需要安装[git-open](https://github.com/paulirish/git-open)，[zsh-autosuggestions](https://github.com/zsh-users/zsh-autosuggestions)和[zsh-syntax-highlighting](https://github.com/zsh-users/zsh-syntax-highlighting)
- 主题建议使用`agnoster`主题，配置中使用了基于该主题自定义的主题[custom_agnoster.zsh-theme](https://gist.github.com/zhlinh/8e9a0b3b22d758fd24d16e3cdc772de4#file-custom_agnoster-zsh-theme)，需要将该文件拷贝到`/Users/$USER/custom/themes/`目录中，主要变更是不显示主机名，只显示用户名
- 此外使用`agnoster`主题时需要安装[nerd字体](https://github.com/ryanoasis/nerd-fonts#option-4-homebrew-fonts)，否则很多符号都无法显示，安装字体后，在iterm2设置-Profile-Text中可以更换Nerd系列的字体

```bash
# 需要安装的
# git-open，在git项目中使用git open可以用浏览器打开对应的链接
git clone https://github.com/paulirish/git-open.git $ZSH_CUSTOM/plugins/git-open
# zsh-autosuggestions，命令提示
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
# zsh-syntax-highlighting，语法高亮
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
# 其他自带的
# z，目录跳转，可以通过z 关键字来跳转目录
```

#### [☆额外的命令行工具-推荐使用brew安装](https://github.com/toolleeo/cli-apps)

以下的都可以通过`brew install xxx`来安装，最好选用c++或者rust实现的，性能会好的多

- [tldr](https://github.com/tldr-pages/tldr) - 替代man，man很多时候都抓不到重点，tldr会给出最常用的例子，以下的命令不懂怎么使用时都可以使用`tldr <xxx>`来查看使用示例

- [ripgrep](https://github.com/BurntSushi/ripgrep) - 替代grep，实在是太好用，而且快很多，不指定目录/文件时会搜索当前目录的所有text文件。安装后使用`rg <xxx正则表达式> [目录/文件名]`查找字符

- [ripgrep-all](https://github.com/phiresky/ripgrep-all) - ripgrep的升级版，不只查找文本文档，而且还会查找pdf，ebook，office文件和压缩文件(zip和tar.gz等)，安装后使用`rga <xxx正则表达式> [目录/文件名]`查找字符

- [fd](https://github.com/sharkdp/fd) - 替代find，优化常用选项为默认，也会快很多，不指定目录/文件时会搜索当前目录的所有文件。安装后使用`fd <xxx正则表达式> [目录/文件名]`查找文件

- [fpp](https://github.com/facebook/PathPicker) - 文件选择器，例如可以使用`ls | fpp`或者`git status | fpp`或者`fd test | fpp`

- [joshuto](https://github.com/kamiyaa/joshuto) - 替代ranger，终端的文件管理器，使用vim的按键模式，而且支持文件预览。建议添加alias到`~/.bash_profile`中，`alias ra=joshuto`，这样就可以用`ra`来唤醒文件管理器

## 12. IDE篇

#### [☆JetBrains Toolbox-推荐安装](https://www.jetbrains.com/toolbox-app/)

jetbrain的全家桶，安装后可以按需下载对应语言的IDE，也可以使用Toolbox来更新。PyCharm Community，Clion，Rider，AppCode，Android Studio等

#### [☆VSCode Insider-推荐常驻Dock](https://code.visualstudio.com/insiders/)

之所以用VSCode Insider而不用VSCode，主要是为了使用GitHub Copilot的Chat功能，可以通过[该网址](https://docs.github.com/en/copilot/github-copilot-chat/using-github-copilot-chat )申请白名单。其他功能的和VSCode 相同



---

The End.

zhlinh

Email: zhlinhng@gmail.com

2023-08-27
