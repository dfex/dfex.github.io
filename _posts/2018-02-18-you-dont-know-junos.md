---
layout: post
title: You don't know Junos
description: "Strengthen your Junos-Fu"
modified: 2018-02-18
tags: [junos, cli]
---

The Junos Operating System is turning 20 next year (the First Release was July 7, 1998 according to Wikipedia) and still to this day it includes what is, in my opinion, the most powerful command line interface available on a networking device.

But with the current trend in networking to move towards fully automated, API-driven, SDN-orchestrated, Jinja2 templated nirvana, you’d be forgiven for thinking that switches will all ship with no CLI at all.

I’m a bit more of a realist and while I’m completely onboard with the benefits of network automation, I (along with what I suspect are most other Network Engineers out there) also don’t want to have to punch out 400 lines of J2 template code every time I need to test out a new feature, or re-create a customer issue in the lab.

So to start the ball rolling in 2018 with a *very* unfashionable topic, I’ve decided to put together a list of lesser-known Junos CLI magic that I’ve picked up over the years from other engineers, customers and pure serendipity that might help you.

#Test

##Test 2

###Test 3

## Protecting your assets

In the lab environment, or anywhere you are doing large scale automation it’s always best to err on the side of caution when making configuration changes.

To this end, I like to use the protect feature in Junos to lock specific sections of configuration that I don’t want accidentally (or deliberately) deleted - specifically management IPs, routes and last-resort local logins.

To implement protect, simply single out the sections of configuration you want to lock like so:

{% highlight html %}
{% raw %}
protect system login user backdoor
protect routing-options static route 0.0.0.0/0
protect interfaces fxp0
{% endraw %}
{% endhighlight %}

Now when you commit that configuration these sections of the configuration will be locked - observe:

{% highlight html %}
{% raw %}
{master:0}[edit]
root@qfx5100-1# delete
This will delete the entire configuration
Delete everything under this level? [yes,no] (no) yes

warning: [system login user backdoor] is protected, 'system login user maintenance' cannot be deleted
warning: [interfaces em0] is protected, 'interfaces em0' cannot be deleted
warning: [routing-options static route 0.0.0.0/0] is protected, 'routing-options static route 0.0.0.0/0' cannot be deleted
{% endraw %}
{% endhighlight %}

In order to delete or change these statements now, you need to unprotect them, commit, and then delete them and commit again:

{% highlight html %}
{% raw %}
unprotect system login user backdoor
unprotect routing-options static route 0.0.0.0/0
unprotect interfaces fxp0
commit
delete system login user backdoor
commit
{% endraw %}
{% endhighlight %}

This should slow down the annihilation when your PyEZ automation script becomes sentient!

## Parse like a Boss

Junos implements quite a few UNIX commands in the shell that are exceptionally useful when parsing log or configuration files.

For example: if you’re browsing through a configuration file and want to quickly move to a specific section of it, you can use / to perform a forward regex search eg: type show configuration and then /interfaces.

In the same manner, you can search backwards for a section you wish to return to using ? to perform a reverse regex match eg: ?system will take you back to the system stanza (or to the first match of the string “system”).

When it comes to logging, we all know how frustrating it is having to troubleshoot issues sifting through on-box syslog files that are filled with hundreds of unrelated event messages.

Well, along with the forward and backward regex matches described above, you can also update match and exclude filters in real-time to rapidly zoom in on precisely the entries you care about.

Here’s an example with a very verbose flow trace file on an SRX:

{% highlight %}
{% raw %}
bdale@0ffnet-srx210-gw> show log FLOW
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:flow process pak, mbuf 0x42e06780, ifl 0, ctxt_type 0 inq type 5

Jun 29 15:39:38 15:39:38.221829:CID-0:RT: in_ifp <junos-host:.local..0>

Jun 29 15:39:38 15:39:38.221829:CID-0:RT:flow_process_pkt_exception: setting rtt in lpak to 0x495f1fe8

Jun 29 15:39:38 15:39:38.221829:CID-0:RT:host inq check inq_type 0x5

Jun 29 15:39:38 15:39:38.221829:CID-0:RT:Using vr id from pfe_tag with value= 0

Jun 29 15:39:38 15:39:38.221829:CID-0:RT:Changing lpak->in_ifp from:.local..0 -> to:.local..0

Jun 29 15:39:38 15:39:38.221829:CID-0:RT:Over-riding lpak->vsys with 0

Jun 29 15:39:38 15:39:38.221829:CID-0:RT:  .local..0:172.16.10.254/22->172.16.10.22/56060, tcp, flag 18

Jun 29 15:39:38 15:39:38.221829:CID-0:RT: find flow: table 0x48924c18, hash 20942(0xffff), sa 172.16.10.254, da 172.16.10.22, sp 22, dp 56060, proto 6, tok 2

Jun 29 15:39:38 15:39:38.221829:CID-0:RT:Found: session id 0x188d. sess tok 2

Jun 29 15:39:38 15:39:38.221829:CID-0:RT:  flow got session.

Jun 29 15:39:38 15:39:38.221829:CID-0:RT:  flow session id 6285
{% endraw %}
{% endhighlight %}

Firstly, it’s all double-spaced which is hard to read, so let’s just focus on lines with “Jun” in them:

{% highlight html %}
{% raw %}
m Jun
{% endraw %}
{% endhighlight %}

{% highlight html %}
{% raw %}
Match for: Jun
Jun 29 15:39:38 15:39:38.221829:CID-0:RT: in_ifp <junos-host:.local..0>
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:flow_process_pkt_exception: setting rtt in lpak to 0x495f1fe8
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:host inq check inq_type 0x5
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:Using vr id from pfe_tag with value= 0
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:Changing lpak->in_ifp from:.local..0 -> to:.local..0
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:Over-riding lpak->vsys with 0
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:  .local..0:172.16.10.254/22->172.16.10.22/56060, tcp, flag 18
Jun 29 15:39:38 15:39:38.221829:CID-0:RT: find flow: table 0x48924c18, hash 20942(0xffff), sa 172.16.10.254, da 172.16.10.22, sp 22, dp 56060, proto 6, tok 2
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:Found: session id 0x188d. sess tok 2
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:  flow got session.
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:  flow session id 6285
Jun 29 15:39:38 15:39:38.221829:CID-0:RT: vector bits 0x2 vector 0x45ace0a0
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:mbuf 0x42e06780, exit nh 0x1f0010
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:flow_process_pkt_exception: Freeing lpak 0x489a9ad0 associated with mbuf 0x42e06780
Jun 29 15:39:38 15:39:38.221829:CID-0:RT: ----- flow_process_pkt rc 0x0 (fp rc 0)
Jun 29 15:39:38 15:39:38.222598:CID-0:RT:<172.16.10.22/56060->172.16.10.254/22;6> matched filter ALL:
Jun 29 15:39:38 15:39:38.222598:CID-0:RT:packet [52] ipid = 26605, @0x4232c51e
Jun 29 15:39:38 15:39:38.222598:CID-0:RT:---- flow_process_pkt: (thd 1): flow_ctxt type 15, common flag 0x0, mbuf 0x4232c300, rtbl_idx = 0
Jun 29 15:39:38 15:39:38.222598:CID-0:RT: flow process pak fast ifl 74 in_ifp vlan.10
Jun 29 15:39:38 15:39:38.222598:CID-0:RT:  vlan.10:172.16.10.22/56060->172.16.10.254/22, tcp, flag 10
Jun 29 15:39:38 15:39:38.222598:CID-0:RT: find flow: table 0x48924c18, hash 42742(0xffff), sa 172.16.10.22, da 172.16.10.254, sp 56060, dp 22, proto 6, tok 6
{% endraw %}
{% endhighlight %}

Now, let’s start culling entries that don’t appear to be useful like the ones mentioning vector and mbuf:

{% highlight html %}
{% raw %}
e vector
e mbuf
{% endraw %}
{% endhighlight %}

{% highlight html %}
{% raw %}
Match except (mbuf): vector
Jun 29 15:39:38 15:39:38.221829:CID-0:RT: in_ifp <junos-host:.local..0>
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:flow_process_pkt_exception: setting rtt in lpak to 0x495f1fe8
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:host inq check inq_type 0x5
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:Using vr id from pfe_tag with value= 0
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:Changing lpak->in_ifp from:.local..0 -> to:.local..0
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:Over-riding lpak->vsys with 0
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:  .local..0:172.16.10.254/22->172.16.10.22/56060, tcp, flag 18
Jun 29 15:39:38 15:39:38.221829:CID-0:RT: find flow: table 0x48924c18, hash 20942(0xffff), sa 172.16.10.254, da 172.16.10.22, sp 22, dp 56060, proto 6, tok 2
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:Found: session id 0x188d. sess tok 2
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:  flow got session.
Jun 29 15:39:38 15:39:38.221829:CID-0:RT:  flow session id 6285
Jun 29 15:39:38 15:39:38.221829:CID-0:RT: ----- flow_process_pkt rc 0x0 (fp rc 0)
Jun 29 15:39:38 15:39:38.222598:CID-0:RT:<172.16.10.22/56060->172.16.10.254/22;6> matched filter ALL:
Jun 29 15:39:38 15:39:38.222598:CID-0:RT:packet [52] ipid = 26605, @0x4232c51e
Jun 29 15:39:38 15:39:38.222598:CID-0:RT: flow process pak fast ifl 74 in_ifp vlan.10
Jun 29 15:39:38 15:39:38.222598:CID-0:RT:  vlan.10:172.16.10.22/56060->172.16.10.254/22, tcp, flag 10
Jun 29 15:39:38 15:39:38.222598:CID-0:RT: find flow: table 0x48924c18, hash 42742(0xffff), sa 172.16.10.22, da 172.16.10.254, sp 56060, dp 22, proto 6, tok 6
{% endraw %}
{% endhighlight %}

As you can see - as you add more matches and exceptions, your logs will gradually be pared down to more useful information.

As with most Junos commands, you can chain these commands using pipes as well and in real-time:

{% highlight html %}
{% raw %}
monitor start messages | except alarmd | except mgd
{% endraw %}
{% endhighlight %}

will give you realtime log output from all daemons except alarmd and mgd.

And finally, don’t forget about <ctrl-r> or reverse search in the CLI.

This allows you to do a quick regex search of your CLI history so you can find that long command you entered earlier without having to press up arrow 47 times!

{% highlight html %}
{% raw %}
{master:0} <ctrl-r>
(history search) '172': show route protocol static table inet.0 172.16.6.0/26 
{% endraw %}
{% endhighlight %}
 
## Realtime location tracking

Well, sort of…  Did you know that tucked away inside Junos is everybody’s favourite real-time traceroute implementation mtr?

It’s a fairly old version (v0.69) but the fact that you can run this from inside Junos is sometimes quite useful for looking at any instantaneous packet loss along a routed path.

To execute simply

{% highlight html %}
{% raw %}
traceroute monitor 8.8.8.8
{% endraw %}
{% endhighlight %}

and you’ll see some output like the following, updating in real-time.

{% highlight html %}
{% raw %}
                             My traceroute  [v0.69]
qfx5100-1 (0.0.0.0)(tos=0x0 psize=64 bitpattern=0x00)  Mon Jan 22 20:50:38 2018
Keys:  Help   Display mode   Restart statistics   Order of fields   quit
                                              Packets               Pings
 Host                                       Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. vlan9.axs1.comlinx.lab                   0.0%     7    1.9   2.5   1.9   2.9   0.4
 2. vlan10.core1.comlinx.lab                 0.0%     7    3.1   3.5   2.5   6.5   1.4
 3. ge-0-0-1.62rob.bne.comlinx.com.au        0.0%     7    1.0   1.3   0.9   2.7   0.6
 4. 10.1.254.5                               0.0%     7    1.6   1.5   1.5   1.6   0.0
 5. 10.1.254.6                               0.0%     7    2.1   1.9   1.8   2.1   0.1
 6. ge-0-0-4.100wic.bne.comlinx.com.au       0.0%     7    1.7   1.6   1.5   1.7   0.1
 7. gen-xxx-xxx-xxx-xxx.ptr4.otw.net.au      0.0%     7    2.2   3.2   2.2   8.1   2.2
 8. gexxx.xx.pe2.100wic.bne.core.otw.net.au  0.0%     7    2.5   2.5   2.2   2.9   0.2
 9. as15169.nsw.ix.asn.au                    0.0%     7   14.3  14.6  14.2  16.7   0.9
10. 108.170.247.65                           0.0%     7   15.0  14.9  14.7  15.0   0.1
11. 209.85.247.157                           0.0%     7   15.4  15.4  15.2  15.9   0.3
12. google-public-dns-a.google.com           0.0%     7   14.4  14.4  14.3  14.4   0.0
{% endraw %}
{% endhighlight %}

Be advised that traceroute monitor is only available from within inet.0 though, which does limit it’s usefulness - if anyone from Juniper engineering is reading this - I’d love to see a routing-instance option added!

## More Refreshments

And speaking of refreshing, another feature that doesn’t get much attention is the refresh action that you can pipe commands to.

What this does is re-run your previous command at the interval you specify, and time-stamp the output for you.

{% highlight html %}
{% raw %}
 show route 0.0.0.0/0 | refresh 5

{master:0}
root@qfx5100-1> show route 172.16.6.0/26 | refresh 5
---(refreshed at 2018-01-22 21:18:27 EST)---

inet.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

172.16.6.0/26      *[Static/200] 00:00:27
                    > to 10.0.9.1 via em0.0
---(refreshed at 2018-01-22 21:18:32 EST)---
inet.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

172.16.6.0/26      *[Static/200] 00:00:32
                    > to 10.0.9.1 via em0.0
---(refreshed at 2018-01-22 21:18:37 EST)---
inet.0: 8 destinations, 9 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

172.16.6.0/26      *[OSPF/150] 00:00:05, metric 0, tag 0
                    > to 192.168.100.1 via xe-0/0/51:0.0
                    [Static/200] 00:00:39
                    > to 10.0.9.1 via em0.0
{% endraw %}
{% endhighlight %}
This is super helpful when troubleshooting intermittent issues - the results can even be piped to a file.
Much better than standing by your console over night pressing Up-Arrow / Enter, and much less dangerous than setting up a drinking bird in your Data Centre.

![Drinking Bird]({{ site.url }}/images/drinking-bird.gif)

### And now for breadcrumbs

Another relatively unknown feature is configuration-breadcrumbs.  This appeared around Junos 12.2 and displays the configuration hierarchy (breadcrumbs) of the currently displayed configuration output:

This is super useful if you’re examining a large configuration file on box and need to keep track of which stanza you’re under (especially subscriber detail on a BRAS, or deep configuration under a routing-instance)

{% highlight html %}
{% raw %}
{master:0}
bdale@qfx5100-2> show configuration
…
routing-instances {
    CRUMMY {
        instance-type virtual-router;
        routing-options {
            autonomous-system 65500;
        }
        protocols {
            bgp {
                group UPSTREAM {
                    neighbor 192.168.5.4 {
                        export LOOPBACK;
---(more 97%)---[routing-instances CRUMMY protocols bgp group UPSTREAM neighbor 192.168.5.4]---
{% endraw %}
{% endhighlight %}

To enable, you need to configure it under your login class:

{% highlight html %}
{% raw %}
 set system login class onLikeDK permissions all
 set system login class onLikeDK configuration-breadcrumbs
{% endraw %}
{% endhighlight %}

and then log back in again.

## Are you committed?

This is one that I was shown by a customer very recently:

When you perform a commit confirmed the configuration will be applied and then Junos will wait for a follow-up commit before it removes the rollback timer.

On large clustered Junos deployments such as EX virtual chassis, or Branch SRX Chassis Clusters with large configurations, running this follow-up commit can take quite a bit of time as your configuration is re-checked and applied against each VC member, even though nothing is changing.

It is especially daunting when you commit confirmed for only a couple of minutes or so, and then realise your testing will actually take all of that time.

It turns out that by running a commit check instead, Junos will re-validate the configuration (against the REs only) and then remove the rollback timer, saving what might be precious seconds or minutes during a change window.

## That’s just the Tip of it

Well, that’s about it for this post - hopefully you’ve picked up at least one new trick that will make your Junos CLI-fu that bit stronger.

For even more CLI tips, Junos includes a whole bunch built in - type

{% highlight html %}
{% raw %}
help tip cli
{% endraw %}
{% endhighlight %}

for a random one, or 

{% highlight html %}
{% raw %}
help tip cli [0-98]
{% endraw %}
{% endhighlight %}

to read through them all independently - you can even give console/SSH users a tip of the day by adding the 

{% highlight html %}
{% raw %}
set system login class helpful login-tip
{% endraw %}
{% endhighlight %}

to their login class.

Long live the CLI!