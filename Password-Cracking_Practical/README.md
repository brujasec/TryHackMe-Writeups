# Password Cracking Practical

## Introduction

Passwords are usually stored as hashes. I say usually, because although storing plain-text passwords is a huge mistake. Before we can crack a hash, we need to know the algorithm that produced it. 

In this practical, TryHackMe gives four files containing hashes. We must then find the hashtypes using `hashid` to help us select the proper mode or format to run password cracking tools Hashcat and John the Ripper. We will then run a dictionary attack using rockyou.txt. Rockyou is a popular wordlist containing 14 million passwords leaked in the 2009 RockYou breach. 

Dictionary attacks will usually be the first step in cracking hashes, as this attack tests pre-built lists of passwords one-by-one, against the hash. 

For a quick way to look up hash types, you can also use crackstation.net and hashes.com.

## Let's Get Cracking

First view the hash with `cat hash1.txt` and so on.

```
cat hash1.txt
e10adc3949ba59abbe56e057f20f883e
```


Identify the hash with `hashid`.

`hashid 'e10adc3949ba59abbe56e057f20f883e'`

```
Analyzing 'e10adc3949ba59abbe56e057f20f883e'
[+] MD2 
[+] MD5 
[+] MD4 
[+] Double MD5 
[+] LM 
[+] RIPEMD-128 
[+] Haval-128 
[+] Tiger-128 
[+] Skein-256(128) 
[+] Skein-512(128) 
[+] Lotus Notes/Domino 5 
[+] Skype 
[+] Snefru-128 
[+] NTLM 
[+] Domain Cached Credentials 
[+] Domain Cached Credentials 2 
[+] DNSSEC(NSEC3) 
[+] RAdmin v2.x
```

The hash type is MD5. Crack it with John the Ripper, using MD5 format, and the path to the wordlist, rockyou.txt

`john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt --rules=wordlist hash1.txt`

Then read the cracked password:

`john --show --format=raw-md5 hash1.txt`
```
?:123456

1 password hash cracked, 0 left
```

123456 is the password for hash1.txt.

Hash2.txt and Hash3.txt share the same plaintext password, but use different algorithms.

Let's figure out the hash for hash2.txt:

```
cat hash2.txt
5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8
```

The hash is SHA-256:

```
hashid '5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8'
Analyzing '5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8'
[+] Snefru-256 
[+] SHA-256 
[+] RIPEMD-256 
[+] Haval-256 
[+] GOST R 34.11-94 
[+] GOST CryptoPro S-Box 
[+] SHA3-256 
[+] Skein-256 
[+] Skein-512(256)
```

Since it is a SHA-256 hash, crack it with hashcat using mode 1400:

`hashcat -m 1400 -a 0 hash2.txt /usr/share/wordlists/rockyou.txt`

How this command works:

*`-m 1400`= hash type

*`-a 0`= dictionary attack mode

*`hash2.txt`= file containing the hash

*The path to the wordlist

When Hashcat is done running, you'll see the password in the output:
```
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344391
* Bytes.....: 139921497
* Keyspace..: 14344384
* Runtime...: 2 secs

5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8:password
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 1400 (SHA2-256)
Hash.Target......: 5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a1..
```

Now onto hash3.txt:

```
cat hash3.txt
8846f7eaee8fb117ad06bdd830b7586c
```
Identify the hash:
```
hashid '8846f7eaee8fb117ad06bdd830b7586c'
Analyzing '8846f7eaee8fb117ad06bdd830b7586c'
[+] MD2 
[+] MD5 
[+] MD4 
[+] Double MD5 
[+] LM 
[+] RIPEMD-128 
[+] Haval-128 
[+] Tiger-128 
[+] Skein-256(128) 
[+] Skein-512(128) 
[+] Lotus Notes/Domino 5 
[+] Skype 
[+] Snefru-128 
[+] NTLM 
[+] Domain Cached Credentials 
[+] Domain Cached Credentials 2 
[+] DNSSEC(NSEC3) 
[+] RAdmin v2.x
```

It looks like MD5, but when I applied the MD5 format to John the Ripper, it didn't work. MD5 and NTLM are both 32 hex characters. NTLM hashes are usually derived from Windows SAM files or dumped from Active Directory. NTLM is used for legacy Windows authentication.

Use John `format=nt` for NTLM to crack the hash.

`john --format=nt --wordlist=/usr/share/wordlists/rockyou.txt hash3.txt`

Read the password in plaintext:

```
john --show --format=NT hash3.txt`
?:password

1 password hash cracked, 0 left
```

Something interesting happens with hash4.txt:
```
cat hash4.txt
$2b$05$9I7YCSrgm6aLO7J5YPC9x.Kp08LQ7cSJTmkALhFTgm5UMFAwBr5.e
```
The hash begins with `$2b$`, identifying it as a bcrypt hash. Bcrypt is a recommended hash for passwords because it is slow and costs more to crack. This attack will take a while. Use hashcat mode 3200.

`hashcat -m 3200 -a 0 hash4.txt /usr/share/wordlists/rockyou.txt`

You'll see the password toward the end of the output:
```
Rejected.........: 0/58560 (0.00%)
Restore.Point....: 58556/14344384 (0.41%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-32
Candidate.Engine.: Device Generator
Candidates.#1....: heather11 -> hayden07
Hardware.Mon.#1..: Util: 84%

Started: Thu Jul  2 19:21:24 2026
Stopped: Thu Jul  2 19:24:15 2026
```

## Conclusion

Cracking a password is easier than it looks! This practical challenge shows why strong passwords are important, along with using strong hashes to store them. Salting is also important to prevent identical hashes. A salt is a unique random string genrated for each user and combined with that user's password before hasing. This is useful because if two users have the same password, adding a salt prevents their hashes from being indentical. 






