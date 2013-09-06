  How To Build a Minimal Linux System from Source Code
  Greg O'Keefe, gcokeefe@postoffice.utas.edu.au
  v0.8, September 2000

  These are instructions for building a minimal linux system from source
  code.  It used to be part of From PowerUp to Bash Prompt
  <http://www.linuxdoc.org/HOWTO/From-PowerUp-To-Bash-Prompt-HOWTO.html>
  but I've separated it to keep both documents short and focussed.  The
  system we build here is _v_e_r_y minimal, and not ready for real work. If
  you want to build a practical system from scratch, see Gerard Beek-
  mans' Linux From Scratch HOWTO <http://www.linuxfromscratch.org>
  instead.
  ______________________________________________________________________

  Table of Contents


  1. What You Will Need

  2. The Filesystem

  3. MAKEDEV

  4. Kernel

  5. Lilo

  6. Glibc

  7. SysVinit

  8. Ncurses

  9. Bash

  10. Util-linux (getty and login)

  11. Sh-utils

  12. Towards Useability

  13. More Information

     13.1 Random Tips
     13.2 Links

  14. Administrivia

     14.1 Copyright
     14.2 Homepage
     14.3 Feedback
     14.4 Acknowledgements
     14.5 Change History
        14.5.1 0.8
     14.6 TODO


  ______________________________________________________________________

  11..  WWhhaatt YYoouu WWiillll NNeeeedd

  We will install a Linux distribution like Red Hat in one partition,
  and use that to build a new Linux system in another partition.  I will
  call the system we are building the ``target'' and the system we are
  using to build it with, the ``source'' (not to be confused with _s_o_u_r_c_e
  _c_o_d_e which we will also be using.)
  So you are going to need a machine with two spare partitions on it.
  If you can, use a machine with nothing important on it.  You could use
  an existing Linux installation as the source system, but I wouldn't
  recommend that. If you leave a parameter out of one of the commands we
  will issue, you could accidentally install stuff to this system. This
  could lead to incompatibilites and strife.


  Older PC hardware, mostly 486's and earlier, have an annoying
  limitation in their bios. They can not read from a hard disk past the
  first 512M.  This is not too much of a problem for Linux, because once
  it is up, it does its own disk io, bypassing the bios.  But for Linux
  to get loaded by these old machines, the kernel has to reside
  somewhere below 512M. If you have one of these machines you will need
  to have a separate partition completely below the 512M mark, to mount
  as /boot for any partitions that are over that 512M mark.


  Last time I did this, I used Red Hat 6.1 as a source system. I
  installed the base system plus


  +o  cpp

  +o  egcs

  +o  egcs-c++

  +o  patch

  +o  make

  +o  dev86

  +o  ncurses-devel

  +o  glibc-devel

  +o  kernel-headers


  I also had X-window and Mozilla so I could read documentation easily,
  but that's not really necessary.  By the time I had finished working,
  it had used about 350M of disk space. (Seems a bit high, I wonder
  why?)


  The finished target system took 650M, but that includes all the source
  code and intermediate build files. If space is tight, you should do a
  make clean after each package is built. Still, this mind boggling
  bloat is a bit of a worry.


  Finally, you are going to need the source code for the system we are
  going to build. These are the ``packages'' that I have discussed in
  this document. These can be obtained from a source cd, or from the
  internet. I'll give URL's for the USA sites and for Australian
  mirrors.




  +o  MAKEDEV USA <ftp://tsx-11.mit.edu/pub/linux/sources/sbin> Another
     USA <ftp://sunsite.unc.edu/pub/Linux/system/admin> site


  +o  Lilo USA <ftp://lrcftp.epfl.ch/pub/linux/local/lilo/>, Australia
     <ftp://mirror.aarnet.edu.au/pub/linux/metalab/system/boot/lilo/>.

  +o  Linux Kernel Use one of the mirrors listed at home page
     <http://www.kernel.org> rather than USA
     <ftp://ftp.kernel.org/pub/linux/kernel> because they are always
     overloaded.  Australia
     <ftp://kernel.mirror.aarnet.edu.au/pub/linux/kernel/>

  +o  GNU libc itself, and the linuxthreads addon are at USA
     <ftp://ftp.gnu.org/pub/gnu/glibc> Australia
     <ftp://mirror.aarnet.edu.au/pub/gnu/glibc>

  +o  GNU libc addons You will also need the linuxthreads and libcrypt
     addons.  If libcrypt is not there it is because of some US export
     laws.  You can get it at libcrypt
     <ftp://ftp.gwdg.de/pub/linux/glibc> The linuxthreads addon is in
     the same places as libc itself

  +o  GNU ncurses USA <ftp://ftp.gnu.org/gnu/ncurses> Australia
     <ftp://mirror.aarnet.edu.au/pub/gnu/ncurses>

  +o  SysVinit USA <ftp://sunsite.unc.edu/pub/Linux/system/daemons/init>
     Australia
     <ftp://mirror.aarnet.edu.au/pub/linux/metalab/system/daemons/init>

  +o  GNU Bash USA <ftp://ftp.gnu.org/gnu/bash> Australia
     <ftp://mirror.aarnet.edu.au/pub/gnu/bash>

  +o  GNU sh-utils USA <ftp://ftp.gnu.org/gnu/sh-utils> Australia
     <ftp://mirror.aarnet.edu.au/pub/gnu/sh-utils>

  +o  util-linux Somewhere else
     <ftp://ftp.win.tue.nl/pub/linux/utils/util-linux/> Australia
     <ftp://mirror.aarnet.edu.au/pub/linux/metalab/system/misc> This
     package contains agetty and login.


  To sum up then, you will need:

  +o  A machine with two spare partitions of about 400M and 700M
     respectively though you could probably get away with less

  +o  A Linux distribution (eg. a Red Hat cd) and a way of installing it
     (eg. a cdrom drive)

  +o  The source code tarballs listed above


  I'm assuming that you can install the source system yourself, without
  any help from me. From here on, I'll assume that its done.


  The first milestone in this little project is getting the kernel to
  boot up and panic because it can't find an init. This means we are
  going to have to install a kernel, and install lilo. To install lilo
  nicely though, we will need the device files in the target /dev
  directory. Lilo needs them to do the low level disk access necessary
  to write the boot sector.  MAKEDEV is the script that creates these
  device files.  (You can just copy them from the source system of
  course, but that's cheating!)  But first of all, we need a filesystem
  to put all of this into.




  22..  TThhee FFiilleessyysstteemm

  Our new system is going to live in a file system. So first, we have to
  make that file system using mke2fs. Then mount it somewhere. I'd
  suggest /mnt/target. In what follows, I'll assume that this is where
  it is.  You could save yourself a bit of time by putting an entry in
  /etc/fstab so that it mounts there automatically when the source
  system comes up.


  When we boot up the target system, the stuff that's now in /mnt/target
  will be in /.


  We need a directory structure on target. Have a look at the File
  Heirarchy Standard (see section ``Filesystem'') to work out what this
  should be, or just cd to where the target is mounted and blindly do



          mkdir bin boot dev etc home lib mnt root sbin tmp usr var
          cd var; mkdir lock log run spool
          cd ../usr; mkdir bin include lib local sbin share src
          cd share/; mkdir man; cd man
          mkdir man1 man2 man3 ... man9



  Since the FHS and most packages disagree about where man pages should
  go, we need a symlink



          cd ..; ln -s share/man man




  33..  MMAAKKEEDDEEVV

  We will put the source code in the target /usr/src directory.  So for
  example, if your target file system is mounted on /mnt/target and your
  tarballs are in /root, you would do



          cd /mnt/target/usr/src
          tar -xzvf /root/MAKEDEV-2.5.tar.gz




  Don't be completely lame and copy the tarball to the place where you
  are going to extract it ;->


  Normally when you install software, you are installing it onto the
  system that is running. We don't want to do that though, we want to
  install it as though /mnt/target is the root filesystem. Different
  packages have different ways of letting you do this. For MAKEDEV you
  do


          ROOT=/mnt/target make install


  You need to look out for these options in the README and INSTALL files
  or by doing a ./configure --help.


  Have a look in MAKEDEV's Makefile to see what it does with the ROOT
  varible that we set in that command. Then have a look in the man page
  by doing man ./MAKEDEV.man to see how it works. You'll find that the
  way to make our device files is to cd /mnt/target/dev and do ./MAKEDEV
  generic.  Do an ls to see all the wonderful device files it has made
  for you.


  44..  KKeerrnneell

  Next we make a kernel. I presume you've done this before, so I'll be
  brief.  It is easier to install lilo if the kernel it is meant to boot
  is already there. Go back to the target usr/src directory, and unpack
  the linux kernel source there. Enter the linux source tree (cd linux)
  and configure the kernel using your favourite method, for example make
  menuconfig.  You can make life slightly easier for yourself by
  configuring a kernel without modules. If you configure any modules,
  then you will have to edit the Makefile, find INSTALL_MOD_PATH and set
  it to /mnt/target.


  Now you can make dep, make bzImage, and if you configured modules:
  make modules, make modules_install. Copy the kernel
  arch/i386/boot/bzImage and the system map System.map to the target
  boot directory /mnt/target/boot, and we are ready to install lilo.


  55..  LLiilloo

  Lilo comes with a neat script called QuickInst. Unpack the lilo source
  into the target source directory, run this script with the command
  ROOT=/mnt/target ./QuickInst. It will ask you questions about how you
  want lilo installed.


  Remember, since we have set ROOT, to the target partition, you tell it
  file names relative to that. So when it asks what kernel you want to
  boot by default, answer /boot/bzImage _n_o_t /mnt/target/boot/bzImage.  I
  found a little bug in the script, so it said



          ./QuickInst: /boot/bzImage: no such file



  But if you just ignore it, it's ok.


  Where should we get QuickInst to put the boot sector?  When we reboot
  we want to have the choice of booting into the source system or the
  target system, or any other systems that are on this box.  And we want
  the instance of lilo that we are building now to load the kernel of
  our new system. How are we going achieve both of these things? Let's
  digress a little and look at how lilo boots DOS on a dual boot Linux
  system. The lilo.conf file on such a system probably looks something
  like this:





  prompt
  timeout = 50
  default = linux

  image = /boot/bzImage
          label  = linux
          root   = /dev/hda1
          read-only

  other = /dev/hda2
          label = dos





  If the machine is set up this way, then the master boot record gets
  read and loaded by the bios, and it loads the lilo bootloader, which
  gives a prompt.  If you type in dos at the prompt, lilo loads the boot
  sector from hda2, and it loads DOS.


  What we are going to do is just the same, except that the boot sector
  in hda2 is going to be another lilo boot sector - the one that
  QuickInst is going to install. So the lilo from the Linux distribution
  will load the lilo that we have built, and that will load the kernel
  that we have built.  You will see two lilo prompts when you reboot.


  To cut a long story short, when QuickInst asks you where to put the
  boot sector, tell it the device where your target filesystem is, eg.
  /dev/hda2.


  Now modify the lilo.conf on your source system, so it has a line like



  other = /dev/hda2
          label = target



  run lilo, and we should be able to do our first boot into the target
  system.


  66..  GGlliibbcc

  Next we want to install init, but like almost every program that runs
  under Linux, init uses library functions provided by the GNU C
  library, glibc. So we will install that first.


  Glibc is a very large and complicated package. It took 90 hours to
  build on my old 386sx/16 with 8M RAM. But it only took 33 minutes on
  my Celeron 433 with 64M. I think memory is the main issue here. If you
  only have 8M of RAM (or, shudder, less!) be prepared for a long build.


  The glibc install documentation recommends building in a separate
  directory.  This enables you to start again easily, by just blowing
  that directory away.  You might also want to do that to save yourself
  about 265M of disk space!


  Unpack the glibc-2.1.3.tar.gz (or whatever version) tarball into
  /mnt/target/usr/src as usual. Now, we need to unpack the ``add-ons''
  into glibc's directory. So cd glibc-2.1.3, and then unpack the glibc-
  crypt-2.1.3.tar.gz and glibc-linuxthreads-2.1.3.tar.gz tarballs there.


  Now we can create the build directory, configure, make and install
  glibc.  These are the commands I used, but read the documentation
  yourself and make sure you do what is best for your circumstances.
  Before you do though, you might want to do a df command to see how
  much free space you have. You can do another after you've built and
  installed glibc, to see what a space-hog it is.



          cd ..
          mkdir glibc-build
          ../glibc-2.1.3/configure --enable-add-ons --prefix=/usr
          make
          make install_root=/mnt/target install




  Notice that we have yet another way of telling a package where to
  install.


  77..  SSyyssVViinniitt

  Making and installing the SysVinit binaries is pretty straight
  forward.  I'll just be lazy and give you the commands, assuming that
  you have unpacked and entered the SysVinit source code directory:



   cd src
   make
   ROOT=/mnt/target make install




  There are also a lot of scripts associated with init.  There are
  example scripts with the SysVinit package, which work fine.  But you
  have to install them manually. They are set up in a heirarchy under
  debian/etc in the SysVinit source code tree. You can just copy them
  straight across into the target etc directory, with something like cd
  ../debian/etc; cp -r * /mnt/target/etc.  Obviously you will want to
  have a look before you copy them across!


  Everything is in place now for the target kernel to load up init when
  we reboot. The problem this time should be that the scripts won't run,
  becasue bash isn't there to interpret them. Also, init will try to run
  getty's, but there is no getty for it to run.  Reboot now and make
  sure there is nothing else wrong.


  88..  NNccuurrsseess

  The next thing we need is Bash, but bash needs ncurses, so we'll
  install it first. Ncurses replaces termcap as the way of handling text
  screens, but it can also provide backwards compatibility by supporting
  the termcap calls.  In the interests of having a clean simple modern
  system, I think its best to disable the old termcap method. You might
  strike trouble later on if you are compiling an older application that
  uses termcap.  But at least you will know what is using what. If you
  need to you can recompile ncurses with termcap support.


  The commands I used are



          ./configure --prefix=/usr --with-install-prefix=/mnt/target --with-shared --disable-termcap
          make
          make install




  99..  BBaasshh

  It me took quite a lot of reading and thinking and trial and error to
  get Bash to install itself where I thought it should go. The
  configuration options I used are



   ./configure --prefix=/mnt/target/usr/local --exec-prefix=/mnt/target --with-curses




  Once you have made and installed Bash, you need to make a symlink like
  this cd /mnt/target/bin; ln -s bash sh. This is because scripts
  usually have a first line like this



  #!/bin/sh




  If you don't have the symlink, your scripts won't be able to run,
  because they will be looking for /bin/sh not /bin/bash.


  You could reboot again at this point if you like. You should notice
  that the scripts actually run this time, though you still can't login,
  because there are no getty or login programs.


  1100..  UUttiill--lliinnuuxx ((ggeettttyy aanndd llooggiinn))

  The util-linux package contains agetty and login. We need both of
  these to be able to log in and get a bash prompt. After it is
  instlalled, make a symlink from agetty to getty in the target /sbin
  directory. getty is one of the programs that is supposed to be there
  on all Unix-like systems, so the link is a better idea than hacking
  inittab to run agetty.


  I have one remaining problem with the compilation of util-linux. The
  package also contains the program more, and I have not been able to
  persuade the make process to have more link against the ncurses 5
  library on the target system rather than the ncurses 4 on the source
  system. I'll be having a closer look at that.


  You will also need a /etc/passwd file on the target system.  This is
  where the login program will check to find out if you are allowed in.
  Since this is only a toy system at this stage, we can do outrageous
  things like setting up only the root user, and not requiring any
  password!! Just put this in the target /etc/passwd




  root::0:0:root:/root:/bin/bash




  The fields are separated by colons, and from left to right they are
  user id, password (encrypted), user number, group number, user's name,
  home directory and default shell.


  1111..  SShh--uuttiillss

  The last package we need is GNU sh-utils. The only program we need
  from here at this stage is stty, which is used in /etc/init.d/rc which
  is used to change runlevels, and to enter the initial runlevel.  I
  actually have, and used a package that contains only stty, but I can't
  remember where it came from. Its a better idea to use the GNU package,
  because there is other stuff in there that you will need if you add to
  the system to make it useable.


  Well that's it. You should now have a system that will boot up and
  prompt you for a login. Type in ``root'', and you should get a shell.
  You won't be able to do much with it. There isn't even an ls command
  here for you to see your handiwork. Press tab twice so you can see the
  available commands. This was about the most satisfying thing I found
  to do with it.


  1122..  TToowwaarrddss UUsseeaabbiilliittyy

  It might look like we have made a pretty useless system here. But
  really, there isn't that far to go before it can do some work. One of
  the first things you would have to do is have the root filesystem
  mount read-write.  There is a script from the SysVinit package, in
  /etc/init.d/mountall.sh which does this, and issues a mount -a so that
  everything gets mounted the way you specify in /etc/fstab. Put a
  symlink called something like S05mountall to it in the target's
  etc/rc2.d.


  You may find that this script will use commands that you haven't
  installed yet. If so, find the package that contains the commands and
  install it. See section ``Random Tips'' for clues on how to find
  packages.


  Look at the other scripts in /etc/init.d. Most of them will need to be
  included in any serious system. Add them in one at a time, make sure
  everthing is running smoothly before adding more.


  Check the File Heirarchy Standard (see section ``Filesystem'').  It
  has lists of the commands that should be in /bin and /sbin. Make sure
  that you have all these commands installed.  Even better, find the
  Posix documentation that specifies this stuff.

  From there, it's really just a matter of throwing in more and more
  packages until everything you want it there. The sooner you can put
  the build tools such as gcc and make in the better. Once that is done,
  you can use the target system to build itself, which is much less
  complicated.


  1133..  MMoorree IInnffoorrmmaattiioonn

  1133..11..  RRaannddoomm TTiippss

  If you have a command called thingy on a Linux system with RPM, and
  want a clue about where to get the source from, you can use the
  command:


          rpm -qif `which thingy`



  And if you have a Red Hat source CD, you can install the source code
  using


          rpm -i /mnt/cdrom/SRPMS/what.it.just.said-1.2.srpm




  This will put the tarball, and any Red Hat patches into
  /usr/src/redhat/SOURCES.


  1133..22..  LLiinnkkss


  +o  There is a mini-howto on building software from source, the
     Software Building mini-HOWTO
     <http://www.linuxdoc.org/HOWTO/Software-Building.html>.

  +o  There is also a HOWTO on building a Linux system from scratch.  It
     focuses much more on getting the system built so it can be used,
     rather than just doing it as a learning exercise.  The Linux From
     Scratch HOWTO <http://www.linuxfromscratch.org>

  +o   Unix File System Standard
     <ftp://tsx-11.mit.edu/pub/linux/docs/linux-standards/fsstnd/>
     Another link <http://www.pathname.com/fhs/> to the Unix File System
     Standard.  This describes what should go where in a Unix file
     system, and why. It also has minimum requirements for the contents
     of /bin, /sbin and so on. This is a good reference if your goal is
     to make a minimal yet complete system.


  1144..  AAddmmiinniissttrriivviiaa

  1144..11..  CCooppyyrriigghhtt

  This document is copyright (c) 1999, 2000 Greg O'Keefe. You are
  welcome to use, copy, distribute or modify it, without charge, under
  the terms of the GNU General Public Licence
  <http://www.gnu.org/copyleft/gpl.html>.  Please acknowledge me if you
  use all or part of this in another document.



  1144..22..  HHoommeeppaaggee

  The lastest version of this document lives at From Powerup To Bash
  Prompt <http://learning.taslug.org.au/power2bash>




  1144..33..  FFeeeeddbbaacckk

  I would like to hear any comments, criticisms and suggestions for
  improvement that you have. Please send them to me Greg O'Keefe
  <mailto:gcokeefe@postoffice.utas.edu.au>



  1144..44..  AAcckknnoowwlleeddggeemmeennttss

  Product names are trademarks of the respective holders, and are hereby
  considered properly acknowledged.


  There are some people I want to say thanks to, for helping to make
  this happen.



     MMiicchhaaeell EEmmeerryy
        For reminding me about Unios.

     TTiimm LLiittttllee
        For some good clues about /etc/passwd

     ssPPaaKKrr oonn ##lliinnuuxx iinn eeffnneett
        Who sussed out that syslogd needs /etc/services, and introduced
        me to the phrase ``rolling your own'' to describe building a
        system from source code.

     AAlleexx AAiittkkiinn
        For bringing Vico and his ``verum ipsum factum'' (understanding
        arises through making) to my attention.

     DDeennnniiss SSccootttt
        For correcting my hexidecimal arithmetic.

     jjdddd
        For pointing out some typos.


  1144..55..  CChhaannggee HHiissttoorryy

  1144..55..11..  00..88


  +o  Initial version. Separated from "From PowerUp to Bash Prompt".


  1144..66..  TTOODDOO


  +o  Convert to docbook.





