---
layout: article
title:  "A Guide to Getting Started With QUIC"
categories: research
tags: [QUIC, Google, tutorial]
toc: true
image:
    teaser: research/2016-12-30-A-Guide-to-Getting-Started-With-QUIC/teaser.jpg
date: 2016-12-30
---

[proto-quic](https://github.com/google/proto-quic)是[QUIC](https://www.chromium.org/quic)的独立库。
proto-quic包含了QUIC所需的Chromium代码和相关依赖。因此可以使用部分Chromium代码，
而不必依赖于完整的Chromium。 proto-quic旨在成为一个跨平台库，但目前只支持Chromium
已经支持的平台（或部分平台）。

QUIC并不是官方支持的Google产品。目前由一些QUIC开发人员作为一个编外计划保持更新
（理论上更新周期为一周）。最糟糕的情况下，如果Google的优先级改变，不再支持一个
独立的QUIC库。但proto-quic是完全开源的，任何感兴趣的社区可以克隆仓库并继续更新。

目前，唯一支持的平台是Linux（并且目前仅测试了Ubuntu版本），但Windows和iOS 应该
很快就会支持。

注意末尾带*的为必要步骤！！！

测试的proto-quic版本为：[Updating to 56.0.2912.0 (#15)](https://github.com/google/proto-quic/commit/0db5f234b86699c182394bb261ee226ab800b60f)

~~~bash
$ git reset --hard 0db5f234b86699c182394bb261ee226ab800b60f
$ git log

commit 0db5f234b86699c182394bb261ee226ab800b60f
Author: fayang <fayang@google.com>
Date:   Tue Nov 8 18:44:44 2016 +0800

    Updating to 56.0.2912.0 (#15)

    * Updating to 56.0.2912.0

    * Updating to 56.0.2912.0
~~~

---

## 1 下载项目仓库并安装相关依赖*

~~~bash
# 下载项目仓库
$ git clone https://github.com/google/proto-quic.git

# 跳转至项目目录
$ cd proto-quic

# 保持在"Updating to 56.0.2912.0 (#15)"版本(目前测试版本)的原始状态，若想尝试最新版，则忽略此语句
$ git reset --hard 0db5f234b86699c182394bb261ee226ab800b60f

# 注意下面这步很重要，否则之后的操作会无法找到gn或ninja等命令，注意此命令设置的环境变量只对当前终端有效
$ export PATH=$PATH:`pwd`/depot_tools

# 安装相关依赖(可能会遇到各种问题，下面会作详细说明)
$ ./proto_quic_tools/sync.sh
~~~

安装相关依赖时需要下载Chrome OS fonts，binutils等，可能需要使用代理。

### 1.1 手动下载相关依赖

若无法直接通过使用代理并执行`./proto_quic_tools/sync.sh`下载相关依赖，如
Chrome OS fonts和binutils。则先手动下载并解压相关文件到相应位置，再执行
`./proto_quic_tools/sync.sh`命令。

若已成功执行`./proto_quic_tools/sync.sh`，请跳过此小节。

#### 1.1.1 手动安装Chrome OS fonts

chrome fonts的安装脚本位于`proto-quic/src/build/linux/install-chromeos-fonts.py`，
其主要代码(若不想看代码，可跳过，直接看之后的总结)如下。

~~~python
#!/usr/bin/env python
# Copyright 2013 The Chromium Authors. All rights reserved.
# Use of this source code is governed by a BSD-style license that can be
# found in the LICENSE file.

# Script to install the Chrome OS fonts on Linux.
# This script can be run manually (as root), but is also run as part
# install-build-deps.sh.

import os
import shutil
import subprocess
import sys

URL_TEMPLATE = ('https://commondatastorage.googleapis.com/chromeos-localmirror/'
                'distfiles/%(name)s-%(version)s.tar.bz2')

# Taken from the media-fonts/<name> ebuilds in chromiumos-overlay.
SOURCES = [
  {
    'name': 'notofonts',
    'version': '20160310'
  }, {
    'name': 'robotofonts',
    'version': '2.132'
  }
]

## 此时可以知道，URLS为一个含两个字符串元素的数组
## URLS为['https://commondatastorage.googleapis.com/chromeos-localmirror/distfiles/notofonts-20160310.tar.bz2',
##       'https://commondatastorage.googleapis.com/chromeos-localmirror/distfiles/robotofonts-2.132.tar.bz2']
URLS = sorted([URL_TEMPLATE % d for d in SOURCES])



FONTS_DIR = '/usr/local/share/fonts'

def main(args):
  if not sys.platform.startswith('linux'):
    print "Error: %s must be run on Linux." % __file__
    return 1

  if os.getuid() != 0:
    print "Error: %s must be run as root." % __file__
    return 1

  if not os.path.isdir(FONTS_DIR):
    print "Error: Destination directory does not exist: %s" % FONTS_DIR
    return 1

  ## 此时可知，dest_dir为'/usr/local/share/fonts/chromeos'
  dest_dir = os.path.join(FONTS_DIR, 'chromeos')

  ## 此时可知，stamp为'/usr/local/share/fonts/chromeos/.stamp02'
  stamp = os.path.join(dest_dir, ".stamp02")

  ## 此时可知，下载程序判断的依据为：存在stamp，且stamp的字符串内容与URLS数组用'\n'连接而成的字符串的内容相同
  ## 注意还需将tarball的内容解压到tarball所在的文件夹中
  if os.path.exists(stamp):
    with open(stamp) as s:
      if s.read() == '\n'.join(URLS):
        ## 若下载程序就此退出，表明安装完chrome os fonts
        print "Chrome OS fonts already up to date in %s." % dest_dir
        return 0

## 若已手动完成了之后程序的内容，则设法使程序在此终止
## ------------------------------------------------------------------------
  if os.path.isdir(dest_dir):
    shutil.rmtree(dest_dir)
  os.mkdir(dest_dir)
  os.chmod(dest_dir, 0755)

  print "Installing Chrome OS fonts to %s." % dest_dir
  ## 循环取URLS数组中的url，URLS为['https://commondatastorage.googleapis.com/chromeos-localmirror/distfiles/notofonts-20160310.tar.bz2',
  ## 'https://commondatastorage.googleapis.com/chromeos-localmirror/distfiles/robotofonts-2.132.tar.bz2']
  for url in URLS:
    ## 此时可知，tarball为'/usr/local/share/fonts/chromeos/notofonts-20160310.tar.bz2'
    ##                 和'/usr/local/share/fonts/chromeos/robotofonts-2.132.tar.bz2'
    tarball = os.path.join(dest_dir, os.path.basename(url))
    ## 下载tarball压缩包 (必要步骤 1)
    subprocess.check_call(['curl', '-L', url, '-o', tarball])
    ## 解压tarball (必要步骤 2)
    subprocess.check_call(['tar', '--no-same-owner', '--no-same-permissions',
                           '-xf', tarball, '-C', dest_dir])
    ## 解压后，就可以删除tarball压缩包
    os.remove(tarball)

  ## 由此可知，readme为'/usr/local/share/fonts/chromeos/README'
  readme = os.path.join(dest_dir, "README")
  ## 写README文件
  with open(readme, 'w') as s:
    s.write("This directory and its contents are auto-generated.\n")
    s.write("It may be deleted and recreated. Do not modify.\n")
    ## __file__为'./src/build/linux/install-chromeos-fonts.py'
    s.write("Script: %s\n" % __file__)

  ## 写stamp文件 (必要步骤 3) ，stamp为'/usr/local/share/fonts/chromeos/.stamp02'
  with open(stamp, 'w') as s:
    s.write('\n'.join(URLS))

  ## 将所有的文件夹的权限设置为755，所有文件的权限设置为644
  for base, dirs, files in os.walk(dest_dir):
    for dir in dirs:
      os.chmod(os.path.join(base, dir), 0755)
    for file in files:
      os.chmod(os.path.join(base, file), 0644)

  return 0

if __name__ == '__main__':
  sys.exit(main(sys.argv[1:]))
~~~

总结一下安装Chrome OS fonts所需的必要步骤：

##### 必要步骤 1 - 下载tarball压缩包

下载下面两个文件到`/usr/local/share/fonts/chromeos`目录中。(注意：如下载地址有变化，
以`proto-quic/src/build/linux/install-chromeos-fonts.py`的内容为准)

~~~bash
['https://commondatastorage.googleapis.com/chromeos-localmirror/distfiles/notofonts-20160310.tar.bz2',
'https://commondatastorage.googleapis.com/chromeos-localmirror/distfiles/robotofonts-2.132.tar.bz2']
~~~

~~~bash
# 可用浏览器下载(推荐)到/usr/local/share/fonts/chromeos目录中，或者在终端中wget下载
$ wget --no-check-certificate https://commondatastorage.googleapis.com/chromeos-localmirror/distfiles/notofonts-20160310.tar.bz2 -O /usr/local/share/fonts/chromeos/notofonts-20160310.tar.bz2
$ wget --no-check-certificate https://commondatastorage.googleapis.com/chromeos-localmirror/distfiles/robotofonts-2.132.tar.bz2 -O /usr/local/share/fonts/chromeos/robotofonts-2.132.tar.bz2
~~~

##### 必要步骤 2 - 解压tarball

~~~bash
$ tar -vxf --no-same-owner --no-same-permissions /usr/local/share/fonts/chromeos/notofonts-20160310.tar.bz2 -C /usr/local/share/fonts/chromeos/
$ tar -vxf --no-same-owner --no-same-permissions /usr/local/share/fonts/chromeos/robotofonts-2.132.tar.bz2 -C /usr/local/share/fonts/chromeos/
~~~

##### 必要步骤 3 - 写stamp文件

~~~bash
$ sudo python [进入python终端后输入如下内容]
>>> URLS = ['https://commondatastorage.googleapis.com/chromeos-localmirror/distfiles/notofonts-20160310.tar.bz2',
           'https://commondatastorage.googleapis.com/chromeos-localmirror/distfiles/robotofonts-2.132.tar.bz2']
>>> stamp = '/usr/local/share/fonts/chromeos/.stamp02'
>>> with open(stamp, 'w') as s:
      s.write('\n'.join(URLS))

>>> exit()  [退出python终端]
~~~

此时，Chrome OS fonts的必要安装步骤就完成了，`install-chromeos-fonts.py`文件中还
删除了解压后的tarball压缩包，并在同一目录下生成一个README文件。

#### 1.1.2 手动安装binutils

binutils的安装脚本位于`proto-quic/src/third_party/binutils/download.py`，
其主要代码(若不想看代码，可跳过，直接看之后的总结)如下。

~~~bash
#!/usr/bin/env python
# Copyright 2014 The Chromium Authors. All rights reserved.
# Use of this source code is governed by a BSD-style license that can be
# found in the LICENSE file.
# vim: set ts=2 sw=2 et sts=2 ai:

"""Minimal tool to download binutils from Google storage.

TODO(mithro): Replace with generic download_and_extract tool.
"""

import argparse
import os
import platform
import re
import shutil
import subprocess
import sys


BINUTILS_DIR = os.path.abspath(os.path.dirname(__file__))
BINUTILS_FILE = 'binutils.tar.bz2'
BINUTILS_TOOLS = ['bin/ld.gold', 'bin/objcopy', 'bin/objdump']
BINUTILS_OUT = 'Release'

DETECT_HOST_ARCH = os.path.abspath(os.path.join(
    BINUTILS_DIR, '../../build/detect_host_arch.py'))


def ReadFile(filename):
  with file(filename, 'r') as f:
    return f.read().strip()


def WriteFile(filename, content):
  assert not os.path.exists(filename)
  with file(filename, 'w') as f:
    f.write(content)
    f.write('\n')


def GetArch():
  gyp_host_arch = re.search(
      'host_arch=(\S*)', os.environ.get('GYP_DEFINES', ''))
  if gyp_host_arch:
    arch = gyp_host_arch.group(1)
    # This matches detect_host_arch.py.
    if arch == 'x86_64':
      return 'x64'
    return arch

  return subprocess.check_output(['python', DETECT_HOST_ARCH]).strip()

## 关键的函数，获取并解压binutils.tar.bz2文件
def FetchAndExtract(arch):
  ## 由此可知，archdir为'<base path>/proto-quic/src/third_party/binutils/Linux_ia32(或Linux_x64)'
  ## arch可由终端下的命令arch得出，若显示x86_64则为x64，否则为ia32
  archdir = os.path.join(BINUTILS_DIR, 'Linux_' + arch)

  ## 由此可知，tarball为'<base path>/proto-quic/src/third_party/binutils/Linux_ia32(或Linux_x64)/binutils.tar.bz2'
  tarball = os.path.join(archdir, BINUTILS_FILE)

  ## 由此可知，outdir为'<base path>/proto-quic/src/third_party/binutils/Linux_ia32(或Linux_x64)/Release'(注意区分大小写，Release的首字母大写)
  outdir = os.path.join(archdir, BINUTILS_OUT)

  ## 由此可知，sha1file为'<base path>/proto-quic/src/third_party/binutils/Linux_ia32(或x64)/binutils.tar.bz2.sha1'
  sha1file = tarball + '.sha1'
  if not os.path.exists(sha1file):
    print "WARNING: No binutils found for your architecture (%s)!" % arch
    return 0

  ## 由此可知，checksum为sha1file的内容，即所下载的tarball的文件的sha1校验码，以确定所下载文件为所需文件
  checksum = ReadFile(sha1file)

  ## 由此可知，stampfile为'<base path>/proto-quic/src/third_party/binutils/Linux_ia32(或Linux_x64)/binutils.tar.bz2.stamp'
  stampfile = tarball + '.stamp'

  ## 此时可知，下载程序判断的依据为：存在stampfile，存在tarball，存在outdir，且stampfile的字符串内容与sha1file的字符串内容相同
  ## 注意还需将tarball的内容解压到outdir中，即outdir文件夹中的结构为bin、include和lib。
  if os.path.exists(stampfile):
    if (os.path.exists(tarball) and
        os.path.exists(outdir) and
        checksum == ReadFile(stampfile)):
      ## 若下载程序就此退出，表明安装完binutls
      return 0
    else:
      os.unlink(stampfile)

## 若已手动完成了之后程序的内容，则设法使程序在此终止
## ------------------------------------------------------------------------

  ## 此处隐藏了真实的下载地址，调用了proto-quic/depot_tools/download_from_google_storage.py进行下载
  ## download_from_google_storage下载地址为gs://<bucket>/<sha1>
  ## 此处的bucket参数为chromium-binutils，故下载地址为gs://chromium-binutils/<sha1file文件存放的sha1>
  ## 例如，CPU为x86_64架构，且'<base path>/proto-quic/src/third_party/binutils/Linux_x64'目录下的sha1file的内容为d9064388bed0e7225b1366d80b59289b1509d7c2
  ## (若CPU为ia32架构，则选取'<base path>/proto-quic/src/third_party/binutils/Linux_ia32'目录下sha1file的内容)
  ## arch可由终端下的命令arch得出，若显示x86_64则为x64，否则为ia32
  ## 并将'gs://'替换为'https://storage.googleapis.com/'
  ## 最终，组合而成的地址https://storage.googleapis.com/chromium-binutils/d9064388bed0e7225b1366d80b59289b1509d7c2  (x64架构，推荐，因ia32架构的如果下载不到，程序会选择下载x64架构的)
  ## (或者 https://storage.googleapis.com/chromium-binutils/<'.../binutils/Linux_ia32'目录下sha1file的内容>  (ia32架构) )

  ## 下载tarball文件 (必要步骤 1)，tarball为'<base path>/proto-quic/src/third_party/binutils/Linux_ia32(或Linux_x64)/binutils.tar.bz2'
  print "Downloading", tarball
  subprocess.check_call([
      'download_from_google_storage',
      '--no_resume',
      '--no_auth',
      '--bucket', 'chromium-binutils',
      '-s', sha1file])
  assert os.path.exists(tarball)

  ## 删除并新建outdir文件夹 (必要步骤 2)，outdir为'<base path>/proto-quic/src/third_party/binutils/Linux_ia32(或Linux_x64)/Release/'(注意区分大小写，Release的首字母大写)
  if os.path.exists(outdir):
    shutil.rmtree(outdir)
  assert not os.path.exists(outdir)
  os.makedirs(outdir)
  assert os.path.exists(outdir)

  ## 解压tarball文件到outdir文件夹中 (必要步骤 3)
  print "Extracting", tarball
  subprocess.check_call(['tar', 'axf', tarball], cwd=outdir)

  ## 确认outdir文件夹下,含有所有BINUTILS_TOOLS数组所列举的文件
  ## 其中，BINUTILS_TOOLS为['bin/ld.gold', 'bin/objcopy', 'bin/objdump']
  for tool in BINUTILS_TOOLS:
    assert os.path.exists(os.path.join(outdir, tool))

  ## 将sha1file的字符串内容写入到stampfile中 (必要步骤 4)
  WriteFile(stampfile, checksum)
  return 0


def main(args):
  parser = argparse.ArgumentParser(description=__doc__)
  parser.add_argument('--ignore-if-arch', metavar='ARCH',
                      action='append', default=[],
                      help='Do nothing on host architecture ARCH')

  options = parser.parse_args(args)

  if not sys.platform.startswith('linux'):
    return 0

  arch = GetArch()
  if arch in options.ignore_if_arch:
    return 0

  ## 执行FetchAndExtract函数，下载并解压binutil文件
  if arch == 'x64':
    return FetchAndExtract(arch)
  if arch == 'ia32':
    ret = FetchAndExtract(arch)
    if ret != 0:
      return ret
    # Fetch the x64 toolchain as well for official bots with 64-bit kernels.
    return FetchAndExtract('x64')

  print "Host architecture %s is not supported." % arch
  return 1


if __name__ == '__main__':
  sys.exit(main(sys.argv[1:]))
~~~

总结一下安装binutils所需的必要步骤：

##### 必要步骤 1 - 下载tarball压缩包

查看计算机的CPU架构

~~~bash
# 若显示的是x86_64，则CPU架构为x64，若显示ia32，则CPU架构为ia32
$ arch
~~~

以下的内容以x64为例，即目标文件夹为`proto-quic/src/third_party/binutils/Linux_x64/`
(若为ia32，则目标文件夹为`proto-quic/src/third_party/binutils/Linux_ia32/`，
注意修改下列的相关内容)。


x64架构的下载地址为`https://storage.googleapis.com/chromium-binutils/d9064388bed0e7225b1366d80b59289b1509d7c2`
(若为ia32架构，则下载地址为`https://storage.googleapis.com/chromium-binutils/24f937cfdad77bdcd6ad8cacc542d806f3eb4b0f`，
最后的字符串为目标文件夹下的`binutils.tar.bz2.sha1`文件的内容，注意修改下列的相关内容)

(注意: 如下载地址有变化，以`proto-quic/src/third_party/binutils/download.py`的内容为准)

下载binutil文件到`proto-quic/src/third_party/binutils/Linux_ia32(或Linux_x64)/`
目录中，并命名为`binutils.tar.bz2`。以x64架构为例。

~~~bash
# 可用浏览器下载(推荐)到proto-quic/src/third_party/binutils/Linux_ia32(或Linux_x64)目录中，或者在终端中wget下载
$ cd proto-quic/src/third_party/binutils/Linux_x64   (x64架构)
(ia32架构的为: cd proto-quic/src/third_party/binutils/Linux_ia32)

$ wget --no-check-certificate https://storage.googleapis.com/chromium-binutils/d9064388bed0e7225b1366d80b59289b1509d7c2 -O ./binutils.tar.bz2  (x64架构)
(ia32架构的为: wget --no-check-certificate https://storage.googleapis.com/chromium-binutils/24f937cfdad77bdcd6ad8cacc542d806f3eb4b0f -O ./binutils.tar.bz2)
~~~

##### 必要步骤 2 - 删除并新建outdir文件夹

outdir为`proto-quic/src/third_party/binutils/Linux_ia32(或Linux_x64)/Release/`
(注意区分大小写，Release的首字母大写)

~~~bash
# 先切换到[proto-quic/src/third_party/binutils/Linux_ia32(或Linux_x64)文件夹中]
$ rm -R Release
$ mkdir Release
~~~

##### 必要步骤 3 - 解压tarball文件到outdir文件夹中

~~~bash
# 先切换到[proto-quic/src/third_party/binutils/Linux_ia32(或Linux_x64)文件夹中]
$ tar -vaxf ./binutils.tar.bz2 -C ./Release/
~~~

##### 必要步骤 4 - 将sha1file的字符串内容写入到stampfile中

~~~bash
$ sudo python [进入python终端后输入如下内容]
>>> sha1file = '/home/vmu/proto-quic/src/third_party/binutils/Linux_x64/binutils.tar.bz2.sha1' (vmu为系统的用户名，需要自行替换，绝对路径)
(ia32架构的为 stampfile = '/home/vmu/proto-quic/src/third_party/binutils/Linux_ia32/binutils.tar.bz2.sha1' (vmu为系统的用户名，需要自行替换，绝对路径))

>>> stampfile = '/home/vmu/proto-quic/src/third_party/binutils/Linux_x64/binutils.tar.bz2.stamp'  (vmu为系统的用户名，需要自行替换，绝对路径)
(ia32架构的为 stampfile = '/home/vmu/proto-quic/src/third_party/binutils/Linux_ia32/binutils.tar.bz2.stamp'  (vmu为系统的用户名，需要自行替换，绝对路径))

>>> checksum = None
>>> with file(sha1file, 'r') as f:
      checksum = f.read().strip()

>>> with file(stampfile, 'w') as f:
      f.write(checksum)
      f.write('\n')

>>> exit() [退出python终端]
~~~

此时，binutils的安装步骤就完成了。

然后，记得在处于`proto-quic`目录的终端下，再次执行`./proto_quic_tools/sync.sh`命令。

成功时会出现：

~~~bash
No missing packages, and the packages are up to date.

Installing Chrome OS fonts.
Installing Chrome OS fonts to /usr/local/share/fonts/chromeos.
  % Total % Received % Xferd Average Speed Time Time Time Current
                                                   Dload Upload Total Spent Left Speed
100 11.7M 100 11.7M 0 0 210k 0 0:00:57 0:00:57 --:--:-- 233k
  % Total % Received % Xferd Average Speed Time Time Time Current
                                                   Dload Upload Total Spent Left Speed
100 2811k 100 2811k 0 0 434k 0 0:00:06 0:00:06 --:--:-- 520k
Installing symbolic links for NaCl.
Creating link: /usr/lib/i386-linux-gnu/libcrypto.so
Creating link: /usr/lib/i386-linux-gnu/libssl.so
Downloading /home/vmu/proto-quic/src/third_party/binutils/Linux_x64/binutils.tar.bz2
NOTICE: You have PROXY values set in your environment, but gsutil in depot_tools does not (yet) obey them.
Also, --no_auth prevents the normal BOTO_CONFIG environment variable from being used.
To use a proxy in this situation, please supply those settings in a .boto file pointed to by the NO_AUTH_BOTO_CONFIG environment var.
0> Downloading /home/vmu/proto-quic/src/third_party/binutils/Linux_x64/binutils.tar.bz2...
Success!
Downloading 1 files took 61.413792 second(s)
Extracting /home/vmu/proto-quic/src/third_party/binutils/Linux_x64/binutils.tar.bz2
done.
~~~

---

## 2 生成QUIC的client，server和tests*

~~~bash
# 由proto-quic进入proto-quic/src目录
$ cd src

# 编译并生成quic_client,quic_server和net_unittests
$ gn gen out/Debug && ninja -C out/Debug quic_client quic_server net_unittests
(或者(如未按第一步添加环境变量) ../depot_tools/gn gen out/Debug && ../depot_tools/ninja -C out/Debug quic_client quic_server net_unittests
[gn位于proto-quic/depot_tools，输出目录为out/Default或out/Debug等均可，名字无要求，后文中为out/Debug])
~~~

成功时会出现:

~~~bash
ninja: Entering directory `out/Default'
[2785/2785] LINK ./quic_client
~~~

此时QUIC所依赖的环境已经搭建完成，
转入下面的**Playing with QUIC**部分。

---

#### [Playing with QUIC](https://www.chromium.org/quic/playing-with-quic)

除了使用proto-quic，还可以使用完整的Chromium环境，并类似proto-quic的使用方法，
生成`quic_server`和`quic_client`二进制文件。

---

## 3 准备测试数据，网页信息来自www.example.org*

### 3.1 下载www.example.org网页(包括HTTP头部)， 将由quic_server来维护*

~~~bash
# 在用户的家目录新建quic-data目录，此时的$HOME目录为'/home/<user name>'，注意若以root用户运行，其$HOME目录为'/'根目录
$ mkdir $HOME/quic-data

# 进入quic-data目录中
$ cd $HOME/quic-data

# 下载https://www.example.org网页(包括HTTP头部)
$ wget -p --save-headers https://www.example.org

# 回到原来的目录proto-quic中
$ cd -
~~~

### 3.2 手动编辑index.html，修改HTTP头部*

~~~bash
# 手动编辑index.html，修改HTTP头部，注意此时的$HOME目录为'/home/<user name>'，不要用root权限运行，否则其$HOME目录为'/'根目录
$ vi $HOME/quic-data/www.example.org/index.html

index.html的HTTP头部为<!doctype html>标签的上部分内容，与<!doctype html>标签要留有一行空行。
其他文件的HTTP头部为与正文相隔一行空行的顶部部分。
HTTP头部的编辑方法如下，
- 移除 (如果有的话): "Transfer-Encoding: chunked"
- 移除 (如果有的话): "Alternate-Protocol: ..."
- 增加 (如果第一行不是的话): HTTP/1.1 200 OK [必须加上，否则客户端访问时会出现500错误]
- 添加 (如果没有的话): X-Original-Url: https://www.example.org/index.html [必须加上，否则客户端访问时会出现404错误]

即index.html的内容至少为以下的格式：
(实际上www.example.org文件夹内的所有文件(包括隐藏文件)的内容也至少为以下格式，
X-Original-Url的路径根据相应文件的具体的路径和文件名而定，
否则之后运行服务器程序时会因无法取出HTTP响应头部信息出错)
HTTP/1.1 200 OK^M
X-Original-Url: https://www.example.org/index.html^M
^M     [一个空行划分HTTP头部和正文]
<!doctype html>...，如果是二进制文件或普通文本，则为二进制内容或普通的文本内容
(其中^M表示Windows的CR/LF换行，在vi中在编辑模式下可以用"ctrl+v, ctrl+m"输入，而非普通的可见字符。)
~~~

---

## 4 生成证书*

为了运行服务器，需要有效的证书和pkcs8格式的私钥。 如果没有，可以使用脚本来生成。

### 4.1 在服务器端生成HTTPS服务器证书[proto-quic/src目录下]*

~~~bash
# 由proto-quic/src进入net/tools/quic/certs目录
$ cd net/tools/quic/certs

# 运行脚本，生成pem格式的证书和pkcs8格式的私钥
$ ./generate-certs.sh
(注意脚本中生成证书的有效期为三天，可以修改脚本，搜索days，将其后的参数3改为365或其他更大的数字，这样就不用时常更新证书了)
~~~

除了服务器的证书和公钥，此脚本还将生成CA证书(`net/tools/quic/certs/out/2048-sha256-root.pem`)
您需要将其添加到操作系统的根证书存储，以便在证书验证期间使其可信。
为了在linux上这样做，请参阅这些[说明](https://chromium.googlesource.com/chromium/src/+/master/docs/linux_cert_management.md)。

### 4.2 在客户端中添加HTTPS服务器证书[proto-quic/src/net/tools/quic/certs目录下]*

~~~bash
# 安装管理证书的工具(选择符合自己Linux发行版的其中一行即可)：
- Debian/Ubuntu: $ sudo apt-get install libnss3-tools
- Fedora: su -c "yum install nss-tools"
- Gentoo: su -c "echo 'dev-libs/nss utils' >> /etc/portage/package.use && emerge dev-libs/nss" (You need to launch all commands below with the nss prefix, e.g., nsscertutil.)
- Opensuse: sudo zypper install mozilla-nss-tools

# 添加证书(证书类型-自签证书)，注意此时的$HOME目录为'/home/<user name>'，不要用root权限运行，否则其$HOME目录为'/'根目录：
$ certutil -d sql:$HOME/.pki/nssdb -A -t "P,," -n quic-server -i out/leaf_cert.pem

# 查看客户端中所添加的证书，检查一下是否添加成功：
$ certutil -d sql:$HOME/.pki/nssdb -L

# 删除证书，慎用，初次添加证书时请忽略。(除非再次添加相同名字的证书时，需要先删除原先添加的证书)：
$ certutil -d sql:$HOME/.pki/nssdb -D -n quic-server

# 返回之前的目录：
$ cd -

注意：
如果出现certutil: function failed: security library: bad database错误，
则删除证书数据库，再重建
$ mkdir -p ~/.pki/nssdb
$ sudo chmod 775 ~/.pki/nssdb
$ certutil -d ~/.pki/nssdb -N
如需要设置数据库的密码时，输入两个回车即可。

如果出现the certificate /key database is in an old ,unsupported format错误，则去掉前缀"sql:"，即
$　certutil -d $HOME/.pki/nssdb -A -t "P,," -n quic-server -i out/leaf_cert.pem
数据库类型从flat files到Berkeley DB，最后到sqllite, 所以最新版的话sql:<DIRECTORY LEVEL PATH OF DATABASE>是必须的，如果显示格式不支持，则去掉前缀"sql:"。
当然，同样可以新建数据库，这样就可以保证使用的是sqllite格式的。
~~~

---

## 5 运行QUIC服务器和客户端*

### 5.1 运行quic_server服务器 [proto-quic/src目录下]*

~~~bash
# 运行quic_server架设网站，此时的$HOME目录为'/home/<user name>'，注意若以root用户运行，其$HOME目录为'/'根目录
$ ./out/Debug/quic_server --quic_in_memory_cache_dir=$HOME/quic-data/www.example.org --certificate_file=net/tools/quic/certs/out/leaf_cert.pem --key_file=net/tools/quic/certs/out/leaf_cert.pkcs8  --port=6121 --v=1（如果需要显示更多的log信息，则使用该参数）

注意：./out/Debug/quic_server是服务器程序，后面的均为参数。

如果程序立即停止运行而不是处于Listening状态，则需要根据--v=1显示的信息来进行调整。
重点检查1： www.example.org文件夹内的所有文件(包括隐藏文件)的内容必须保持3.2所提格式。
#查看www.example.org文件夹内所有文件的命令:
$ ls -a /home/<user name>/quic-data/www.example.org/
~~~

quic_server程序的其他可选参数:

~~~bash
$./out/Debug/quic_server -h
Usage: quic_server [options]

Options:
-h, --help show this help message and exit
--port=<port> specify the port to listen on
--quic_in_memory_cache_dir directory containing response data
  to load
--certificate_file=<file> path to the certificate chain
--key_file=<file> path to the pkcs8 private key
--v=1 show verbose which level equals to 1
~~~

### 5.2 使用quic_client客户端，通过QUIC请求文件[proto-quic/src目录下]*

~~~bash
$ ./out/Debug/quic_client --host=127.0.0.1 --port=6121 https://www.example.org/index.html --v=1(如果需要显示更多的信息，则使用该参数） --disable-certificate-verification(如果在客户端没有添加服务器根证书的信任，则使用该参数)  

注意：./out/Debug/quic_client是客户端程序，后面的均为参数。
如果让服务器的端口默认为6121，您必须指定客户端端口，因为它默认为80。 此外，如果您的本地计算机有多个环回地址（如果使用IPv4和IPv6），您必须选择一个特定的地址。

如果仍然有错误如404或500，则需要根据--v=1显示的信息来进行调整。
重点检查1： quic_client请求的文件https://www.example.org/index.html，必须与在服务器目录中的文件index.html中的HTTP头部 X-Original-Url: https://www.example.org/index.html^M 相对应
(其中^M表示Windows的CR/LF换行，在vi中在编辑模式下可以用"ctrl+v, ctrl+m"输入，而非普通的可见字符。)
重点检查2:  服务器是否生成了证书，客户端是否添加了相应证书
重点检查3:  服务器目录中的文件index.html中的HTTP头部的第一行必须为HTTP/1.1 200 OK^M，且HTTP头部与正文之间必须相隔一个空行。
~~~

quic_client程序的其他可选参数:

~~~bash
$./out/Debug/quic_client -h
Usage: quic_client [options] <url>

<url> with scheme must be provided (e.g. http://www.google.com)

Options:
-h, --help show this help message and exit
--host=<host> specify the IP address of the hostname to connect to
--port=<port> specify the port to connect to
--body=<body> specify the body to post
--body_hex=<body_hex> specify the body_hex to be printed out
--headers=<headers> specify a semicolon separated list of key:value pairs to add to request headers
--quiet specify for a quieter output experience
--quic-version=<quic version> specify QUIC version to speak
--version_mismatch_ok if specified a version mismatch in the handshake is not considered a failure
--redirect_is_success if specified an HTTP response code of 3xx is considered to be a successful response, otherwise a failure
--initial_mtu=<initial_mtu> specify the initial MTU of the connection
--disable-certificate-verification do not verify certificates
--v=1 show verbose which level equals to 1
~~~

特别说明 - 如果传输其他文件:

~~~bash
如果需要传输视频、图像、文本等任何其他文件(包括index.html)，则需要在该文件的头部加上Http头部。
例如添加了bunny_480_100kbit_dash.mp4到quic-data/www.example.org目录下。

注意：www.example.org文件夹内的所有文件(包括隐藏文件)的内容也必须保持以下格式，
否则之后运行服务器程序时可能因无法取出HTTP响应头部信息出错。
#查看www.example.org文件夹内所有文件的命令:
$ ls -a /home/<user name>/quic-data/www.example.org/

# 注意保证 文件夹 的默认权限为775，非可执行文件 的默认权限为664。
$ sudo chmod 775 /home/<user name>/quic-data/www.example.org/
$ sudo chmod 664 /home/<user name>/quic-data/www.example.org/bunny_480_100kbit_dash.mp4

-------------------编辑 bunny_480_100kbit_dash.mp4 文件----------------------
HTTP/1.1 200 OK^M
X-Original-Url: https://www.example.org/bunny_480_100kbit_dash.mp4^M
^M
..................(文件正文二进制内容)
------------------------------End-------------------------------------------

注意Http的头部(Head)与正文(Body)需要相隔一个空行，其中上述两个头部内容是必须的，当然还可以添加其他的Http头部信息。
其中^M表示Windows的CR/LF换行，在vi中在编辑模式下可以用"ctrl+v, ctrl+m"输入，而非普通的可见字符。
注意Http头部中不能含有"Transfer-Encoding: chunked" 和 "Alternate-Protocol: ..."。

# 服务器端不需要变动[proto-quic/src目录下]：
./out/Debug/quic_server --quic_in_memory_cache_dir=$HOME/quic-data/www.example.org --certificate_file=net/tools/quic/certs/out/leaf_cert.pem --key_file=net/tools/quic/certs/out/leaf_cert.pkcs8 --port=6121 --v=1

# 客户端则需要改变请求的URL地址[proto-quic/src目录下]：
./out/Debug/quic_client --host=127.0.0.1 --port=6121 https://www.example.org/bunny_480_100kbit_dash.mp4 --disable-certificate-verification --v=1

# 或者使用google-chrome浏览器访问 (建议先在浏览器设置中清空浏览器缓存)：
google-chrome --user-data-dir=/tmp/chrome-profile --no-proxy-server --enable-quic --origin-to-force-quic-on=www.example.org:443 --host-resolver-rules='MAP www.example.org:443 127.0.0.1:6121' https://www.example.org/bunny_480_100kbit_dash.mp4
~~~

如果出现以下认证问题，则请按4.2删除证书后，再次添加证书。或者在client命令后后加上
`--disable-certificate-verification`参数。

~~~bash
[1109/061201:ERROR:cert_verify_proc_nss.cc(942)] CERT_PKIXVerifyCert for www.example.org failed err=-8179
[1109/061201:WARNING:proof_verifier_chromium.cc(466)] Failed to verify certificate chain: net::ERR_CERT_AUTHORITY_INVALID
Failed to connect to 127.0.0.1:6121. Error: QUIC_PROOF_INVALID
~~~

成功时会显示:

~~~bash
Connected to 127.0.0.1:6121
Request:
headers:
{
  :method:GET
  :authority:www.example.org
  :scheme:https
  :path:/index.html
}
body:

Response:
headers:
{
  :status:200
  cache-control:max-age=604800
  content-type:text/html
  date:Wed, 09 Nov 2016 12:50:08 GMT
  etag:"359670651+ident"
  expires:Wed, 16 Nov 2016 12:50:08 GMT
  last-modified:Fri, 09 Aug 2013 23:54:35 GMT
  server:ECS (cpm/F845)
  vary:Accept-Encoding
  x-cache:HIT
  x-ec-custom-error:1
  content-length:1270
  x-original-url:https://www.example.org/index.html
}

body: <!doctype html>
<html>
<head>
  <title>Example Domain</title>

  <meta charset="utf-8" />
  <meta http-equiv="Content-type" content="text/html; charset=utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <style type="text/css">
  body {
  background-color: #f0f0f2;
  margin: 0;
  padding: 0;
  font-family: "Open Sans", "Helvetica Neue", Helvetica, Arial, sans-serif;

  }
  div {
  width: 600px;
  margin: 5em auto;
  padding: 50px;
  background-color: #fff;
  border-radius: 1em;
  }
  a:link, a:visited {
  color: #38488f;
  text-decoration: none;
  }
  @media (max-width: 700px) {
  body {
  background-color: #fff;
  }
  div {
  width: auto;
  margin: 0 auto;
  border-radius: 0;
  padding: 1em;
  }
  }
  </style>
</head>

<body>
<div>
  <h1>Example Domain</h1>
  <p>This domain is established to be used for illustrative examples in documents. You may use this
  domain in examples without prior coordination or asking for permission.</p>
  <p><a href="http://www.iana.org/domains/example">More information...</a></p>
</div>
</body>
</html>

Request succeeded (200).
~~~

### 5.3 也可以使用chrome浏览器作为客户端来请求文件

Ubuntu 64bit的google-chrome浏览器下载地址:
[google-chrome-stable_current_amd64.deb](https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb)。

下载后，使用dpkg进行安装。

~~~bash
# 安装chrome浏览器
$ dpkg -i google-chrome-stable_current_amd64.deb
~~~

接着，使用使用chrome请求文件。

~~~bash
$ google-chrome \
  --user-data-dir=/tmp/chrome-profile \
  --no-proxy-server \
  --enable-quic \
  --origin-to-force-quic-on=www.example.org:443 \
  --host-resolver-rules='MAP www.example.org:443 127.0.0.1:6121' \
  https://www.example.org/index.html

注意：可能浏览器程序的名字是chrome，而不是google-chrome.
浏览器Host绑定www.example.org:443 到 127.0.0.1:6121。
不能直接访问 127.0.0.1:6121 的原因是认证的问题，根证书添加的对应的host名是example.org，
所以浏览器访问的是 https://www.example.org/index.html 。

注意：如果改动了网页的内容，然后浏览器访问没有变化，则可能是浏览器缓存的问题。可以到浏览器里清空缓存(Clear browsing data - Cached images and files)。
~~~


## 参考文献

- [proto-quic - README](https://github.com/google/proto-quic/blob/master/README.md)
- [Playing with QUIC](https://www.chromium.org/quic/playing-with-quic)

---
The End.

zhlinh

Email: zhlinhng@gmail.com

2016-12-30
