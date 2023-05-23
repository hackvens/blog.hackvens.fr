---
layout: post
title: Baby Octogone
category: Sthack-2021
---

## 1 - The challenge

The challenge consists of a `baby_octogone.elf` file to download and that's it!

## 2 - The basics

First let's get some informations about the file

```bash
file baby_octogone.elf
readelf -a baby_octogone.elf
binwalk baby_octogone.elf
```

Here is a little summary of the most important informations :

- ELF 32-bit LSB executable
- QUALCOMM DSP6 Processor
- statically linked
- little endian

Let's use `strings` to see if we get something:
```
...
RUsage: %s <secret0> <secret1>
[!] Well done, the flag is: STHACK{ %s%s}
[!] Noooope
Invalid command Line string!
...
```

It looks like we will have to reverse two secrets that will give us the flag.
Maybe a simple concatenation of the secrets will do the job?
We will see.

## 3 - Getting to know the processor architecture

The first thing I read was the [Wikipedia page](https://en.wikipedia.org/wiki/Qualcomm_Hexagon).
In the sources we can find some valuable informations about the way the processor works.
For example this short [presentation](http://pages.cs.wisc.edu/~danav/pubs/qcom/hexagon_hotchips2013.pdf).

Now that we have a global idea of how the processor works let's check the
[technical documentation given by the editor](https://developer.qualcomm.com/software/hexagon-dsp-sdk/tools).

The [Qualcomm Hexagon V67 Programmer’s Reference Manual](https://developer.qualcomm.com/qfile/67417/80-n2040-45_b_qualcomm_hexagon_v67_programmer_reference_manual.pdf)
is quite a good start to understant the basics for the next reverse step.
Let's take some time to quickly read it and get the basics of the assembly language.

## 4 - The reverse

### 4.1 - What tools to use

I tried to find a way to emulate the processor but I was not able to have something
to work within the time constraints.

It looks like we will have to do a complete static analysis.

After some trial and error with the Cutter configuration I was able to get a proper disassembly.

### 4.2 - The main function

_Note: The screenshots contains the already renamed functions because they were taken after the end of the CTF._

The are no available symbols so we will need to do some checks by hand.
Maybe finding the function that refers tu the usage string we found earlier ?

![Cutter graph of the main function](/assets/img/sthack2021/main_graph.png){:class="img-responsive smallpict"}

Bingo !
We can also see the main condition calling the format string giving us the flag and
the "Noooope".

After a quick analysis, we rename the functions as we think they might work.

_NOTE: I renamed the functions "maybe\_check\_secret\_1" and "maybe\_check\_secret\_2" instead of 0 and 1. It was late._

### 4.3 - maybe\_check\_secret\_1

This function is quite simple, it is a simple string length check and some xor.

![Cutter graph of the maybe_check_secret_1 function](/assets/img/sthack2021/maybe_check_secret_1_graph.png){:class="img-responsive smallpict"}

Let's decrypt the first secret:

```
xor 1732584193 153835079 = 1852732742   = 0x6e6e7546 (nnuF)
xor -271733879 -1618021168 = 1883463513 = 0x70435f59 (pC_Y)
xor -1732584194 -359758933 = 1916034901 = 0x72345f55 (r4_U)
xor 271733878 956840981 = 691693667     = 0x293a6863 ():hc)

===> FunnY_CpU_4rch:)
```

### 4.4 - maybe\_check\_secret\_2

This one had me think a bit more.

![Cutter graph of the maybe_check_secret_2 function](/assets/img/sthack2021/maybe_check_secret_2_graph.png){:class="img-responsive smallpict"}

To put it in a nutshell, the function:
- iterate over the given string AND a data stored in memory at address `50472`
- stores in R2 the byte that is stored in memory at address `50216 + the value of the current char`
- stores in R3 the byre that stored in memory at address `50472 + iteration number`
- compares R2 and R3
- continues if they are equals

Because some values appears multiple times in the memory space used to get R2's value
we might have multiple combinations possible.
To get the secret I simply copied the memory spaces used in hexadecimal and
made a python script that gives me the possible combinations.

```python
hex_str = '5ef1799f18f8b3c641a0a88e8dbeef273c771b96d986838eca043ce9f808bf05b06edde1a4097db775189eed6566e068662c09142fab72dc565c3648f1f34f9e9a2f0800b05c49bfdc6691156f358bcc83351263f578abb0d687b6042f80ab9b63c18d458a5b5cede91d89e7af31596746a6604cf134da7a961cfc8737f0604c91a9635e5873141ae22995a27c9bf6b08f515d6e5c470a380f7e468edf0ff5122f88debc2e396605a8a267cbc055e18e008d98b367ec8a635c7a760f244777eadd45ceceffcd5b70498a3f9982ffb6ff398005f378d411cb6d4ce7bc4658803584b49fe2000c75ef3a257fa23544a45b1a9eb779d88c042585d569ba66901847'

hex_ref = '2f598a9b5bc14c1c9b45e92fafaf6e'

mem = bytes.fromhex(hex_str)
ref = bytes.fromhex(hex_ref)

out = []

for r in ref:
    tmp = []
    for i in range(32, 127):
        if r == mem[i]:
            tmp.append(chr(i))
    out.append(tmp)

print(out)
```

Here are all the possibilities:
- 4nd_easy_ch4ll!
- And_easy_chAll!
- 4nd_easy_chAll!
- And_easy_ch4ll!

### 4.5 - Build the flag

Because of the second secret's results they were 4 possibilities :
- STHACK{FunnY_CpU_4rch:)4nd_easy_ch4ll!}
- STHACK{FunnY_CpU_4rch:)And_easy_chAll!}
- STHACK{FunnY_CpU_4rch:)4nd_easy_chAll!}
- STHACK{FunnY_CpU_4rch:)And_easy_ch4ll!}

I assume all of them are valid but I only tried the first one.

I hope you enjoyed :)

___Author: Hugo SOSZYNSKI___