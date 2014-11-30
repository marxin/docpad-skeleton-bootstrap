---
layout: post
title: "Gentoo Linux Packages with GCC 4.9 LTO"
tags: ['Gentoo', 'GCC', 'LTO', 'post']
date: 2014-04-24 13:49:51 +0200
comments: true
published: true 
categories: Gentoo GCC LTO
---

Introduction
===

Gentoo Linux is a cool flavor of GNU/Linux that is world-known as a distribution building all packages from source code. Latest version of GCC, version 4.9.0, was released few days ago and this blog tests maturity of Link-Time optimization in the GNU Compiler Collection. My post was inspired by [Nikos Chantziaras](https://plus.google.com/113033135204616446740) who wrote similar post about 2 years ago. If you interested how to set-up LTO for a Gentoo installation, please follow [this link](http://realnc.blogspot.cz/2012/06/building-gentoo-linux-with-gcc-47-and.html). Apart from creation of package list that cannot be built with LTO, I spent some time with investigation of problems that break compilation of these packages.

Gentoo System
===

My Gentoo is a QEMU KVM virtual machine utilizing my i7-4770 CPU with following run options:

``` bash
qemu-kvm -cpu host,level=9 -smp cores=8 -vga std -hda vdisk.img -boot d -m 4096 -net nic,vlan=1 -net user,vlan=1 -redir tcp:2222::22
```

System consists of __802__ packages that cover all problematic packages listed in aforementioned blog, covering both essential libraries for KDE and GNOME desktop environment. For a complete list of installed packages and corresponding versions, please follow [this link](/files/gentoo-emerge-world-packages.txt). For everyone interested in my USE flags, download my [make.conf](/files/gentoo-make.conf).

Installation
---

At time of writing this post, I installed GCC from source code. I took configure options from stable Gentoo package and simply run: ./configure [options] && make && make install. After that, tell your Gentoo system about a new compiler by creation of file (/etc/env.d/gcc/x86_64-pc-linux-gnu-4.9.0) in /etc/env.d/gcc. Last step is to switch default compiler by running __gcc-config__ command. Nevertheless, preferable way would be usage of package from [Gentoo overlay](http://gpo.zugaina.org/sys-devel/gcc).

To make the toolchain really up-to-date, I prefer to install latest binutils, another essential part of LTO tool-chain. Important to notice, there is still need to either use LTO wrappers for __nm__, __ar__ and __ranlib__. I really recommend Markus' [patch](https://sourceware.org/ml/binutils/2014-01/msg00213.html) for binutils, you will prevent any problems related to correct loading of LTO plug-in. The patch has been just applied, use latest bintuils. I used standard Gentoo package binutils-9999 installed in following steps:

``` bash
emerge =binutils-9999
ln -s /usr/libexec/gcc/x86_64-pc-linux-gnu/4.9.0/liblto_plugin.so.0.0.0 /usr/x86_64-pc-linux-gnu/binutils-bin/lib/bfd-plugins
```

Applied patch causes binutils to automatically load LTO plug-in, where __ln__ command creates a symlink to default plug-in folder.


Problematic packages
===

There are __30__ packages that suffer from diverse problems and cannot be built with __-flto__. Full list of packages:

``` bash
app-admin/gam-server
app-crypt/mit-krb5
app-emulation/virtualbox
app-text/rarian
dev-lang/perl
dev-lang/ruby
dev-libs/elfutils
dev-python/notify-python
dev-python/numpy
dev-qt/qtscript
dev-qt/qtwebkit
dev-tex/luatex
dev-vcs/cvs
media-libs/alsa-lib
media-libs/x264
media-sound/pulseaudio
media-sound/wavpack
media-video/ffmpeg
media-video/libav
media-video/mplayer2
sys-apps/hwinfo
sys-apps/pciutils
sys-devel/llvm
www-client/chromium
www-client/firefox
x11-base/xorg-server
x11-drivers/xf86-video-intel
x11-libs/cairo
x11-libs/wxGTK
sys-libs/glibc
```

Problem analysis
---

These packages has many common issued that block proper compilation:

### Configuration scripts

 *  __dev-vcs/cvs__ - Configure script checks whether a function exists with pointer equality. Link-Time optimization proves these pointers are equal and optimizes out these symbols, for more details please read [LTO FAQ](http://gcc.gnu.org/wiki/LinkTimeOptimizationFAQ#Elimination_of_unused_functions).
 *  __dev-lang/ruby__ - Very similar problem, I created a bug [#9692](https://bugs.ruby-lang.org/issues/9692) that was fixed.
 *  __app-text/rarian__ - Execution of __nm_test_func__ fails. GCC, starting from version 4.9.0, does not generate fat objects files ([GCC 4.9 Release Series](http://gcc.gnu.org/gcc-4.9/changes.html)). Thus, no assembly output is presented in objects files. As a result, assembly tools like __nm__ cannot be used e.g. for symbol assembly extraction. For complete magic related to check, follow [pastebin snippet](http://pastebin.com/RF1VubdR).
 *  __dev-python/notify-python__ - Likewise.
 *  __media-sound/wavpack__ - Likewise.
 *  __notify-python__ - Likewise.
 *  __x11-libs/wxGTK__ - Similar problem during checking for thr_setconsurrency.
 *  __x11-libs/cairo__ - Configure script compiles a source file with a float constant having magic number. After that, 'noonsees' is searched in the object file. If your machine is big endian, check succeeds. Again, existence of slim object file breaks the test. Cairo package builds with LTO internally, so even if you add the package to no-lto group, it is built with Link-Time optimization.
 *  __media-libs/x264__ - Likewise.

As you can see, aforementioned problems can be simply fixed by adding -fno-lto automatically to autoconf pass, Markus Tripperlsdorf suggested this solution on autoconf [mailing list](http://lists.gnu.org/archive/html/autoconf/2014-02/msg00005.html). On the other hand, every single problematic check must be fixed by hand. To make matters even worse, autoconf configure scripts are often a copy residing in version control system.

### Assembly usage

 *  __dev-tex/luatex__ - With a compiler, one can combine source code and assembly language ([Assembler Instructions with C expression Operands](http://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html)). Except of assembly code generation, compiler does not parse and understand assembler statements. If there's a constant symbol used in e.g. top-level assembler, GCC can't connect these symbols together. Thus, during LTRANS partitioning, these symbols can go to different partitions and linker error occures. For more detail information, follow [link](http://gcc.gnu.org/wiki/LinkTimeOptimizationFAQ#Symbol_usage_from_assembly_language).
 *  __media-video/libav__ - Likewise.
 *  __media-video/ffmpeg__ - Likewise.
 *  __media-video/mplayer2__ - Likewise.
 *  __ses-devel/llvm__ - Likewise.

### Linker related problems

Following group of tests suffer from linker issues, I haven't had time for more detail investigation. I think part of these packages cannot be built because of missing __\_\_attribute\_\_ ((used))__.

 *  __dev-lang/perl__
 *  __sys-apps/pciutils__
 *  __dev-libs/elfutils__
 *  __dev-qt/qtscript__
 *  __dev-qt/qtwebkit__

### Others

 *  __x11-drivers/xf86-video-intel__ - For a function marked with \_\_attribute\_\_ ((flatten)), every call (recursively) inside the function is inlined. With LTO, the compiler can process entire program analysis and the attribute can lead to extreme number of inlined functions. Before the compiler was killed, it performed inlining of about 3 million functions; I reported [a bug #77580](https://bugs.freedesktop.org/show_bug.cgi?id=77580).
 *  __media-sound/pulseaudio__ - dll_open related problem.
 *  __x11-base/xorg-server__ - Array bounds check is hit, there is [a bug #71127](https://bugs.freedesktop.org/show_bug.cgi?id=71127).
 *  __www-client/firefox__ - Firefox is almost ready for LTO, but there are audio/video codecs libraries that incorrectly use assembly symbols. Jan Hubička wrote very detail [post](http://hubicka.blogspot.cz/2014/04/linktime-optimization-in-gcc-2-firefox.html) about Firefox.
 *  __www-client/chromium__ - Chromium relates on gold linker as a part of source repository. Apart from that, there are also translation units that must be compiler without LTO.
 *  __app-crypt/mit-krb5__ - Package contains a conflicting name for a variable __link__ (conflict with: /usr/include/unistd.h:812:12: error: variable ‘link’ redeclared as function). ~~I have created a [pull request](https://github.com/krb5/krb5/pull/107) for krb5 Github repository.~~ __Update__: Eventually, the problem is caused by GCC which delays symbol renaming: [PR61012](http://gcc.gnu.org/bugzilla/show_bug.cgi?id=61012).
 *  __app-admin/gam-server__ - ~~Very similar issue, static variable 'socket' is redeclared. I sent email about the bug to corresponding mailing list.~~ __Update__: Likewise.
 *  __app-emulation/virtualbox__ - Not investigated yet.
 *  __media-libs/alsa-lib__ - Likewise.
 *  __sys-apps/hwinfo__ - Likewise.
 *  __dev-python/numpy__ - Likewise.
 *  __sys-libs/glibc__ - Likewise.

### Package unrelated problems

 *  __dev-qt/qtcore-4.8.5-r1:4__ + __kde-base/kdelibs-4.11.5:4/4.11__ - If I enable ld.gold to link following two packages, there's an infinite loop observed in meinproc4. BFD does not suffer from the issue observable even with -O0 and -fno-lto. I will create an issue after I understand the strange behavior.
 * There are a few packages, build with LTO, that have disproportionately big first ELF section. Looks the problem is not related to specific linker, both BFD and gold behave the same.

``` bash
$ readelf -S /usr/bin/gst-launch-1.0
There are 26 section headers, starting at offset 0x207740:

Section Headers:
[Nr] Name              Type             Address           Offset
Size              EntSize          Flags  Link  Info  Align
[ 0]                   NULL             0000000000000000  00000000
0000000000000000  0000000000000000           0     0     0
[ 1] .interp           PROGBITS         0000000000400200  00200200
000000000000001c  0000000000000000   A       0     0     1
[ 2] .note.ABI-tag     NOTE             000000000040021c  0020021c
0000000000000020  0000000000000000   A       0     0     4
[ 3] .hash             HASH             0000000000400240  00200240
0000000000000434  0000000000000004   A       4     0     8
[ 4] .dynsym           DYNSYM           0000000000400678  00200678
0000000000000cc0  0000000000000018   A       5     1     8
[ 5] .dynstr           STRTAB           0000000000401338  00201338
0000000000000b2f  0000000000000000   A       0     0     1
[ 6] .gnu.version      VERSYM           0000000000401e68  00201e68
...

$ ls -l /usr/bin/gst-launch-1.0
-rwxr-xr-x 1 root root 2129344 Apr 23 11:39 /usr/bin/gst-launch-1.0

```

__Update__: I found out that the binary is prolonged by __paxctl__ command (paxctl -qCm /usr/bin/gst-launch-1.0). I am not familiar with this tool, but LTO does not contribute to the initial ELF section expansion.

Statistics
===

To present statistical data about size reduction impact of LTO, I created a [python script](https://github.com/marxin/script-misc/blob/master/binaries_walker.py) that simply walks all ELF binaries located at $PATH and all shared libraries in folders /lib64/ and /usr/lib64. According to data presented in following tables, binaries shrink by __10.6%__ (__41 MB__) and shared libraries by __3.3%__ (__10 MB__). As you can see, Link-Time optimization can reach better results for executables that have (in general) less globally visible symbols. For both categories, I've chosen the most interesting results (complete statistics can be found [here](/files/lto-sizes-table.html)).

ELF executables
---

<table class="table table-stripped table-bordered table-condensed binaries-report">
  <thead>
    <tr>
      <th>Binary name</strong></th>
      <th class="tr">no-LTO size</th>
      <th class="tr">LTO size</th>
      <th class="tr">Fraction</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>/usr/bin/jsc-1</td>
      <td>2154080</td>
      <td>61104</td>
      <td>2.84%</td>
    </tr>
    <tr>
      <td>/usr/bin/git-shell</td>
      <td>692152</td>
      <td>25552</td>
      <td>3.69%</td>
    </tr>
    <tr>
      <td>/usr/bin/sapWatch</td>
      <td>623640</td>
      <td>33088</td>
      <td>5.31%</td>
    </tr>
    <tr>
      <td>/usr/bin/MPEG2TransportStreamIndexer</td>
      <td>605144</td>
      <td>37632</td>
      <td>6.22%</td>
    </tr>
    <tr>
      <td>/usr/bin/testRelay</td>
      <td>623384</td>
      <td>41856</td>
      <td>6.71%</td>
    </tr>
    <tr>
      <td>/usr/bin/testReplicator</td>
      <td>623576</td>
      <td>44224</td>
      <td>7.09%</td>
    </tr>
    <tr>
      <td>/usr/bin/testMPEG2TransportStreamTrickPlay</td>
      <td>606104</td>
      <td>44432</td>
      <td>7.33%</td>
    </tr>
    <tr>
      <td>/usr/bin/testMPEG1or2Splitter</td>
      <td>605016</td>
      <td>47960</td>
      <td>7.93%</td>
    </tr>
    <tr>
      <td>/usr/bin/testH264VideoToTransportStream</td>
      <td>604888</td>
      <td>48000</td>
      <td>7.94%</td>
    </tr>
    <tr>
      <td>/usr/bin/testMPEG1or2ProgramToTransportStream</td>
      <td>604824</td>
      <td>52056</td>
      <td>8.61%</td>
    </tr>
    <tr>
      <td>/sbin/thin_dump</td>
      <td>2215560</td>
      <td>228432</td>
      <td>10.31%</td>
    </tr>
    <tr>
      <td>/usr/bin/testMPEG1or2VideoReceiver</td>
      <td>623704</td>
      <td>64704</td>
      <td>10.37%</td>
    </tr>
    <tr>
      <td>/usr/bin/testMP3Receiver</td>
      <td>623704</td>
      <td>64704</td>
      <td>10.37%</td>
    </tr>
    <tr>
      <td>/usr/bin/testMPEG2TransportStreamer</td>
      <td>624408</td>
      <td>68864</td>
      <td>11.03%</td>
    </tr>
    <tr>
      <td>/usr/bin/testMP3Streamer</td>
      <td>624408</td>
      <td>69056</td>
      <td>11.06%</td>
    </tr>
    <tr>
      <td>/sbin/thin_restore</td>
      <td>2215544</td>
      <td>253128</td>
      <td>11.43%</td>
    </tr>
    <tr>
      <td>/usr/bin/testMPEG1or2VideoStreamer</td>
      <td>624408</td>
      <td>77376</td>
      <td>12.39%</td>
    </tr>
    <tr>
      <td>/sbin/thin_repair</td>
      <td>2215520</td>
      <td>299232</td>
      <td>13.51%</td>
    </tr>
    <tr>
      <td>/usr/bin/testMPEG1or2AudioVideoStreamer</td>
      <td>624472</td>
      <td>97864</td>
      <td>15.67%</td>
    </tr>
    <tr>
      <td>/usr/bin/wpa_passphrase</td>
      <td>36096</td>
      <td>5816</td>
      <td>16.11%</td>
    </tr>
    <tr>
      <td>/usr/bin/testAMRAudioStreamer</td>
      <td>624536</td>
      <td>101808</td>
      <td>16.30%</td>
    </tr>
    <tr>
      <td>/usr/bin/l2ping</td>
      <td>66720</td>
      <td>10960</td>
      <td>16.43%</td>
    </tr>
    <tr>
      <td>/usr/bin/testWAVAudioStreamer</td>
      <td>625176</td>
      <td>105968</td>
      <td>16.95%</td>
    </tr>
    <tr>
      <td>/usr/bin/testDVVideoStreamer</td>
      <td>624536</td>
      <td>105968</td>
      <td>16.97%</td>
    </tr>
    <tr>
      <td>/sbin/thin_rmap</td>
      <td>466400</td>
      <td>85576</td>
      <td>18.35%</td>
    </tr>
    <tr>
      <td>/usr/bin/testH264VideoStreamer</td>
      <td>624536</td>
      <td>114800</td>
      <td>18.38%</td>
    </tr>
    <tr>
      <td>/usr/bin/testMPEG4VideoStreamer</td>
      <td>624472</td>
      <td>118320</td>
      <td>18.95%</td>
    </tr>
    <tr>
      <td>/usr/bin/vobStreamer</td>
      <td>626136</td>
      <td>134896</td>
      <td>21.54%</td>
    </tr>
    <tr>
      <td>/usr/bin/ciptool</td>
      <td>111632</td>
      <td>29128</td>
      <td>26.09%</td>
    </tr>
    <tr>
      <td>/usr/bin/rfcomm</td>
      <td>75856</td>
      <td>20200</td>
      <td>26.63%</td>
    </tr>
    <tr>
      <td>/sbin/thin_check</td>
      <td>434232</td>
      <td>118368</td>
      <td>27.26%</td>
    </tr>
    <tr>
      <td>/usr/bin/testRTSPClient</td>
      <td>629400</td>
      <td>172656</td>
      <td>27.43%</td>
    </tr>
    <tr>
      <td>/usr/bin/rctest</td>
      <td>114208</td>
      <td>31600</td>
      <td>27.67%</td>
    </tr>
    <tr>
      <td>/usr/bin/lyxclient</td>
      <td>422744</td>
      <td>122760</td>
      <td>29.04%</td>
    </tr>
    <tr>
      <td>/usr/bin/testMPEG4VideoToDarwin</td>
      <td>624856</td>
      <td>196080</td>
      <td>31.38%</td>
    </tr>
    <tr>
      <td>/usr/bin/l2test</td>
      <td>83232</td>
      <td>26264</td>
      <td>31.56%</td>
    </tr>
    <tr>
      <td colspan="4">…</td>
    </tr>
    <tr>
      <td>/usr/sbin/pppgetpass.vt</td>
      <td>8280</td>
      <td>8472</td>
      <td>102.32%</td>
    </tr>
    <tr>
      <td>/usr/bin/ilbmtoppm</td>
      <td>34616</td>
      <td>35496</td>
      <td>102.54%</td>
    </tr>
    <tr>
      <td>/usr/bin/kross</td>
      <td>15624</td>
      <td>16024</td>
      <td>102.56%</td>
    </tr>
    <tr>
      <td>/usr/bin/kconfig_compiler</td>
      <td>118912</td>
      <td>122608</td>
      <td>103.11%</td>
    </tr>
    <tr>
      <td>/usr/bin/kmailservice</td>
      <td>8296</td>
      <td>8568</td>
      <td>103.28%</td>
    </tr>
    <tr>
      <td>/usr/bin/ktelnetservice</td>
      <td>15768</td>
      <td>16424</td>
      <td>104.16%</td>
    </tr>
    <tr>
      <td>/usr/sbin/ab</td>
      <td>46664</td>
      <td>48768</td>
      <td>104.51%</td>
    </tr>
    <tr>
      <td>/usr/bin/swig</td>
      <td>1431120</td>
      <td>1500392</td>
      <td>104.84%</td>
    </tr>
    <tr>
      <td>/usr/bin/okular</td>
      <td>66184</td>
      <td>70280</td>
      <td>106.19%</td>
    </tr>
    <tr>
      <td>/usr/bin/meinproc4</td>
      <td>37544</td>
      <td>40608</td>
      <td>108.16%</td>
    </tr>
    <tr>
      <td>/usr/bin/gif2tiff</td>
      <td>13544</td>
      <td>15040</td>
      <td>111.05%</td>
    </tr>
    <tr>
      <td>/usr/bin/kcachegrind</td>
      <td>890184</td>
      <td>1023024</td>
      <td>114.92%</td>
    </tr>
    <tr>
      <td>/usr/bin/htop</td>
      <td>124904</td>
      <td>152296</td>
      <td>121.93%</td>
    </tr>
    <tr>
      <td>/usr/bin/node</td>
      <td>6916664</td>
      <td>8764904</td>
      <td>126.72%</td>
    </tr>
    <tr class="strong">
      <td><strong>SUMMARY (of 1688 binaries)</strong></td>
      <td><strong>387.0 MB</strong></td>
      <td><strong>346.1 MB</strong></td>
      <td><strong>89.43%</strong></td>
    </tr>
  </tbody>
</table>

Shared libraries
---

<table class="table table-stripped table-bordered table-condensed binaries-report">
  <thead>
    <tr>
      <th>Shared library name</strong></th>
      <th class="tr">no-LTO binary size</th>
      <th class="tr">LTO size</th>
      <th class="tr">Fraction</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>/usr/lib64/libsandbox.so</td>
      <td>72120</td>
      <td>16064</td>
      <td>22.27%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libboost_wave.so.1.52.0</td>
      <td>1196640</td>
      <td>632872</td>
      <td>52.89%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libboost_program_options.so.1.52.0</td>
      <td>496008</td>
      <td>270224</td>
      <td>54.48%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libgirepository-1.0.so.1.0.0</td>
      <td>206424</td>
      <td>120272</td>
      <td>58.26%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libboost_wserialization.so.1.52.0</td>
      <td>334168</td>
      <td>201520</td>
      <td>60.30%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libboost_thread.so.1.52.0</td>
      <td>176000</td>
      <td>108016</td>
      <td>61.37%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libgnutls-extra.so.26.22.6</td>
      <td>34136</td>
      <td>21248</td>
      <td>62.25%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libsbc.so.1.1.0</td>
      <td>64016</td>
      <td>40856</td>
      <td>63.82%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libboost_unit_test_framework.so.1.52.0</td>
      <td>702112</td>
      <td>453160</td>
      <td>64.54%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libboost_serialization.so.1.52.0</td>
      <td>462904</td>
      <td>299592</td>
      <td>64.72%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libicule.so.51.2</td>
      <td>524104</td>
      <td>346992</td>
      <td>66.21%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libsmbclient.so.0</td>
      <td>5904400</td>
      <td>3934144</td>
      <td>66.63%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libboost_locale.so.1.52.0</td>
      <td>981776</td>
      <td>672160</td>
      <td>68.46%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libwbclient.so.0</td>
      <td>67888</td>
      <td>47104</td>
      <td>69.38%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libnetapi.so.0</td>
      <td>6592792</td>
      <td>4662200</td>
      <td>70.72%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libboost_date_time.so.1.52.0</td>
      <td>71552</td>
      <td>50872</td>
      <td>71.10%</td>
    </tr>
    <tr>
      <td>/lib64/libblkid.so.1.1.0</td>
      <td>206568</td>
      <td>147336</td>
      <td>71.33%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libboost_graph.so.1.52.0</td>
      <td>363624</td>
      <td>260904</td>
      <td>71.75%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libinproctrace.so</td>
      <td>38752</td>
      <td>28328</td>
      <td>73.10%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libiculx.so.51.2</td>
      <td>62224</td>
      <td>45872</td>
      <td>73.72%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libattica.so.0.4.2</td>
      <td>1020296</td>
      <td>753632</td>
      <td>73.86%</td>
    </tr>
    <tr>
      <td>/lib64/libudev.so.1.4.0</td>
      <td>79832</td>
      <td>59184</td>
      <td>74.14%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libGLU.so.1.3.1</td>
      <td>526216</td>
      <td>390768</td>
      <td>74.26%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libboost_iostreams.so.1.52.0</td>
      <td>106728</td>
      <td>79656</td>
      <td>74.63%</td>
    </tr>
    <tr>
      <td>/lib64/libaio.so.1.0.1</td>
      <td>3984</td>
      <td>2992</td>
      <td>75.10%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libpipeline.so.1.2.5</td>
      <td>52368</td>
      <td>39824</td>
      <td>76.05%</td>
    </tr>
    <tr>
      <td>/lib64/libmount.so.1.1.0</td>
      <td>222824</td>
      <td>169824</td>
      <td>76.21%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libdbusmenu-qt.so.2.6.0</td>
      <td>200184</td>
      <td>155520</td>
      <td>77.69%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libboost_signals.so.1.52.0</td>
      <td>95000</td>
      <td>73824</td>
      <td>77.71%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libstreamanalyzer.so.0.7.8</td>
      <td>527632</td>
      <td>413168</td>
      <td>78.31%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libboost_math_tr1l.so.1.52.0</td>
      <td>250968</td>
      <td>199000</td>
      <td>79.29%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libmp4v2.so.2.0.0</td>
      <td>975072</td>
      <td>777440</td>
      <td>79.73%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libboost_python-2.7.so.1.52.0</td>
      <td>316264</td>
      <td>253360</td>
      <td>80.11%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libboost_math_tr1.so.1.52.0</td>
      <td>250264</td>
      <td>200488</td>
      <td>80.11%</td>
    </tr>
    <tr>
      <td colspan="4">…</td>
    </tr>
    <tr>
      <td>/usr/lib64/libknewstuff2.so.4.11.5</td>
      <td>408864</td>
      <td>434296</td>
      <td>106.22%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libmdigest.so.1.0</td>
      <td>37248</td>
      <td>39624</td>
      <td>106.38%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libkmediaplayer.so.4.11.5</td>
      <td>38200</td>
      <td>40784</td>
      <td>106.76%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libknotifyconfig.so.4.11.5</td>
      <td>73576</td>
      <td>79240</td>
      <td>107.70%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libkfile.so.4.11.5</td>
      <td>707824</td>
      <td>762848</td>
      <td>107.77%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libkdeinit4_kio_http_cache_cleaner.so</td>
      <td>48776</td>
      <td>52952</td>
      <td>108.56%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libthreadweaver.so.4.11.5</td>
      <td>94056</td>
      <td>102576</td>
      <td>109.06%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libkutils.so.4.11.5</td>
      <td>6920</td>
      <td>7592</td>
      <td>109.71%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libkhtml.so.5.11.5</td>
      <td>8475712</td>
      <td>9339944</td>
      <td>110.20%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libkimproxy.so.4.11.5</td>
      <td>70648</td>
      <td>78448</td>
      <td>111.04%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libkjs.so.4.11.5</td>
      <td>880976</td>
      <td>985072</td>
      <td>111.82%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libkdeinit4_kded4.so</td>
      <td>71072</td>
      <td>79888</td>
      <td>112.40%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libkdeclarative.so.5.11.5</td>
      <td>61688</td>
      <td>70272</td>
      <td>113.92%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libkdeinit4_klauncher.so</td>
      <td>87024</td>
      <td>100000</td>
      <td>114.91%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libsolid.so.4.11.5</td>
      <td>1054680</td>
      <td>1249792</td>
      <td>118.50%</td>
    </tr>
    <tr>
      <td>/usr/lib64/libkidletime.so.4.11.5</td>
      <td>62584</td>
      <td>75256</td>
      <td>120.25%</td>
    </tr>
    <tr class="strong">
      <td><strong>SUMMARY (of 575 libraries)</strong></td>
      <td><strong>299.5 MB</strong></td>
      <td><strong>289.7 MB</strong></td>
      <td><strong>96.72%</strong></td>
    </tr>
  </tbody>
</table>

Final thoughts
---

Main intention for writing this post is to popularize Link-Time optimization in GCC. Presented results definitely proof that LTO significantly reduces binary size and majority of Gentoo packages can be built with LTO. Even though I do not benchmark LTO against classical build system, performance speed-up can be seen e.g. in my [diploma thesis](http://arxiv.org/abs/1403.6997). Another, more than interesting, source is [blog](http://hubicka.blogspot.cz/) of Jan Hubička who has been intensively writing about Link-Time optimization. I would really welcome any help, not just, from Gentoo community to decrease the number of Open-source software that cannot be built with LTO. Feel free to add packages that I missed.
