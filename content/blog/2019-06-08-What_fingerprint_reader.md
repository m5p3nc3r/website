---
layout: post
title: What Fingerprint Reader?
---

I have been struggling with a rather odd problem on my laptop since updating to Fedora 30. Everything had been going fine for a number of weeks, but all of a sudden anything that required a login was taking a /very/ long time. This included the initial login and sudo, which was extremely annoying!

Long story short, a quick look at the output of

```bash
strace -s 1024 su
```

showed that it was timing out on a dbus call to an Fprint service.  A quick check of /etc/pam.d/system-auth showed that a fingerprint was a valid login method.

Now, my laptop does not have a fingerprint reader, so a quick run of

```bash
sudo authconfig --disablefingerprint --updateall
```

and all is well with the world (after a huge wait for sudo to try and find the fingerprint reader!).
