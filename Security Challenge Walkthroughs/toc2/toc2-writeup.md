
*toc2

Welcome 

**Enumeration:

Port Scan:
  SSH on Port 20
  Apache on Port 80

Going to the web server we see a an "Under Construction!" notice, with a short message from an angry Sysadmin and some credentials. Enumerating further, by checking the 
robots.txt we see a directory that's been blocked from crawling, along with another note about finishing the CMS setup. 

** Foothold: **

Visiting that directory, brings us to the CMS Made Simple "Installation and upgrade assistant" page. This would normally be used to set up a working cmsms installation,
however- by checking the version number (2.1.5) and looking it up, we find that there is a CVE (CVE-2018-7448) associated with it. This CVE allows Arbitrary PHP code to be 
injected into configuration file, using the "timezone" parameter during setup. 

In order to exploit this, we follow the installation as normal, using the credentials from the "Under Construction!" page, and the database name from the robots.txt message- 
we proceed until step 4. Configuration Info. Here we use burp, or any intercept proxy of your choice, to capture the request, and insert our PHP code into the timezone
parameter. A good example from the CVE is:

  timezone=junk';echo%20system($_GET['cmd']);$junk='junk 
  
We then finish the installation. In order to get code execution, following the example above, we go to the standard location of config.php for a cmsms install:

  /cmsms/config.php?cmd=id;uname

Where we can use the "cmd=" paramater to run commands, in this case "id". 

** User: **

Leveraging this, we can explore the system. Checking the /home directory, we find one user- Frank. That's our disgruntled admin from before. He's left a note in his home 
directory that gives us a clue as to what his password could be. Trying his login through SSH gives us access to his account, and access to the user flag.

** Privesc: **

In the folder "root_access", there will be an SUID binary called "read_creds", the source code for that binary, and a file called "root_password_backup". This file is owned 
by root, and cannot be read by anyone except the owner. Fortunately, there's an SUID binary that reads files just sitting there, right? 

Well it's not quite that simple, the "read_creds" binary won't read any file owned by uid 0 (root) as we can see in the source code. Looking into the source code, we can 
break the binary down into each process:

1. The binary first calls "stat" to get the file information, using the file path. 
2. It then checks the uid using the information from stat, if the file's uid is 0 (root) it prints an error message and exits
3. If not, it uses open on the file path passed to the binary and assigns that to a variable
4. If the variable is not empty, it cats the variable to output it. 

There's a cheeky race condition lurking here. The binary calls the file path twice, once to check the status- and once to actually open the file. Therefore, if we can switch 
the file fast enough, we can change the file path, so that the first time it gets called- it's a file belonging to the user, which we can read. And then- the second time it 
calls the file path- thinking it belongs to us, but actually it's the file belonging to root, it will read it out, due to it being SUID and having correct permissions to 
do so. 

This type of Race Condition is called Time of Check to Time of Use, you can read more about it here: https://en.wikipedia.org/wiki/Time-of-check_to_time-of-use 

But how do we switch two files that fast! Fortunately, there is a syscall that lets us swap two files, called "renameat2". Using this system call, with two file path 
arguments and the option "RENAME_EXCHANGE", we can switch the names and file paths of the two files. There is already a code snippet that lets you use this online, for 
exploiting these vulnerabilities : https://github.com/sroettger/35c3ctf_chals/blob/master/logrotate/exploit/rename.c

Getting this code via wget, and compiling it with gcc:
  
  gcc rename.c -o rename

Gives us an executable binary called "rename" that we can use to switch two file paths extremely quickly, using the syscall. Pretty cool right?

So, lets create a file owned by us that we can swap with, for this example I used "touch" to create a file called "swap". 

  ./rename root_password_backup swap & 

To run the file path swapper in the background, and then used:

  ./readcreds root_password_backup

Repeatadly until it outputs the root password. You can then use this to login to the root account with "su root" and get the root flag!
