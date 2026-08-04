<!--
WHAT THIS FILE IS
Short print version for the university magazine. ~450 words of body copy.
Written to be dropped into Canva, InDesign, or a Google Doc and laid out.

For whoever is designing this:
- No code blocks, no tables, no links. Everything is plain text you can flow into columns.
- The piece is built from a HEADLINE, a STANDFIRST, four short SECTIONS, one PULL QUOTE,
  and one SIDEBAR BOX. Those labels are marked below and should be deleted once placed.
- The pull quote and the sidebar are meant to sit outside the main text flow.
- Body copy runs about 450 words, which is roughly one magazine page with the box.
- Cut the fourth section first if you need to lose space.
- Delete this comment block before you paste.
-->

**HEADLINE**

Your app is running. Nothing can reach it.

**STANDFIRST**

I put my website on a free server. Here is what broke, in the order it broke, and what I
wish someone had told me first.

**SECTION 1**

Oracle Cloud gives students a free server that never expires: four processors and 24 GB of
memory, permanently, for nothing. Mine has run for three months and hosts a website, a
database, and two other people's projects.

Getting one is the easy part. Everything after it fails quietly.

**SECTION 2**

Your first deploy works. You run your app, you check it, it responds. Then you close your
laptop and the site dies with it, because your program was a child of your login session
and the session ended.

Linux already has the fix. It is called systemd, it starts every other service on the
machine, and it takes nine lines to describe your program. After that it starts on boot,
restarts on crash, and keeps your logs.

My first attempt failed three times. The most useful lesson: systemd has no shell, so it
does not understand the shortcut for your home folder. Write every path out in full.

**PULL QUOTE**

Your program is a child of your login session. When the session ends, so does it.

**SECTION 3**

The next failure wastes days. Your app says it is listening. Nothing can reach it.

Three separate systems have to agree before a request arrives: the address your app is
listening on, the firewall on the machine, and the cloud provider's firewall, which is
invisible from inside the server. Any one can say no, and none of them tells you.

Read the shape of the failure, not the error. If the connection hangs and then times out,
a firewall silently dropped it. If it fails instantly with "connection refused", the
request reached your machine and nothing was listening. If it connects but nothing comes
back, your app is broken and the network was never the problem.

Three outcomes, three layers, and you can tell them apart in a second.

**SECTION 4**

Eventually you want a real address instead of a string of numbers, and a padlock in the
browser. Both are free: a domain from the GitHub Student Pack, a certificate from Let's
Encrypt.

Getting a certificate takes one command. Keeping one is where people fail. Mine renew
themselves now. For a while they did not, and I found out when a browser warned a friend.

**SIDEBAR BOX**

Four things that cost me an evening each

The free server is out of stock constantly. That error is temporary, not a rejection. Try
again at a different hour.

Every guide opens ports with a tool called ufw. It is not installed on this kind of server.

A folder marked private makes every file inside it unreadable, however the file itself is
set. Permissions stack downwards.

Never paste a config containing a password into a chatbot. You will export that
conversation later and commit it somewhere public. Ask me how I know.

**END**
