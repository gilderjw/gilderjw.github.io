---
title: Extracting Code from Crackme to Create Keygen
layout: post
data: 2018-02-14
categories: reversing
---

In this post we will go over the process that I used to solve the [Abduction crackme](./assets/2018-02-14-extracting-code-from-crackme-to-create-keygen/Abduction_Keygenme.zip)

## The Challenge

The challenge zip contains two files: The challenge binary and and readme with rules for the challenge.

<strong>Do not read me!.txt</strong>
> / Abduction Crackme \

> \    by sd333221    /

> / as the 23.10.07   \

> \   Level 3.5/10    /

> \|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|

> In this crackme i use new technologies I learned, to make your life as
> a cracker very hard.
> 
> I coded it, as I would code a software, I want to protect (expect that
> I didn't use a strong protector).
> 
> There are several anti-debug tricks to beat your debugger up,
> I avoided to make it crash your system but I don't take any responsabilities for that.
> 
> Rules:
> - Break the Security
> - Recover the unlock key - For a username you want,
>   but this task is not enough for a valid solution it is
>   just for noobs since you can get the serial directly
> - Do not patch or selfgen
> - Make a Keygen
> - Write a tutorial
> 
> Enjoy the intro,
> \>\>you can skip it by selecting it and pressing Alt+F4\<\<
> 
> 
> I hope you enjoy it!
> 
> Greetings

> sd333221

## The Program

This application consists of two text boxes for a username and a password. When you enter an invalid password, it spawns a dialog box saying that you have failed.

![crackme with invalid input](./assets/2018-02-14-extracting-code-from-crackme-to-create-keygen/initial_run.png)

## Analysis


### Looking for Strings

We being our analysis in IDA Pro. To find the logic that we are interested, we can find a reference to the failure string in the code. 

![ida disassembly of the button listener](./assets/2018-02-14-extracting-code-from-crackme-to-create-keygen/ida_button_listener.png)

The code that references our string is in a large function that looks like some sort of button handler. The relevant code for our purposes begins at loc_40BD20 after the program knows which button was pressed. There are two calls to `GetDlgItemTextA` which makes sense as calls to get the username and password boxes from the GUI. We can use WinDbg to figure out the which calls get which text box.

### The Debugger Trick

We can open the program in gdb and set a breakpoint before the first call but when we try to execute the program, the debugger will freeze up and never hit the breakpoint:

![WinDbg not able to deal with debugger tricks](./assets/2018-02-14-extracting-code-from-crackme-to-create-keygen/windbg_broken.png)

This isn't too much of an issue, we just need to go back to IDA and patch out the trick so we can debug the application. After opening the function list in IDA, it appears that there is a TLS callback.

![TLS callback in function list](./assets/2018-02-14-extracting-code-from-crackme-to-create-keygen/ida_function_list.png)

A Thread Local Storage (TLS) callback is a function that you can point to in the PE header that will execute before the main. WinDbg will not break before the TLS code runs, so we don't know what it does. The readme mentions that there are anti-debug tricks in the executable, so we can assume that this is one of them. To bypass this possible trick, we can just zero out the entry in the PE header and see if we still have a functioning executable. This is easy to do with [PE Explorer](http://www.heaventools.com/overview.htm).

![clearing out TLS table in PE Explorer](./assets/2018-02-14-extracting-code-from-crackme-to-create-keygen/PE_explorer.png)

Once that is cleared and we have verified that the program still works, we can see if our debugger breaks after the first call to `GetDlgItemTextA`.

![break right after first call to `GetDlgItemTextA`](./assets/2018-02-14-extracting-code-from-crackme-to-create-keygen/break_on_GetDlgItemTextA.png)

After checking the state of memory after each call, we know that username is stored at `ebp-0x208` and the password is stored at `ebp-0x108`

### More Static Analysis

With just a little bit more assembly reading, we see that the username and password get passed into the last call of the function. The return value of that function is then compared to zero and the application jumps to the message box containing a success or failure depending on that value. It looks like in order to crack this software we will need to look at this function call.

Looking at this function, we can see that it takes the username, sums up the ascii characters, does some multiplications on the result, then prepends "gRn-" to the beginning of the hex value. This string is then hashed with some unknown function and compared to the password field.

![interesting hash stuff 1](./assets/2018-02-14-extracting-code-from-crackme-to-create-keygen/ida_interesting_hash_stuff_1.png)

![interesting hash stuff 2](./assets/2018-02-14-extracting-code-from-crackme-to-create-keygen/ida_interesting_hash_stuff_2.png)

I was unable to identify the hash function used and was not entirely thrilled to read through a bunch of assembly to reverse engineer what I believed to be a custom hash function. To avoid hours of mind-melting assembly reading, I decided that it would be much easier to write my keygen using the code that is in the binary itself.

### Carving Out the Hash Function

Carving out the code for the custom hash function proved to be a much harder task than anticipated. My strategy was to disassemble `interesting_hash_stuff` and all functions that it depended on. Then I would be able to just reassemble the code and call it from a C program. 

I started by producing an ASM file with IDA and tried to reassemble. It turns out that IDA does not actually produce assembly code that can be assembled by gas or nasm. Luckily I found a [nice ida script](https://github.com/gilderjw/abduction_crackme_solution/blob/master/generate_nasm.idc) that turns the IDA assembly of the current function selected into something pretty close to nasm. I extracted the main function into something that I could call from my own IDAPython script.

After learning some basics about writing an IDAPython script, I made [this one](https://github.com/gilderjw/abduction_crackme_solution/blob/master/get_instructions.py) to recursively disassemble all function that `intersting_hash_stuff` depended on. Standard library functions called were omitted from the disassembly so that they could just be linked in when making the final product. The output of my script then had to go through a couple regex replaces in a text editor and references to the data segment had to be fixed before I had a perfectly fine assembly. I also removed the equality check from `interesting_hash_stuff` so it would return the value of the password instead of the value of the comparison.

![comparison removed from interesting_hash_stuff](./assets/2018-02-14-extracting-code-from-crackme-to-create-keygen/ida_interesting_hash_stuff_3.png)

Once I had my object file generated with nasm, I was able to write my C program:


```C
#include <stdio.h>
#include <string.h>

#define HASH_SIZE 32
#define MAX_USER_LENGTH 128

char* _interesting_hash_stuff(char*);

int main(int argc, char const *argv[])
{
  if (argc != 2) {
    printf("Usage: %s <username>\n", argv[0]);
    return 1;
  }

  char *serial = interesting_hash_stuff((char*) argv[1]);

  printf("%s\n", serial);

  return 0;
}
```

When the program was compiled and linked with my carve out of the crackme, I had a working keygen:

![Keygen generate password for j33m](./assets/2018-02-14-extracting-code-from-crackme-to-create-keygen/keygen_j33m.png)

![key tried](./assets/2018-02-14-extracting-code-from-crackme-to-create-keygen/keygen_success.png)