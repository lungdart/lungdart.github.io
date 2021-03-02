# TryHackMe : Wonderland
## HTTP

http://10.10.221.206/r/a/b/b/i/t

Hidden p = alice:HowDothTheLittleCrocodileImproveHisShiningTail

## SSH
Hidden <p> gave me access

## USER
Hint: **Everything is upside down here**

* Alice does not have user.txt in her directory, but she has root.txt
* There are a few other users:
  * *rabit*
  * *hatter*
* Alice has a python script that's root owned, and can not be written to
* Alice can run the python script as the rabbit user
* The script contains a powem by Lewis Carroll"

With the hint being about "upside down" and the root flag in the user folder, I checked the root folder for the user flag and there it was

### Rabbit
If I can change the contents of that python script, I could gain access to the *rabbit* user

That python script imports random. By creating a random.py in the same directory as the script to be run, I can override the random.choice() function to drop me into a shell (Since local files take precident over installed modules)

**I'm now rabbii**

Rabbit has a setuid binary in his home folder that always waits 1 hour to get tea from the mad hatter. It asks you to ask nicely for tea while you wait. If you get it wrong, it segfaults

If you *strings* that binary, it calls date without an absolute path. Created a date script to run bash interactively, and added rabbit's home directory to the **front** of PATH. running the script now gives me bash as the hatter user


### Hatter
/home/hatter/password.txt
Hatters password: WhyIsARavenLikeAWritingDesk?
SUDO?
