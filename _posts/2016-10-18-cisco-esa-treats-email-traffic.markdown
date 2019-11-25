---
layout: post
title:  "How Cisco ESA treats email traffic"
date:   2016-10-18 15:04:00 +0100
categories: security
---
I was working with Cisco ESA (previously - Ironport) lately and would like to write down some notes about how it works (*at least the basics*). I\'ve already covered [spam and antivirus testing techniques](https://askbow.com/2016/08/13/test-email-antispam-antivirus-work/) in a previous post. Here I\'ll try to walk through the message filtering steps performed by a Cisco ESA appliance.

### E-mail traffic general description

First I think it is logical to define <<**email traffic**>>: it is a flow of messages attributed to email Internet service transferred over [SMTP](https://www.ietf.org/rfc/rfc2821.txt) between servers called Mail Transfer Agents (MTA). In that sense, an ESA appliance (be it physical or virtual) works as a frontend MTA between the world (global Internet) and inner (enterprise) mail servers (Lotus, Exchange, etc.).

In a general case, when an MTA receives a message to be transferred, it performs [routing not unlike *in principo* to that of IP](https://askbow.com/2015/04/06/general-ip-routing/): there is even a sort-of default routing, based on DNS. MTA sends a DNS request for the destination domain, asking for any MX-type records, which supposedly define the MTAs admins assigned to receive mail for that domain. For example, here\'s what it looks like for my domain:

```
<pre class="">~$ dig MX @8.8.8.8 askbow.com +short
10 mail1.namebrightmail.com.
20 mail2.namebrightmail.com.
```

Recursively, the DNS subsystem resolves these records through the respective A-records into IP addresses. The `10` and `20` at the beginning of the lines show priority (though, I would argue that it, semantically , is a *cost*, as *lower* means *better*) . The sending MTA selects the next destination MTA according to this priority. Our MTA opens a connection to destination MTA on TCP port 25 (SMTP) and starts a dialogue like this (*domain names changed to protect the innocent*):

```
<pre class="">220 esa2.example.com ESMTP
helo
250 esa2.example.com
mail from: <test@example.com>
250 sender <test@example.com> ok
rcpt to: <test@example.com>
250 recipient <test@example.com> ok
data
354 go ahead
subject: TEST
test-1
test-2
.
250 ok:  Message 2331 accepted
quit
221 esa2.example.com
```

(*note that during one dialogue we can technically send several messages*)

#### SMTP reliability

Ok, but what happens if the first MTA we tried to contact wasn\'t available? In that situation, the sending MTA acts according to the RFC and thus MUST try to deliver mail through the next destination MTA listed in ascending priority order. And it should continue attempts for four days. As a result, mail delivery in general is pretty robust: it takes a very long outage to prevent mail from being delivered.

> *N.B.: it\'s interesting that some spammers\' software won\'t follow RFC in that respect and hence won\'t try to deliver the message later, or even won\'t try the next server; clearly this works for them because of their economics: it\'s more profitable to try next target in a million-address list faster than to make the delivery reliable. There is an anti-spam technique which utilizes this, but it shouldn\'t be your only protection.*

The same is true in case the *frontend* MTA (i.e. ESA, or Barracuda, or your good old postfix, or whatever) can\'t connect to the backend mail system: it will queue the messages for several days, attempting delivery from time-to-time. The default queueing time for Cisco ESA is 72 hours; when this time elapses, the ESA sends a notification to the sender, telling about inability to deliver.

### What Cisco ESA does with this email message?

In the previous section I looked at some general principles which work for everyone. Now let\'s dig deeper into Cisco ESA, how it filters spam and works with viruses.

#### Antispam filtering

Antispam filtering in Cisco ESA works in layers and begins with host reputation. This step alone removes about 80-96% of traffic which would otherwise hit more (computationally) costly procedures or even backend mail servers.

Reputation data comes from [Senderbase](https://www.senderbase.org/) which presents it as an integral score between -10.0 (*most definitely spam*) to +10.0 (*definitely not spam*). The default behaviour for the ESA is to completely cut (deny) all traffic coming from hosts with the reputation closest to the low end. The top-end traffic can be set to bypass all further antispam checks (*but personally I prefer to still check everything, for reasons I describe below*). The system throttles the middle-class traffic (*by limiting the amount of messages per dialogue*) and checks it further down the line with antispam engines.

#### A short note on Senderbase

It\'s worth it to understand where the Senderbase score comes from.

Cisco claims that this database integrates data on 30% of world-wide email and web traffic. This is a lot, and the volume brings statistical significance to the table, giving a strong foundation to any judgements they infer about the whole traffic.

Clearly we can\'t judge just by traffic volume coming from a host: a spammer usually sends a lot, but then so does a legitimate subscription or news server.

To lower the number of false-positives, they use additional data available: server\'s location (*country*), end-user feedback (*the users can send it through Cisco-owned [spamcop service](https://www.spamcop.net/) or with an MS Outlook plugin*), other blacklists (*SpamHaus, Abuseseat, SORBS, TrendMicro, etc*). They also integrate data from spam traps, presence of non-existent recipients in the envelopes of the traffic associated with the host, and IP address space hijacking attempts.

#### Antispam filtering continued

Senderbase reputation filtering is effective, but just like any defence, it can be circumvented. For example, spammers may use the following tactic: they would buy a large block of IP space and for some time use it for *legit* traffic, hence getting a good reputation. After that they start sending spam. *That\'s why* I prefer to check everything with antispam, as I said before. The engine in the heart of Cisco ESA is called Context Adaptive Scanning Engine (CASE), which includes a Bayesian filter.

CASE, according to the documentation available, works through message\'s metadata which forms so-called context (*who-when-where etc*). Although some of the code created by [Ironport team is opensourced](https://github.com/ironport), the engine is not.

Anyway, the filtering system checks every bit of information available - message body and headers, the sender\'s identity and recipients. Everything to tell Spam from Ham. Well, it does a bit more: it determines spam, ham, suspicious, marketing and bulk (i.e. sent to thousands). The later can be further classified as general bulk messages and social network-sourced.

> *N.B.: Indeed, I remember reading a few years ago a report that said that social network\'s combined volume rivals that of spam, because Facebook and the likes send a notification for every move our acquaintances make online.*

Cisco ESA can drop the messages of a certain class, mark them (by adding to the subject [SPAM], [SUSPECTED SPAM], [MARKETING], [SOCIAL NETWORK], or [BULK] , respectively; this is the default behaviour). The system is pretty flexible and allows some very deep modifications to the messages.

#### Antivirus scanning and filtering - Outbreak Filters

Cisco ESA integrates several stages of antivirus protection. To start with, it employs Senderbase again, but this time it provides data about encountering certain payloads - so called Outbreak Filters. In it\'s essence, it is a sort of Collective Defence mechanism. When the network (*i.e. the data analytics inside Cisco Talos group, who receive the data from the network*) detects a mass distribution of, say, some file, all traffic containing such file is throttled and put to a special dynamic quarantine.

[![Cisco](https://askbow.com/wp-content/uploads/2016/10/cloud-esa-outbreakfilter-1024x422.png)](https://askbow.com/wp-content/uploads/2016/10/cloud-esa-outbreakfilter.png)

> *N.B. careful examination of Cisco ESA configuration file reveals other means of collective antivirus defence, namely [Immunet ](https://www.immunet.com/)(owned by Cisco) to be present somewhere under the hood - but there is no mention of it directly in the interface or documentation. It is possible that it is a part of the AMP license (sold separately from the main ESA features).*
> 
> *I actually tried using the free version of Immunet for several months. They lost me when at some point it started quarantining system files, killing my Windows installation. Virustotal showed nothing on the very same files and it took me some time to restore things back to normal.*

The Outbreak Filters are there to slow down or stop entirely the spreading of potential malware, to give time to AV labs to study it, confirm its malevolence and prepare signatures. [Here](https://www.senderbase.org/static/malware/) you can see that senderbase is usually the fastest to react, with lead time in the range of several hours to a day before AV signatures are ready.

[![Senderbase](https://askbow.com/wp-content/uploads/2016/10/screenshot-www.senderbase.org-2016-10-18-11-30-57-300x274.png)](https://askbow.com/wp-content/uploads/2016/10/screenshot-www.senderbase.org-2016-10-18-11-30-57.png)

The default action for outbreak filters is to delay a potential threat for a day. Other types of filtered messages are delayed up to 4 hours. *Other* here refers to messages containing links to suspicious websites, and also potential phishing attempts and scam. These are further marked with the [SUSPICIOUS MESSAGE] tag to warn the users.

If the Senderbase threat level is high, ESA can also modyfy the message body, prepending it with a friendly notice, warning the user to be cautious and not to open files or links. Here\'s what the default message looks like:

[![Cisco](https://askbow.com/wp-content/uploads/2016/10/screenshot-ironport-2016-10-18-11-49-27.png)](https://askbow.com/wp-content/uploads/2016/10/screenshot-ironport-2016-10-18-11-49-27.png)

#### Antivirus scanning and filtering - AV engines

The Outbreak Filters are pretty effective as the first line of defence against many Zero-Day attacks, but they don\'t replace more general antivirus checks. There are two kinds of antivirus that can be used with Cisco ESA (e.g. those that you can by a subscription license for; as already mentioned, there seems to be traces of Immunet (*which might or might not act as an antivirus here; for all I know, it is used for AMP only*)):

- [Sophos](https://www.sophos.com/)
- [McAffee ](https://www.mcafee.com)

You can use both or any one at the same time if necessary. Both engines are pretty effective at what they do, and they include signature-based analysis as well as sandboxing and sophisticated heuristics.

> *Personally, I don\'t think running two AVs on the same box is a good idea. I would prefer to run a secondary AV solution closer to my users, say on the internal mail servers themselves. The front-end MTA must be as fast as possible, and the AVs are resource-consuming, which works against performance. But it\'s only an opinion.*

By default, ESA would try to make the AV to remove the malware if it was found. If successful, the message is marked with [WARNING: VIRUS REMOVED]. Otherwise the message should be dropped.

Given that there are messages that can\'t be scanned for legitimate reasons, there are two further steps. If the message contains a file that can\'t be scanned (for example, it is too big), the message is marked with [WARNING : A/V UNSCANNABLE]. If the reason is that the contents is encrypted, the tag used is [WARNING : MESSAGE ENCRYPTED]. This at least tries to warn the end-user of a potential threat.

### Message flow walkthrough

If I look into message tracking of my ESA appliance, I can find the message I\'ve sent earlier in the post:

```
<pre class="">Message 4242 matched per-recipient policy DEFAULT for inbound mail policies.
Message 4242 scanned by Anti-Spam engine: CASE. Interim verdict: Suspect
Message 4242 scanned by Anti-Spam engine: CASE. Final verdict: Suspect
Message 4242 scanned by Anti-Virus engine Sophos. Interim verdict: CLEAN
Message 4242 scanned by Anti-Virus engine. Final verdict: Negative
Message 4242 scanned by Outbreak Filters. Verdict: Negative
Message 4242 queued for delivery.
```

*Well, I didn\'t expect much for an empty message.*

#### First steps - the connection

In this chain of events, after we establish an SMTP session, ESA checks us (*the connecting host*) against the Host Address Table (HAT). This table maps several static lists (white, black, etc.) and Senderbase reputation ranges to access policies. These contain a lot of details on how to treat the connecting host (including the ability to override SMTP status codes), most importantly the action: Reject, Accept or Relay. The later is used to put the traffic from the trusted hosts to an expedited queue.

My message from the example above fell into the DEFAULT policy and the ESA Accepted it for further checks.

Next comes pre-processing: the system inserts special headers in the message and runs some optional tests. For example, if it is integrated into your internal MS ActiveDirectory, ESA will check the validity of the recipients at this stage. If the recipient is not found, the system silently drops the message to protect against the [Directory Harvest attacks](https://en.wikipedia.org/wiki/Directory_Harvest_Attack).

On the next step, the system applies message filters. This is a very flexible filtering engine which allows the administrator a great lot of customisation, if needed. From the point of view of this post, this filter is interesting mostly because it happens to be the last to be applied to any message which was sent to many recipients at once (say, copy or multiple addresses in the rcpt field). For the purposes of later checks, the ESA will logically split the message per-recipient.

> *The reason for this is that you can have very different policies per-recipient, concerning the main sections of the incoming mail processing: antispam and graymail, antivirus, content filtering, etc.*
> 
> *I use this property to experiment with the options, modifying only the handling of the messages I send to myself. The traffic that goes to the rest of the company remains intact and follows internal security guidelines.*

#### Per-recipient checks

For the next step, end-users can make their own **personal** white and black lists via the quarantine interface. End-users can\'t work around antivirus filtering, but they can make the message to bypass antispam engine entirely. This is actually empowering and it gives the users just enough control without compromising security.

After the message went through the personal filters, the ESA runs it against the previously discussed antispam and antivirus engines. No need to repeat the description here.

Next, the system checks if the message is graymail (the marketing-bulk-social network part). I think it is pretty intensive computationally, that\'s why it runs one of the last. The content filtering follows, with all its power of regular expressions and matching whatever your heart desires. It can be used as a sort of [policy routing](https://askbow.com/2015/04/13/policy-based-routing-ip/), because you can override SMTP routing here.

Lastly, the Outbreak Filter module checks the message and quarantines accordingly, as I described before.

Finally, the message that made it through all the filters (such as the message in my example) is placed on the delivery queue.

### Conclusive thoughts

Overall, Cisco ESA seems like a robust solution based on a mature algorithm. Don\'t get me wrong here, there are plenty of ways to shoot yourself in the foot with it (i.e. get your mail dropped). And there is a lot of cool checks included I didn\'t describe. For example, you can test PTR records, perform SPF, DKIM and DMARK checks. These all are pretty effective, but fail at one thing: rarely you can find them setup correctly.

My practice shows that acting on them results in higher-than-usual false-positives. That\'s why I usually would leave them turned off in production and recommend others do the same. At least with Cisco ESA, other mechanics are enough to protect the users.