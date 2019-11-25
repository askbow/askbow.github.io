---
layout: post
title:  "How do I test if my email antispam and antivirus work?"
date:   2016-08-13 13:54:00 +0100
categories: security
---
It might come in a greenfield antispam / antivirus deployment or during an audit that one needs to make sure that the protection (against spam or viruses) is enabled (N.B.: *measuring protection efficiency is a completely different problem*) and the configured policies are applied as expected.

As with any program testing, for that task we need sample input data. In particular, there are two ASCII strings that should trigger antispam or antivirus response respectively, and then there are vendor-specific ways to test.

### EICAR - antivirus test file

The antivirus test file is perhaps the most well-known of the two: [EICAR](https://www.eicar.org/86-0-Intended-use.html) (there is a number of links to ready for consumption test files on their download page). Its contents is this:

###### X5O!P%@AP[4\\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*

Simply save it into an empty file and attach to your test email. Try also putting it as the only body of the email.
*N.B. immediately as I\'ve put this string into my text editor, the antivirus on my laptop quarantined the editor\'s temp file.*

### GTUBE - spamassasin test string

Finding a similar test string for antispam is more complex. There is one, of course: the [GTUBE](https://spamassassin.apache.org/gtube/), but unlike antivirus test listed above, I\'m not sure if every antispam solution supports it! (I\'ve tested it with Cisco [*Ironport*] ESA, it works). Here it is:

###### XJS*C4JDBQADN1.NSBN3*2IDNEN*GTUBE-STANDARD-ANTI-UBE-TEST-EMAIL*C.34X

Just put it into the email message body and send (preferably from some public host, not your enterprise network) to some target protected by the protection system you are testing.

### Cisco ESA-specific tests

With specific systems there come specific means to test, provided by the developers. For example, Cisco ESA docs ([TAC document here](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117865-qanda-esa-00.html)) says that to test antispam policies you can insert one of the following headers in your email message:

###### X-Advertisement: Suspect
 X-Advertisement: Spam
 X-Advertisement: Marketing

Note that it\'s a little harder, at least with some email clients, to insert headers, so the GTUBE string is very useful. These, it seems, are there to trigger specific parts of policies in ESA. Other antispam vendors are sure to have their own version of these (to test their specific capabilities).

Obviously, ESA has policy-tracing tools inside it, but nothing beats an outside black-box testing by sending an email from some public server to see how it\'s actually treated.

### Final thoughts on testing

While looking it all up I came across some other standard testing objects that might come useful in networking. For example, when we test VoIP and teleconferencing, TV test cards ([PM5644 - a fullHD test](https://upload.wikimedia.org/wikipedia/commons/thumb/b/b6/PM5644-1920x1080.gif/1280px-PM5644-1920x1080.gif)), and [phrases to pronounce](https://www.cs.columbia.edu/~hgs/audio/harvard.html) (wish there was such a collection in other languages) might come useful for formal testing / acceptance procedures.