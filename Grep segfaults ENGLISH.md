https://blog.loadzero.com/blog/tracking-down-a-segfault-in-grep/


Tracking down a segfault in grep

Feb 18, 2015 - Jason McSweeney

I was happily tooling around on my macbook at the command line, poking around in the MAME source code as you do, and then this happened:

$ grep -f pats listing
704 ./powerpc
724 ./m68000
872 ./i386
1092    ./upd7810
Segmentation fault: 11

Record scratching sound. WTF.

grep just segfaulted. grep.

I use grep hundreds of times a day, going back over 15 years. I have used it in countless scripts and shovelled many terabytes of data through it. It has never failed on me before. It is one of the quintessential reliable unix tools. To say I was surprised at its failure is an understatement.

After recovering from my shock, I felt a pressing need to find out why it failed.

First things first - which grep am I using?

$ file $(which grep)
/usr/bin/grep: Mach-O 64-bit executable x86_64

$ type grep
grep is aliased to `grep --color '

$ grep -V
grep (BSD grep) 2.5.1-FreeBSD

Huh. A BSD grep. Interesting. It is also worth noting that I turn color on by default because I like to see my patterns highlighted.

I did a bit of playing around to see if I could create a smaller test case:

$ grep -f <(tail -n50 pats) <(tail -n50 listing)
</snip>
normal output,
then a bunch of line noise
<snip>

Segmentation fault: 11

After a few back and forths of bisecting the input, I pared it down to a fairly minimal case that consisted of four patterns and one input line:

$ xxd pats.sml

0000000: 2e2f 6938 3630 0a2e 2f73 6861 7263 0a2e  ./i860../sharc..
0000010: 2f67 3635 3831 360a 2e2f 6938 360a       /g65816../i86.

$ xxd listing.sml

0000000: 3139 3209 2e2f 6938 3630 0a              192../i860.

$ grep -f pats.sml listing.sml

192 ./i860
Segmentation fault: 11

So, does the debugger offer any help? It probably won't since this is a stripped binary, but lets give it a whirl:


$ lldb -- grep  -f pats.sml listing.sml 
Current executable set to 'grep' (x86_64).
(lldb) r
Process 8906 launched: '/usr/bin/grep' (x86_64)
192 ./i860
Process 8906 exited with status = 0 (0x00000000) 
(lldb) q

Hmm, no crash. After a head scratch, I realise the debugger is not invoking with --color.

Lets try again:

$ lldb -- grep --color -f pats.sml listing.sml 
Current executable set to 'grep' (x86_64).
(lldb) r
Process 8892 launched: '/usr/bin/grep' (x86_64)
192 ./i860
Process 8892 stopped
* thread #1: tid = 0x14d28, 0x00007fff8cd9dc64 libsystem_platform.dylib`_platform_memchr + 84, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=1, address=0x101000000)
    frame #0: 0x00007fff8cd9dc64 libsystem_platform.dylib`_platform_memchr + 84
libsystem_platform.dylib`_platform_memchr + 84:
-> 0x7fff8cd9dc64:  movdqa (%rdi), %xmm1
   0x7fff8cd9dc68:  pcmpeqb %xmm0, %xmm1
   0x7fff8cd9dc6c:  pmovmskb %xmm1, %eax
   0x7fff8cd9dc70:  testl  %eax, %eax
(lldb) bt
* thread #1: tid = 0x14d28, 0x00007fff8cd9dc64 libsystem_platform.dylib`_platform_memchr + 84, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=1, address=0x101000000)
  * frame #0: 0x00007fff8cd9dc64 libsystem_platform.dylib`_platform_memchr + 84
    frame #1: 0x00007fff85789be0 libsystem_c.dylib`__sfvwrite + 272
    frame #2: 0x00007fff8578a102 libsystem_c.dylib`fwrite + 136
    frame #3: 0x0000000100003517 grep`___lldb_unnamed_function21$$grep + 313
    frame #4: 0x00000001000030bf grep`___lldb_unnamed_function16$$grep + 1707
    frame #5: 0x0000000100002039 grep`___lldb_unnamed_function5$$grep + 2781
(lldb) 

Huh, yep - thats a segfault alright. And the bug is definitely related to --color. At this stage it is hard to figure out what is going on, and all this shows is that a bogus pointer is eventually handed to fwrite. The crash is probably some way off from the original bug. Joy.

Lets make our life a bit easier and see if we can track down the source code at apple.

AFAICT, there doesn't seem to be a nice way to match an apple binary to the source code on their website.

Does the binary tell us anything?

$ strings $(which grep) |tail
<snip/>
@(#)PROGRAM:grep  PROJECT:bsdgrep-16

After a bunch of digging around on Apple's website, I came up empty handed.

Time to swim upstream to FreeBSD.

According to FreeBSD, the current stable is 10.1. To save myself some time, I looked around for a suitable vagrant VM image. Thankfully, wunki has done all the hard work:

$ wget https://github.com/wunki/vagrant-freebsd/blob/master/Vagrantfile
$ vagrant up
<snip>lots of downloading and configuring</snip>

Lets test it out:

$ vagrant ssh
[vagrant@vagrant]$ grep --color -f /vagrant/pats /vagrant/listing
<snip>a working grep - hmm.</snip>

[vagrant@vagrant]$ grep --version
grep (GNU grep) 2.5.1-FreeBSD

Well that is interesting, they ship GNU grep as the system grep on FreeBSD and it works just fine.

Any other greps on the system?

[vagrant@vagrant]$ bsdgrep --version
bsdgrep (BSD grep) 2.5.1-FreeBSD

Ah, that looks closer to our mac version. How well does it work?

[vagrant@vagrant]$ bsdgrep --color -f /vagrant/pats /vagrant/listing
[vagrant@vagrant]$
[vagrant@vagrant]$ echo $?
1

Hmm, no output, and an exit code of 1.

That indicates that no match, which is incorrect. So, no crash - but not working properly either.

Maybe the distributed version is weird. Lets pull down a version from ports:

[vagrant@vagrant]$ su -

[root@vagrant]# portsnap fetch
<snip/>

[root@vagrant]# portsnap extract
<snip/>

[root@vagrant]# portsnap fetch update
<snip/>

[vagrant@vagrant]# cd /usr/ports/textproc/bsdgrep
[vagrant@vagrant]# make
<snip> A lot of compiling </snip>

[vagrant@vagrant]# ./work/stage/usr/local/bin/grep --version
grep (BSD grep) 2.5.1-FreeBSD

Cool, lets try blow it up:


[root@vagrant]# ./work/stage/usr/local/bin/grep --color -f /vagrant/pats /vagrant/listing

Segmentation fault (core dumped)

Great! - how about a debug build?

[root@vagrant]# make clean
[root@vagrant]# make WITH_DEBUG=yes
<snip/>

[root@vagrant]# gdb --args ./work/stage/usr/local/bin/grep --color -f /vagrant/pats /vagrant/listing
GNU gdb 6.1.1 [FreeBSD]
<snip>copyright</snip>
(gdb) r
Starting program: /usr/ports/textproc/bsdgrep/work/stage/usr/local/bin/grep --color -f /vagrant/pats /vagrant/listing

Program received signal SIGSEGV, Segmentation fault.
0x000000000040d893 in fastcmp (fg=0x801c500f0, data=0x80226d000, type=STR_BYTE) at /usr/ports/textproc/bsdgrep/work/grep-20111002/regex/tre-fastmatch.c:1031
1031          if (fg->icase ? (tolower(pat_byte[i]) == tolower(str_byte[i]))
Current language:  auto; currently minimal
(gdb) bt
#0  0x000000000040d893 in fastcmp (fg=0x801c500f0, data=0x80226d000, type=STR_BYTE) at /usr/ports/textproc/bsdgrep/work/grep-20111002/regex/tre-fastmatch.c:1031
#1  0x000000000040cc5b in tre_match_fast (fg=0x801c500f0, data=0x80226d000, len=9, type=STR_BYTE, nmatch=1, pmatch=0x7fffffffe778, eflags=4)
    at /usr/ports/textproc/bsdgrep/work/grep-20111002/regex/tre-fastmatch.c:940
#2  0x0000000000406afd in tre_fastnexec (preg=0x801c500f0, 
    string=0x801c6d000 "12\t./mb86235\n16\t./arc\n20\t./ie15\n20\t./ssem\n24\t./8x300\n24\t./ucom4\n28\t./i4004\n32\t./ccpu\n32\t./i8008\n32\t./scmp\n32\t./unsp\n36\t./amis2000\n36\t./ssp1601\n40\t./apexc\n40\t./mb88xx\n40\t./upd7725\n44\t./scudsp\n48\t./es55"..., len=18446744073709551615, nmatch=1, pmatch=0x7fffffffe778, eflags=4)
    at /usr/ports/textproc/bsdgrep/work/grep-20111002/regex/fastmatch.c:135
#3  0x0000000000406bf2 in tre_fastexec (preg=0x801c500f0, 
    string=0x801c6d000 "12\t./mb86235\n16\t./arc\n20\t./ie15\n20\t./ssem\n24\t./8x300\n24\t./ucom4\n28\t./i4004\n32\t./ccpu\n32\t./i8008\n32\t./scmp\n32\t./unsp\n36\t./amis2000\n36\t./ssp1601\n40\t./apexc\n40\t./mb88xx\n40\t./upd7725\n44\t./scudsp\n48\t./es55"..., nmatch=1, pmatch=0x7fffffffe778, eflags=4)
    at /usr/ports/textproc/bsdgrep/work/grep-20111002/regex/fastmatch.c:146
#4  0x00000000004057e7 in procline (l=0x7fffffffe8e8, nottext=0) at util.c:287
#5  0x0000000000405455 in procfile (fn=0x7fffffffedb3 "/vagrant/listing") at util.c:231
#6  0x0000000000404152 in main (argc=5, argv=0x7fffffffeb00) at grep.c:719

Sweet - we are starting to home in on the problem

Lets check the behaviour without color:

[vagrant@vagrant]# ./work/stage/usr/local/bin/grep -f /vagrant/pats /vagrant/listing
<snip> working </snip>

[vagrant@vagrant]# echo $?
0

This version of grep is showing the same behaviour as the apple version, and we have a nice debuggable binary.

What next? Well, it's time to get up close and personal with the source code, and fire up valgrind1.

At this point, I consider that I may need to do a port over to linux, since that is valgrind's native platform.

Lets try the FreeBSD valgrind anyway:

[vagrant@vagrant]# valgrind --db-attach=yes --track-origins=yes ./work/grep-20111002/grep --color -f /vagrant/pats /vagrant/listing
==54317== Memcheck, a memory error detector
==54317== Copyright (C) 2002-2012, and GNU GPL'd, by Julian Seward et al.
==54317== Using Valgrind-3.8.1 and LibVEX; rerun with -h for copyright info
==54317== Command: ./work/grep-20111002/grep --color -f /vagrant/pats /vagrant/listing
==54317== 
==54317== Conditional jump or move depends on uninitialised value(s)
==54317==    at 0x40D899: fastcmp (tre-fastmatch.c:1031)
==54317==    by 0x40CC5A: tre_match_fast (tre-fastmatch.c:940)
==54317==    by 0x406AFC: tre_fastnexec (fastmatch.c:135)
==54317==    by 0x406BF1: tre_fastexec (fastmatch.c:146)
==54317==    by 0x4057E6: procline (util.c:287)
==54317==    by 0x405454: procfile (util.c:231)
==54317==    by 0x404151: main (grep.c:719)
==54317==  Uninitialised value was created by a heap allocation
==54317==    at 0x10152B3: malloc (in /usr/local/lib/valgrind/vgpreload_memcheck-amd64-freebsd.so)
==54317==    by 0x4056B4: grep_malloc (util.c:390)
==54317==    by 0x402CBA: grep_open (file.c:268)
==54317==    by 0x405216: procfile (util.c:193)
==54317==    by 0x404151: main (grep.c:719)
==54317== 
==54317== 
==54317== ---- Attach to debugger ? --- [Return/N/n/Y/y/C/c] ---- y

valgrind: m_debugger.c:241 (Int ptrace_setregs(Int, VexGuestArchState *)): Assertion 'Unimplemented functionality' failed.
valgrind: valgrind

<snip> valgrind stacking it </snip>

Well - it got halfway there - and it did give us a bit more insight into the problem.

The benefit of valgrind is that it lets you stop closer to the original error - in this case it looks like a read past the end of a buffer.

Ideally, if you understand the source code well enough, you can prod around with gdb at this point and inspect things.

Unfortunately, this appears to not work in FreeBSD.

Thankfully, we can just do a quick and dirty port to linux, and go from there.

[jason@ubuntu]$ bmake clean
[jason@ubuntu]$ bmake

<snip> 
Compile Errors 
</snip>

Looks like the main problems are related to

    __FBSDID macro
    missing mmap stuff
    OFF_MAX
    fgetln

After fixing that, bmake was a little finicky about building a debug version, so I had to hack up a build script.

[jason@ubuntu]$ bmake clean
[jason@ubuntu]$ bmake grep > build.log
[jason@ubuntu]$ cat build.log  |sed 's/-O2//g' |sed 's/cc/cc -g /g' > build.sh
[jason@ubuntu]$ chmod u+x build.sh
[jason@ubuntu]$ ./build.sh
[jason@ubuntu]$ file grep
grep: ELF 64-bit LSB  executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=88e9f08cff89f00ad7f29339fdd63f7d3b45cc50, not stripped

What does the linux version have for us?


[jason@ubuntu]$ valgrind --db-attach=yes --track-origins=yes ./grep --color -f /tmp/pats.sml /tmp/listing.sml
==17763== Memcheck, a memory error detector
==17763== Copyright (C) 2002-2013, and GNU GPL'd, by Julian Seward et al.
==17763== Using Valgrind-3.10.0.SVN and LibVEX; rerun with -h for copyright info
==17763== Command: ./grep --color -f /tmp/pats.sml /tmp/listing.sml
==17763== 
==17763== Conditional jump or move depends on uninitialised value(s)
==17763==    at 0x40BB09: fastcmp (tre-fastmatch.c:1031)
==17763==    by 0x40B084: tre_match_fast (tre-fastmatch.c:940)
==17763==    by 0x405734: tre_fastnexec (fastmatch.c:135)
==17763==    by 0x40582F: tre_fastexec (fastmatch.c:146)
==17763==    by 0x404A78: procline (util.c:286)
==17763==    by 0x40478B: procfile (util.c:230)
==17763==    by 0x403ECD: main (grep.c:743)
==17763==  Uninitialised value was created by a heap allocation
==17763==    at 0x4C2AB80: malloc (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==17763==    by 0x404F77: grep_malloc (util.c:389)
==17763==    by 0x4027E2: grep_open (file.c:267)
==17763==    by 0x40457D: procfile (util.c:192)
==17763==    by 0x403ECD: main (grep.c:743)
==17763== 
==17763== 
==17763== ---- Attach to debugger ? --- [Return/N/n/Y/y/C/c] ---- y

<snip> debugger gumf </snip>
0x000000000040bb09 in fastcmp (fg=0x5a4e9a8, data=0x5a51810, type=STR_MBS)
    at regex/tre-fastmatch.c:1031
1031          if (fg->icase ? (tolower(pat_byte[i]) == tolower(str_byte[i]))
(gdb) 

Success - we have replicated the same bug under linux too. We can start debugging in earnest. It is complaining about line 1031 - we are either reading uninitialised data from either pat_byte or str_byte:


(gdb) p i
$10 = 4
(gdb) p pat_byte[i]
$11 = 54 '6'
(gdb) p str_byte[i]
$12 = 0 '\000'

(gdb) x/5c pat_byte 
0x5a510d0:  46 '.'  47 '/'  105 'i' 56 '8'  54 '6'
(gdb) x/5c str_byte 
0x5a51810:  0 '\000'    0 '\000'    0 '\000'    0 '\000'    0 '\000'

(gdb) p str_byte + i
$16 = 0x5a51814 ""

str_byte looks more suss than pat_byte. Why?

(gdb) bt
$10 = 4
#0  0x000000000040bb09 in fastcmp (fg=0x5a4e9a8, data=0x5a51810, type=STR_MBS)
    at /home/jason/proto/bsdgrep/usr/ports/textproc/bsdgrep/work/grep-20111002/regex/tre-fastmatch.c:1031
#1  0x000000000040b085 in tre_match_fast (fg=0x5a4e9a8, data=0x5a51810, len=6, type=STR_MBS, nmatch=1, pmatch=0xfff000190, eflags=4)
    at /home/jason/proto/bsdgrep/usr/ports/textproc/bsdgrep/work/grep-20111002/regex/tre-fastmatch.c:940
#2  0x0000000000405735 in tre_fastnexec (preg=0x5a4e9a8, string=0x5a51800 "192\t./i860\n", len=18446744073709551615, nmatch=1, pmatch=0xfff000190, 
    eflags=4) at /home/jason/proto/bsdgrep/usr/ports/textproc/bsdgrep/work/grep-20111002/regex/fastmatch.c:135
#3  0x0000000000405830 in tre_fastexec (preg=0x5a4e9a8, string=0x5a51800 "192\t./i860\n", nmatch=1, pmatch=0xfff000190, eflags=4)
    at /home/jason/proto/bsdgrep/usr/ports/textproc/bsdgrep/work/grep-20111002/regex/fastmatch.c:146
#4  0x0000000000404a79 in procline (l=0xfff0002e0, nottext=0) at util.c:286
#5  0x000000000040478c in procfile (fn=0xfff000787 "/tmp/listing.sml") at util.c:230
#6  0x0000000000403ece in main (argc=5, argv=0xfff000528) at grep.c:743

(gdb) up 4

#4  0x0000000000404a79 in procline (l=0xfff0002e0, nottext=0) at util.c:286
286                 r = fastexec(&fg_pattern[i],

(gdb) list

281         pmatch.rm_eo = l->len;
282 
283         /* Loop to compare with all the patterns */
284         for (i = 0; i < patterns; i++) {
285             if (fg_pattern[i].pattern)
286                 r = fastexec(&fg_pattern[i],
287                     l->dat, 1, &pmatch, eflags);
288             else
289                 r = regexec(&r_pattern[i], l->dat, 1,
290                     &pmatch, eflags);

(gdb) p l->dat
$19 = 0x5a51800 "192\t./i860\n"

(gdb) p l->len
$20 = 10

(gdb) p (char*) 0x5a51814 - (l->dat)
$21 = 20

According to this, fastexec is operating on l->dat, which is a pointer to a line read from the listing file. The program thinks it has a length of 10, which matches the file, and the string

"192\t./i860\n"

which is 10 characters (without the newline)

So why is fastcmp wandering around at offset 20 ? That smells like an error, considering the listing file is smaller than 20 bytes.

It looks like the invariant here should be that the valid range for fastexec to access is

l->dat, l->dat + l->len

After a quick dig through the code, I determined that access to the string was being controlled by the offsets pmatch.rm_so and pmatch.rm_eo, and added a bit of debug output to clarify.

In the non color case:

offsets 0 10

with color:

offsets 0 10
offsets 4 10
offsets 8 14
offsets 16 22

Bingo, the offsets are being incremented past the end of the buffer. This incrementing occurs inside tre_fastnexec inside this macro:

CALL_WITH_OFFSET(tre_match_fast(preg, &string[offset], slen, type, 
                                nmatch, pmatch, eflags));

It is not clear whether the callee or the caller is in error here, as the intent of the code isn't straightforward.

At this point I had collected enough data to send a bug report - 197531 - to FreeBSD.

Something was still bugging me though - why was the behaviour of the distributed bsdgrep binary different?

I decided to do some more digging - to be continued in part 2.


Digging deeper into the grep segfault

Feb 24, 2015 - Jason McSweeney

In the last article, I discussed tracking down a segfault in grep. This article continues down the rabbit hole.

At the end of the last instalment, I was left with a puzzle. Why was the version of grep from textproc/bsdgrep acting in a different manner to /usr/bin/bsdgrep? Time to put on my debugging cap, and dig down a little further.

Let's start off with trying a few variants of the input set:

$ printf "192\t./i860\n"  | bsdgrep --color -e i860 -e sharc
192 ./i860
$ printf "192\t./i860\n"  | bsdgrep --color -e i860 -e g65816
192 ./i860
$ printf "192\t./i860\n"  | bsdgrep --color -e i860 -e i86
192 ./i860
192 ./i860
Segmentation fault (core dumped)

Excellent, we made it crash! Can we slim it down even more?

[vagrant]$ echo i860  | bsdgrep --color -e i860 -e i86
i860
i860
Segmentation fault (core dumped)

What about the behaviour on mac?

$ echo i860  | grep --color -e i860 -e i86
i860
Segmentation fault: 11

It was around this time that I realised I had been looking at the wrong binary. D'oh!

After actually reading TFM, I determined that FreeBSD builds its system binaries from SVN and not from ports. The binary I had been looking at previously was a different version!

A bit of poking around at freebsd.org and googling for grep.c revealed this as the likely repo path:

base/stable/10/usr.bin/grep/

Lets pull it down from svn:

[vagrant]$ svn checkout http://svn.freebsd.org/base/head --depth=immediates stable
<snip/>
[vagrant]$ svn up stable/usr.bin/ --set-depth=immediates
<snip/>
[vagrant]$ svn up stable/usr.bin/grep --set-depth=infinity
<snip/>
[vagrant]$ cd stable/usr.bin/grep
[vagrant]$ make
<snip/>compiling a non stripped release version</snip>

[vagrant]$ file ./bsdgrep
bsdgrep: ELF 64-bit LSB executable, x86-64, version 1 (FreeBSD), dynamically linked (uses shared libs), for FreeBSD 10.1, not stripped

[vagrant]$ objdump --syms bsdgrep |grep debug_info
[vagrant]$ 

Note - I use set depth with svn to avoid pulling down the whole FreeBSD source tree, which is quite large.

Lets check the behaviour:


[vagrant]$ ./bsdgrep --version
bsdgrep (BSD grep) 2.5.1-FreeBSD

[vagrant]$ echo i860 | bsdgrep --color -e i860 -e i86
i860
i860
Segmentation fault (core dumped)

So, what does the debugger have for us?

[vagrant]$ echo i860 > /tmp/l
[vagrant]$ gdb --args ./bsdgrep --color -e i860 -e i86 /tmp/l

(gdb) r
Starting program: /usr/home/vagrant/testsvn/stable/usr.bin/grep/bsdgrep --color -e i860 -e i86 /tmp/l
(no debugging symbols found)

Program received signal SIGSEGV, Segmentation fault.
0x00000008011cca60 in memchr () from /lib/libc.so.7
(gdb) bt
#0  0x00000008011cca60 in memchr () from /lib/libc.so.7
#1  0x00000008011cc599 in fwrite () from /lib/libc.so.7
#2  0x00000008011cc43d in fwrite () from /lib/libc.so.7
#3  0x0000000000404df3 in printline ()
#4  0x000000000040485a in procfile ()
#5  0x0000000000403935 in main ()
(gdb)

That looks very familiar indeed. In my last article, that was the backtrace I got on my macbook when I first started to inspect the bug.

For comparison purposes, here it is again:

* thread #1: tid = 0x14d28, 0x00007fff8cd9dc64 libsystem_platform.dylib`_platform_memchr + 84, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=1, address=0x101000000)
  * frame #0: 0x00007fff8cd9dc64 libsystem_platform.dylib`_platform_memchr + 84
    frame #1: 0x00007fff85789be0 libsystem_c.dylib`__sfvwrite + 272
    frame #2: 0x00007fff8578a102 libsystem_c.dylib`fwrite + 136
    frame #3: 0x0000000100003517 grep`___lldb_unnamed_function21$$grep + 313
    frame #4: 0x00000001000030bf grep`___lldb_unnamed_function16$$grep + 1707
    frame #5: 0x0000000100002039 grep`___lldb_unnamed_function5$$grep + 2781
(lldb) 

Lets help ourselves out by making a debug build:

[vagrant]$ cd ../../
[vagrant]$ svn up share/mk --set-depth=infinity
<snip/>
[vagrant]$ cp share/mk/sys.mk share/mk/sys.mk.orig
<snip> hack build flags </snip>

[vagrant]$ diff share/mk/sys.mk.orig share/mk/sys.mk
61c61
< CFLAGS        ?=  -O2 -pipe
---
> CFLAGS        ?=  -g -pipe

[vagrant]$ cd usr.bin/grep
[vagrant]$ export MAKESYSPATH=../../share/mk
[vagrant]$ make clean
[vagrant]$ make
</snip>

[vagrant]$ objdump --syms bsdgrep |grep debug
0000000000000000 l    d  .debug_info    0000000000000000              .debug_info

What does valgrind have for us?

[vagrant]$ valgrind --db-attach=yes --track-origins=yes ./bsdgrep --color -e i860 -e i86 /tmp/l
==68354== Memcheck, a memory error detector
==68354== Copyright (C) 2002-2012, and GNU GPL'd, by Julian Seward et al.
==68354== Using Valgrind-3.8.1 and LibVEX; rerun with -h for copyright info
==68354== Command: ./bsdgrep --color -e i860 -e i86 /tmp/l
==68354== 
i860
==68354== Conditional jump or move depends on uninitialised value(s)
==68354==    at 0x1020E54: memchr (in /usr/local/lib/valgrind/vgpreload_memcheck-amd64-freebsd.so)
==68354==    by 0x1BC8598: ??? (in /lib/libc.so.7)
==68354==    by 0x1BC843C: fwrite (in /lib/libc.so.7)
==68354==    by 0x4062FB: printline (util.c:469)
==68354==    by 0x405EEE: procline (util.c:366)
==68354==    by 0x405587: procfile (util.c:231)
==68354==    by 0x404291: main (grep.c:738)
==68354==  Uninitialised value was created by a heap allocation
==68354==    at 0x101D2B3: malloc (in /usr/local/lib/valgrind/vgpreload_memcheck-amd64-freebsd.so)
==68354==    by 0x405814: grep_malloc (util.c:390)
==68354==    by 0x402CBA: grep_open (file.c:285)
==68354==    by 0x40535A: procfile (util.c:194)
==68354==    by 0x404291: main (grep.c:738)
==68354== 
==68354== 
==68354== ---- Attach to debugger ? --- [Return/N/n/Y/y/C/c] ---- y

<snip> valgrind stacking it </snip>

As in the previous article, valgrind crashes when I try to attach the debugger. At least it gives up some some useful info before doing so. The badness starts in the printline function, in util.c on line 469.

With valgrind down for the count, I decide to soldier on with gdb alone:

[vagrant]$ gdb --args ./bsdgrep --color -e i860 -e i86 /tmp/l
<snip>copyright</snip>
(gdb) r
Starting program: /usr/home/vagrant/testsvn/stable/usr.bin/grep/bsdgrep --color -e i860 -e i86 /tmp/l
i860
i860

Program received signal SIGSEGV, Segmentation fault.
0x00000008011d1a60 in memchr () from /lib/libc.so.7
(gdb) bt
#0  0x00000008011d1a60 in memchr () from /lib/libc.so.7
#1  0x00000008011d1599 in fwrite () from /lib/libc.so.7
#2  0x00000008011d143d in fwrite () from /lib/libc.so.7
#3  0x00000000004062fc in printline (line=0x7fffffffe878, sep=58, matches=0x7fffffffe710, m=2) at util.c:469
#4  0x0000000000405eef in procline (l=0x7fffffffe878, nottext=0) at util.c:366
#5  0x0000000000405588 in procfile (fn=0x7fffffffed65 "/tmp/l") at util.c:231
#6  0x0000000000404292 in main (argc=7, argv=0x7fffffffeab0) at grep.c:738

(gdb) up 4
(gdb) list

464         putchar(sep);
465     /* --color and -o */
466     if ((oflag || color) && m > 0) {
467         for (i = 0; i < m; i++) {
468             if (!oflag)
469                 fwrite(line->dat + a, matches[i].rm_so - a, 1,
470                     stdout);
471             if (color) 
472                 fprintf(stdout, "\33[%sm\33[K", color);

(gdb) p line->dat 
$1 = 0x801c30000 "i860\n"

(gdb) p a
$2 = 4

(gdb) p matches[i].rm_so
$3 = 0

(gdb) p (matches[i].rm_so - a)
$4 = 18446744073709551612

Ouch - that would explain the segfault. A huge value is being passed as the size param to fwrite. Why?

(gdb) p sizeof(a)
$10 = 8

(gdb) p sizeof(matches[i].rm_so)
$11 = 4

(gdb) ptype matches[i].rm_so
type = int

(gdb) ptype a
type = long unsigned int

(gdb) print (unsigned long)(-4)
$5 = 18446744073709551612

I'm not going to pretend I know all the C conversion rules down to the letter, but it seems fairly obvious that we are ending up with a -4 in a place where it shouldn't be, and it is getting converted into an unsigned long in the process.

As in the previous article, this bug comes down to the offsets rm_so and rm_eo. These are offsets that point to the beginning and end of the matching parts of the line.

I read through the code a few times, and added some debug output to get an idea of what it was trying to do.

Here is the problematic loop inside util.c written in pythonic pseudo-code:

for match in matches:
    if not print_only_matching_part:
        print non_matching_part

    if print_in_color:
        print color_escape_sequence_start

    print matching_part

    if print_in_color:
        print color_escape_sequence_end

    start_of_next_non_matching_part = match.end_offset

    if print_only_matching_part:
        print end_line

This is the c version with some debugging output added to the relevant parts:

for (i = 0; i < m; i++) {

    if (!oflag)
    {
        fprintf(stderr, "%s:%d match %d fwrite rm_so %d rm_eo %d a %zu\n",
                __FILE__, __LINE__, i, matches[i].rm_so, matches[i].rm_eo, a);
        fwrite(line->dat + a, matches[i].rm_so - a, 1, stdout);
    }
    if (color)
        fprintf(stdout, "\33[%sm\33[K", color);

    fwrite(line->dat + matches[i].rm_so,
        matches[i].rm_eo - matches[i].rm_so, 1,
        stdout);

    if (color)
        fprintf(stdout, "\33[m\33[K");

    a = matches[i].rm_eo;

    fprintf(stderr, "%s:%d match %d a = %zu\n", __FILE__, __LINE__, i, a);

    if (oflag)
        putchar('\n');
}

After adding the debug output, I ran a few experiments with different patterns:

[vagrant]$ echo i860 | ./bsdgrep --color -e i -e 6
util.c:472 match 0 fwrite rm_so 0 rm_eo 1 a 0
util.c:487 match 0 a = 1
util.c:472 match 1 fwrite rm_so 2 rm_eo 3 a 1
util.c:487 match 1 a = 3
i860

[vagrant]$ echo i860 | ./bsdgrep --color -e i860 -e i86
util.c:472 match 0 fwrite rm_so 0 rm_eo 4 a 0
util.c:487 match 0 a = 4
util.c:472 match 1 fwrite rm_so 0 rm_eo 3 a 4
i860
i860
Segmentation fault (core dumped)

This shows us that the loop is choking when handed overlapping pattern matches. There must be an assumption here that matching parts of the line do not overlap. In this case, they do - and when the code tries to print out the non-matching part of the line, it ends up with a negative value for the size of that part.

At this point I verified that this bug was still happening in the head branch, and then sent a bug report off to FreeBSD - 197555.

The probable fix for this bug is to check whether the non-matching part of the line has already been printed out by a previous match. A fairly big assumption was violated however, so some more careful testing and debugging is warranted. I am going to leave that to the FreeBSD maintainers.

Apple can either poke around and fix their own version, or wait for a fix from upstream. I do wonder why they switched from GNU grep in the first place - possibly a licensing issue?

If you've made it this far, thanks for coming along on the ride. And remember, there are always more bugs lurking just under the surface.
