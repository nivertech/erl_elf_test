# Manually Creating an ELF Executable

by Irene Hoksbergen on Sep 8, 2014 


Hello class, and welcome to X86 Masochism 101. Here, you’ll learn how to use opcodes directly to create an executable without ever actually touching a compiler, assembler or linker. We’ll be using only an editor capable of modifying binary files (i.e. a hex editor) and ‘chmod’ to make the file executable.[1]

If that's not awesome, I don't know what is.

On a more serious note, this is one of those things that I personally think are a lot of fun. Obviously, you’re not going to be using this to create serious million-line programs. However, it can give you an enormous amount of satisfaction to know that you actually understand how this kind of thing really works on a low level. It’s also cool to be able to say you wrote an executable without ever touching a compiler or interpreter. Beyond that, there are applications in kernel programming, reverse engineering and (perhaps unsurprisingly) compiler creation.

First of all, let’s take a very quick look at how executing an ELF file actually works. A lot of details will be left out. What’s important is getting a good idea of what your PC does when you tell it to execute an ELF binary file.

When you tell the computer to execute an ELF binary, the first thing it’ll look for are the appropriate ELF headers. These headers contain all sorts of important information about CPU architecture, sections and segments of the file, and much more – we’ll talk more about that later. The header also contains information that helps the computer identify the file as ELF. Most importantly, the ELF header contains information about the program header table in the case of an executable, and the virtual address to which the computer transfers control upon execution.

The program header table, in turn, defines several segments in program headers. If you’ve ever programmed in assembly, you can think of some of the sections such as ‘text’ and ‘data’ as segments in an executable. The program headers also define where the data of these segments are in the actual file, and what virtual memory address to assign to them.

If everything’s been done correctly, the computer loads all segments into virtual memory based on the data in the program headers, then transfers control to the virtual memory address assigned in the ELF header, and starts executing instructions.

Before we begin with the practical stuff , please make sure you’ve got an actual hex editor on your computer, and that you can execute ELF binaries and are on an x86 machine. Most hex editors should work as long as they actually allow you to edit and save your work – I personally like Bless . If you’re on Linux, you should be fine as far as ELF binaries are concerned. Some other Unix-like operating systems might work, too, but different OSes implement things in slightly different ways, so I cannot be sure. I also use system calls extensively, which further limits compatibility. If you’re on Windows, you’re out of luck. Likewise if your CPU architecture is anything other than x86 (though x86_64 should work), since I simply cannot provide opcodes for each and every architecture out there.

There are three phases to creating an ELF executable. First , we’ll construct the actual payload using opcodes. Second , we’ll build the ELF and program headers to turn this payload into a working program. Finally, we’ll make sure all offsets and virtual addresses are correct and fill in the final blanks.

A word of warning: constructing an ELF executable by hand can be extremely frustrating. I’ve provided an example binary myself which you can use to compare your work to, but keep in mind that there is no compiler or linker to tell you what you’ve done wrong. If (read: when) you screw up, all your computer will tell you is ‘I/O Error’ or ‘Segmentation Fault’, which makes these programs extremely hard to debug. No debugging symbols for you!

Constructing the payload – let’s try to keep the payload simple but sufficiently challenging to be interesting. Our payload should put ‘Hello World!’ on the screen, then exit with code 42. This is harder than it looks. We’ll need both a text segment (containing executable instructions) and a data segment (containing the ‘Hello World!’ string and some other minor data. Let’s take a look at the assembly code we need to achieve this:

    (text segment)
    mov ebx, 1
    mov eax, 4
    mov ecx, HWADDR
    mov edx, HWLEN
    int 0x80

    mov eax, 1
    mov ebx, 0x2A
    int 0x80

The code above isn’t too hard, even if you’ve never done much assembly. Interrupt 0×80 is used to make system calls, with the values in the registers EAX and EBX telling the kernel what kind of call it is. You can get a more comprehensive reference of the system calls and their values in assembly here.

For our payload, we’ll need to convert these instructions to hexadecimal opcodes. Luckily, there are good online references that help us do just that. Try to find one for the x86 family, and see if you can figure out how to go from the above code to the hex codes below:

    0xBB 0x01 0x00 0x00 0x00
    0xB8 0x04 0x00 0x00 0x00
    0xB9 0x** 0x** 0x** 0x**
    0xBA 0x0D 0x00 0x00 0x00
    0xCD 0x80

    0xB8 0x01 0x00 0x00 0x00
    0xBB 0x2A 0x00 0x00 0x00 
    0xCD 0x80

(The *s denote virtual addresses. We don’t know these yet, so we’ll leave them blank for now. [2])

The second part of the payload consists of the data segment, which is actually just the string “Hello World!\n”. Use a nice ASCII conversion table (‘man ascii’, anyone?) to convert these values to hex, and you’ll see that we’ll get the following data:

    (data segment)
    0x48 0x65 0x6C 0x6C 0x6F 0x20 0x57 0x6F 0x72 0x6C 0x64 0x21 0x0A

And there’s our final payload!

Building the headers – this is where it can get very complicated very quickly. I’ll explain some of the more important parameters in the process of building the headers, but you’ll probably want to take a good look at a somewhat larger reference if you’re ever going to build ELF headers completely by yourself.

An ELF header has the following structure, byte size between parentheses:

    e_ident(16), e_type(2), e_machine(2), e_version(4), e_entry(4), e_phoff(4),
    e_shoff(4), e_flags(4), e_ehsize(2), e_phentsize(2), e_phnum(2), e_shentsize(2)
    e_shnum(2), e_shstrndx(2)

Now we’ll fill in the structure, and I’ll explain a bit more about these parameters where appropriate. You can always check the reference I linked to before if you want to find out more.

e_ident(16) – this parameters contains the first 16 bytes of information that identifies the file as an ELF file. The first four bytes always hold 0x7F, ‘E’, L’, F’. Bytes five to seven all contain 0×01 for 32-bit binaries on lower-endian machines. Bytes eight to fifteen are padding, so those can be 0×00, and the sixteenth byte contains the length of this block, so that has to be 16 (=0×10).
e_type(2) – set it to 0×02 0×00. This basically tells the computer that it’s an executable ELF file.
e_machine(2) – set it to 0×03 0×00, which tells the computer that the ELF file has been created to run on i386 type processors.
e_version(4) – set it to 0×01 0×00 0×00 0×00.
e_entry(4) – transfer control to this virtual address on execution. We haven’t determined this, yet, so it’s 0x** 0x** 0x** 0x** for now.
e_phoff(4) – offset from file to program header table. We put it right after the ELF header, so that’s the size of the ELF header in bytes: 0×34 0×00 0×00 0×00.
e_shoff(4) – offset from file to section header table. We don’t need this. 0×00 0×00 0×00 0×00 it is.
e_flags(4) – we don’t need flags, either. 0×00 0×00 0×00 0×00 again.
e_ehsize(2) – size of the ELF header, so holds 0×34 0×00.
e_phentsize(2) – size of a program header. Technically, we don’t know this yet, but I can already tell you that it should hold 0×20 0×00. Scroll down to check, if you like.
e_phnum(2) – number of program headers, which directly corresponds to the number of segments in the file. We want a text and a data segment, so this should be 0×02 0×00.
e_shentsize(2), e_shnum(2), e_shstrndx(2) – all of these aren’t really relevant if we’re not implementing section headers (which we aren’t), so you can simply set this to 0×00 0×00 0×00 0×00 0×00 0×00.

And that’s the ELF header! It’s the first thing in the file, and if you’ve done everything correctly the final header should look like this in hex:

    0x7F 0x45 0x4C 0x46 0x01 0x01 0x01 0x00 0x00 0x00 0x00 0x00
    0x00 0x00 0x00 0x10 0x02 0x00 0x03 0x00 0x01 0x00 0x00 0x00
    0x** 0x** 0x** 0x** 0x34 0x00 0x00 0x00 0x00 0x00 0x00 0x00
    0x00 0x00 0x00 0x00 0x34 0x00 0x20 0x00 0x02 0x00 0x00 0x00
    0x00 0x00 0x00 0x00

We’re not done with the headers, though. We now need to build the program header table, too. A program header has the following entries:

    p_type(4), p_offset(4), p_vaddr(4), p_paddr(4), p_filesz(4), p_memsz(4),
    p_flags(4), p_align(4)

Again, I’ll fill in the structure (twice, this time: one for the text segment, one for the data segment) and explain a number of things on the way:

p_type(4) – tells the program about the type of segment. Both text and data use PT_LOAD (=0×01 0×00 0×00 0×00) here.
p_offset(4) – offset from the beginning of the file. These values depend on how big the headers and segments are, since we don’t want any overlap there. Keep at 0x** 0x** 0x** 0x** for now.
p_vaddr(4) – what virtual address to assign to segment. Keep at 0x** 0x** 0x** 0x** 0x** now, we’ll talk more about it next.
p_paddr(4) – physical addressing is irrelevant, so you may put 0×00 0×00 0×00 0×00 here.
p_filesz(4) – number of bytes in file image of segment, must be larger than or equal to size of payload in segment. Again, set it to 0x** 0x** 0x** 0x**. We’ll change this later.
p_memsz(4) – number of bytes in memory image of segment. Note that this doesn’t necessarily equal p_filesz, but it may as well in this case. Keep it at 0x** 0x** 0x** 0x** for now, but remember that we can later set it to the same value we assign to p_filesz.
p_flags(4) – these flags can be tricky if you’re not used to working with them. What you need to remember is that READ permissions is 0×04, WRITE permissions is 0×02, and EXEC permissions is 0×01. For the text segment we want READ+EXEC, so 0×05 0×00 0×00 0×00, and for the data segment we prefer READ+WRITE+EXEC, so 0×07 0×00 0×00 0×00.
p_align(4) – handles alignment to memory pages. Page size are generally 4KiB, so the value should be 0×1000. Remember, x86 is little-endian, so the final value is 0×00 0×10 0×00 0×00.

Whew. We’ve certainly done a lot now. We haven’t yet filled in many of the fields in the program headers, and we’re missing a few bytes in the ELF header, too, but we’re getting there. If everything went as planned, your program header table (which you can paste directly behind the ELF header, by the way – remember our offset in that header?) should look something like this:

    0x01 0x00 0x00 0x00 0x** 0x** 0x** 0x** 0x** 0x** 0x** 0x**
    0x00 0x00 0x00 0x00 0x** 0x** 0x** 0x** 0x** 0x** 0x** 0x**
    0x05 0x00 0x00 0x00 0x00 0x10 0x00 0x00 
    0x01 0x00 0x00 0x00 0x** 0x** 0x** 0x** 0x** 0x** 0x** 0x**
    0x00 0x00 0x00 0x00 0x** 0x** 0x** 0x** 0x** 0x** 0x** 0x**
    0x07 0x00 0x00 0x00 0x00 0x10 0x00 0x00

Filling in the blanks – while we’ve finished most of the hard work by now, there are still some tricky things we need to do. We’ve got an ELF header and program table we can place at the beginning of our file, and we’ve got the payload for our actual program, but we still need to put something in the table that tells the computer where to find this payload, and we need to position our payload in the file so it can actually be found.

First, we’ll want to calculate the size of our headers and payload before we can determine any offsets. Simply add the sizes of all fields in the headers together and that’s the minimal offset for any of the segments. There are 116 bytes in the ELF header + 2 program headers, and 116 = 0×74, so the minimum offset is 0×74. To stay on the safe side, let’s put the initial offset at 0×80. Fill 0×74 to 0x7F with 0×00, then put the text segment at 0×80 in the file.
The text segment itself is 34 = 0×22 bytes, which means the minimal offset for the data segment is 0×80 + 0×22 = 0xA2. Let’s put the data segment at 0xA4 and fill 0xA2 and 0xA3 with 0×00.

If you’ve been doing all the above in your hex editor, you will now have a binary file that contains the ELF and program headers from 0×00 to 0×73, 0×74 to 0x7F will be filled with zeroes, the text segment is placed from 0×80 to 0xA1, 0xA2 and 0xA3 are zeroes again, and the data segment goes from 0xA4 to 0xB1. If you’re following these instructions and that’s not what you’ve got, now would be a good time to see what went wrong.

Assuming everything’s now in the right place in the file, it’s time to change some of our previous *s into actual values. I’m simply going to give you the values for each parameter first, and then explain why we’re using those particular values.

e_entry(4) – 0×80 0×80 0×04 0×08; We’ll choose 0×8048080 as our entry point in virtual memory. There are some rules about what you can and cannot choose as an entry point, but the most important thing to remember is that a starting virtual memory address modulo page size must be equal to the offset in the file modulo page size. You can check the ELF reference and some other good books for more information, but if it seems too complicated, just forget about it and use these values.
p_offset(4) – 0×80 0×00 0×00 0×00 for text, 0xA4 0×00 0×00 0×00 for data. This is because of the obvious reason that that’s where these segments are in the file.
p_vaddr(4) – 0×80 0×80 0×04 0×08 for text, 0xA4 0×80 0×04 0×08 for data. We want the text segment to be the entry point for the program, and we’re placing the data segment in memory in such a way that it is directly congruent to their physical offsets.
p_filesz(4) – 0×24 0×00 0×00 0×00 for text, 0×20 0×00 0×00 0×00 for data. These are simply the bytesizes of the different segments in the file and memory. In this case, p_memsz = p_filesz, so use those same values there.

The final result – assuming you followed everything to the letter, this is what you would get if you dumped out everything in hex:

    7F 45 4C 46 01 01 01 00 00 00 00 00 00 00 00 10 02 00 03 00
    01 00 00 00 80 80 04 08 34 00 00 00 00 00 00 00 00 00 00 00
    34 00 20 00 02 00 00 00 00 00 00 00 01 00 00 00 80 00 00 00
    80 80 04 08 00 00 00 00 24 00 00 00 24 00 00 00 05 00 00 00
    00 10 00 00 01 00 00 00 A4 00 00 00 A4 80 04 08 00 00 00 00
    20 00 00 00 20 00 00 00 07 00 00 00 00 10 00 00 00 00 00 00
    00 00 00 00 00 00 00 00 BB 01 00 00 00 B8 04 00 00 00 B9 A4
    80 04 08 BA B1 80 04 08 CD 80 B8 01 00 00 00 BB 2A 00 00 00
    CD 80 00 00 48 65 6C 6C 6F 20 57 6F 72 6C 64 21 0A

That’s it. Run chmod +x on the binary file and then execute it. Hello World in 178 bytes.[3] I hope you enjoyed writing it. :-) If you thought this HOWTO was useful or interesting, let me know! I always appreciate getting an email. Tips, comments and/or constructive criticism are always welcome, too.

[1] Theoretically, you don’t even need to use chmod or a similar command if your configuration files contain stupid umask values. Don’t do that, though.

[2] Your hex editor may not allow adding garbage data that isn’t actually in hex to your files. If so, it’s preferable to use magic hex numbers to denote “This should be changed later”, such as 0xDEADBEEF or 0xFEEDFACE.

[3] You could of course go much smaller than that, but that’s something for another post.
