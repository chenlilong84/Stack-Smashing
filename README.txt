Group Members: Nick Balkissoon, Ethan Gottlieb, Arihant Jain
NetIDs: ngb5, eig4, aj148 
Date: 3/31/2015
Server: http://cps110.cs.duke.edu:8102

Smashing the Stack -- Hacking into a Vulnerable Server

In this project, we took advantage of a buffer overflow attack in order to gain access to a vulnerable server.  This attack, in brief, involves sending a request (a string) to the server which overflows the string's buffer, thus spilling into the function's stack and allowing us to manipulate the stack's data.  Specifically, we format this attack string such that it overwrites the function's return address and instead directs the function to return into our attack string.  Within our attack string we have shellcode which the program will begin executing, opening up a shell listening on a specific port of the server.  Then, anyone can remotely connect to this shell and begin manipulating files!

THE VULNERABILITY

After scanning the server code for vulnerabilities, we discovered a bug within the handle() method that we were able to exploit in our attack.  This method calls:

check_filename_length(len)

which then makes the following check:

int check_filename_length(byte len) { 
	if (len < 100)
		return 1;
	return 0;
}

This check looks at the length of this string as a byte instead of an int, therefore as long as the last 8 bits of the length of our string are less than 100 (or 1100100 in binary) we should be able to pass this check.  This will allow us to pass in an arbitrarily large string to the handle() function, allowing us to overflow the buffer because the program only allocates space for a string with length 100.

THE ATTACK

The next challenge is to carefully write this attack string so that we can:
1) Overflow the buffer
2) Return into and begin executing our shellcode
3) Open up a shell for remote connection
4) Cause havoc

We do this by formatting our string in the following way: {Return Addresses} + {No-Op Code} + {Shell Code}. We must include the No Op code ("\x90" on x86 machines) so that we can increase the probability of "guessing" the location of our shellcode on the server.  (Originally, we formatted our string in this way: {No-Op Code} + {Shell Code} + {Return Addresses}. However, this format required us to condense everything into a much shorter string, and therefore lowered our probability of “guessing” locations correctly and made our job more difficult)

Before we attacked the actual server, we hosted the server code on our local machine and attacked it using GDB to find a working return address. For our shell code, we tried a using and editing a few different variations, and eventually settle on the shellcode from an online source (http://www.steve.org.uk/Security/Exploits/cmd-overflow.c).

To ensure that our return address would replace the server’s return address, we repeated the return address 50 times (200 bytes) at the beginning of our string. We then used 555 bytes of No-Ops, because it provided a large enough “slide” that we were eventually able to scan 3000 bytes of address space with only 6 guesses! After the return addresses and no ops, we included one copy of our shellcode, and were able to stay under 1000 bytes of total length in order to avoid accidentally overwriting kernel code. We wrote mainframe.c (attached) in order to automatically generate these attack strings. The c file takes our addresses and length specifications and creates the attack string in the correct format.

After we were able to successfully smash the stack on our local server, we then attempted to hack into the web server. The main difference was we needed to find the correct return address from the server, without the help of GDB. To do this we started from the highest stack address (0xBFFFFFFF) and then found the address values downwards at increments of 300 bytes (using hex arithmetic). After testing a few return addresses on the server unsuccessfully (seg faults), we stumbled upon a working return address (\xa7\xfd\xff\xbf).

The  mainframe.c generates a file called shellcode.dat which we can send as a request to the server.  We run the following commands:

gcc -m32 -z execstack -fno-stack-protector mainframe.c -o mainframe
./mainframe

This generates shellcode.dat.  We then send this as a request to the server with the following command: 

cat shellcode.dat | nc cps110.cs.duke.edu 8102

Now, the server has spawned a shell listening to port 2121.  We are able to remotely connect to this port with the following command:

nc cps110.cs.duke.edu 2121

Once we have opened up this shell we can manipulate data on the server in a variety of ways. In the shell, we used the “vim” command to make some minor edits to the index.html file and we used the “wget” command to download an image of Landon Cox in the www directory.

PROJECT REVIEW

I thought this project was a great introduction to understanding networking and GDB, and a basic introduction to shell and shell programming.  This was also a nice refresher on computer architecture (i.e. stacks, return addresses, assembly). I wish we had a little bit more instruction on what shell is and what shellcode is and how to write it.  I think a lot of our difficulties came from not completely understanding what our shellcode was doing. It is cool knowing that we can now hack into a website, like anonymous and the hackers in the movies!


