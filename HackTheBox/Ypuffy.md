# HackTheBox | Ypuffy Walkthrough

## My journey to user.txt

When I start off with a box I like to either do a bigboi masscan or smolboi nmap scan depending where I am with time. For this box I started off with a super simple nmap scan. (My terminal colors look like barf but that's not important).

`-F Fast mode - Scan fewer ports than the default scan`

So. We definitely don’t have enough information from that initial scan. Let’s try enumerating these specific ports even more.

For this, I used the command:

`nmap -p 22,80,139,389,445 -sV -sC 10.10.10.107`

![NmapScan](https://cdn-images-1.medium.com/max/1600/1*gbHqCEaq5kuo32DQahp5GA.png)

Interesting. So now we have a little more information. However, this is where I felt like I hit a wall for 30 minutes and tried brute-forcing the login page for about 10 minutes before something clicked.. What if everything is right in front of me? That’s when I tried messing around with port 389/tcp ldap. This is when I threw myself into learning about ldapsearch.

![ThinkingDeku](https://cdn-images-1.medium.com/max/1600/1*o8c1fjMGGj_E9D_dJiLL-w.gif)

After reading the ldapsearch man page through a couple of times I finally sat down and crafted my command.

`ldapsearch -h 10.10.10.107 -xb “dc=hackthebox,dc=htb”`

![LdapSearch](https://cdn-images-1.medium.com/max/1600/1*pz2ChZJOMNkE9DXFCMGflw.png)

Now that we have all of Alice’s details we can enumerate some information. Let’s use smbmap to log straight in — if you’re not familiar with smbmap it “allows users to enumerate samba share drives across an entire domain. List share drives, drive permissions, share contents, upload/download functionality, file name auto-download pattern matching, and even execute remote commands.”

`smbmap -H 10.10.10.107 -u alice1978 -p ‘00000000000000000000000000000000:0B186E661BBDBDCF6047784DE8B9FD8B’`

![SMBMap](https://cdn-images-1.medium.com/max/1600/1*oNDh47s4-YK1E5w6Y5eRXw.png)

As we can see there are two shares: alice and IPC — However, it’s important to note that we have “NO ACCESS” to IPC so let’s go straight into alice.

If we look at the man page of smbmap we can see that -R will help us out with this.

`-R: Recursively list dirs, and files (no share\path lists ALL shares), ex. 'C$\Finance'`

This gives us the result:

`[SMBMapRecursive](https://cdn-images-1.medium.com/max/1600/1*rh0bSVMXGrjDVZJ2u5zB7g.png)

There’s nothing in this share other than “my_private_key.ppk” so it’s okay to assume that this is the direction to head in.

We’re now going to download this key and view its contents — this will allow us to see what we’re actually working with.

![PrivateKeyDiscovery](https://cdn-images-1.medium.com/max/1600/1*YRhG3GaPQdTVlmS8UUP4Ag.png)

As we can see — this key is actually a PuTTY key and we’re going to want to convert that to an OpenSSH key.

Some quick googling told me this was possible by apt-get’ing putty-tools and using puttygen. To convert the key I used the command:

`puttygen 10.10.10.107-alice_my_private_key.ppk -o alice -O private-openssh`

Now all we have to do is (hopefully) ssh in and..

![SuccessfulSSH](https://cdn-images-1.medium.com/max/1600/1*79qRKuUKdYs2rs3hL3mb4A.png)

It worked! Now let’s ls and see if we can find the user.txt file!

![UserTXT](https://cdn-images-1.medium.com/max/1600/1*jnaPzg6CppvikdAMZWe2Kg.png)

## My journey to root.txt

When I get access to system I like to be nosy and see what users are on the system/what’s in the history/everything that’s of importance. One of the first things I noticed about this box is that there’s three users.

![UserEnumeration](https://cdn-images-1.medium.com/max/1600/1*jw6-pIVqX8jI0qfdSn77wA.png)

Interesting.. We already know what’s in Alice’s directory so let’s just go straight for userca since that sounds more interesting than bob8791

![AliceDirectory](https://cdn-images-1.medium.com/max/1600/1*2IZ9PyRAZ0C0YT8BzPnSBg.png)

There doesn’t seem to be much here.. Let’s go and check out bob8791’s directory.

![BobsDirectory](https://cdn-images-1.medium.com/max/1600/1*1Wdnpz0pe6fd3DlzBEH04g.png)

Now this is looking way better.. I wonder what we can do with sshauth.sql? Let’s inspect it and see what’s going on.

![SSHAuthSQL](https://cdn-images-1.medium.com/max/1600/1*B1inOmwn9JWAH3hzvU4C5g.png)

You know what would be dumb? If we could just use PostgreSQL and view all the datas. So let’s do that.

![psql](https://cdn-images-1.medium.com/max/1600/1*1GEBOfIY7UC2yrUw7I2yHg.png)

Shoot. Did I hit a dead end?

The answer is no. I just didn’t read the output of more sshauth.sql — it clearly says “grant select on principals,keys to appsrv;”

So how about we do the same thing but just throw in an appsrv?

![BobbyTables](https://cdn-images-1.medium.com/max/1600/1*EKmuwEtPWvhQqIEW6L0krg.png)

Now let’s see some tables

![YummyTables](https://cdn-images-1.medium.com/max/1600/1*EapmsyftVtT2qbqF_8Ip5g.png)

Waduhek? This stuff looks pretty important. Let’s select from principals and see what’s going on.

![l00t](https://cdn-images-1.medium.com/max/1600/1*jDb1K-UFFln7P26J0-aHcg.png)

So.. Does this mean I can just create an SSH key and sign it with the principal 3m3rgencyB4ckd00r and get root? There’s only one way to find out.

Let’s start off by creating a key named root and then trying to sign it.

![SignKey](https://cdn-images-1.medium.com/max/1600/1*4YT-CECPuKyZwrVOVtLhzg.png)

Yet again — I felt like I hit a wall for a bit here. I kept trying to sudo (and couldn’t) when I finally realized that this is OpenBSD and the doas command will work instead.

![OpenBSD](https://cdn-images-1.medium.com/max/1600/1*3eWIppT9ZDQ_DdKKIqo81g.png)

Finally! I signed the key. My command syntax got a little weird — but I decided to leave the “full” command out. I learned a lot from just trying to nail the syntax and I think it’s beneficial if you’re doing this machine to mess around with it and see what works/what doesn’t.

![SuccessfulAuth](https://cdn-images-1.medium.com/max/1600/1*hVyekwe5_jSmzNapWgAl0w.png)

And we’re in! We can now view the root.txt and we’ve successfully owned this machine.

![Success](https://cdn-images-1.medium.com/max/1600/1*47wI_cfCT4GYxlRvZEJrNg.gif)

## Thoughts
Overall — I didn’t find this machine super easy. I found that obtaining user.txt was fairly simple but root.txt required a little thinking outside the box (for myself).

It’s important to note that this box wasn’t as straightforward to me as it may seem in this walkthrough. There were hours spent on it and I often find HackTheBox walkthroughs not acknowledging the difficulty of some of these boxes.

Thanks for reading, I hope you learned something from reading this because I definitely did from doing it. I find that writing down my steps after completing a box helps me learn even more.
