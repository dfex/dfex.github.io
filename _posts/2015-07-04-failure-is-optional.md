---
layout: post
title: Failure is optional - Reth Interfaces and LACP
description: "SRX Reth Interfaces and LACP"
modified: 2015-07-04
tags: [junos, srx, lacp, reth, chassis-cluster, redundancy-groups]
---

I've been building and deploying Juniper SRX Firewall clusters for a good 6 years now and even managed to pick up a JNCIE-SEC along the way, but last week I stumbled across an interesting configuration feature when using LACP and Reth interfaces that I'd never seen documented before.

Let's start with a quick primer on SRX Redundant Ethernet (Reth) Interfaces and LACP:

Firstly, one or more physical ports from each SRX chassis-cluster node are assigned to a Reth interface:
{% highlight html %}
{% raw %}
ge-0/0/4 {
    gigether-options {
        redundant-parent reth4;
    }
}
ge-5/0/5 {
    gigether-options {
        redundant-parent reth4;
    }
}
ge-5/0/4 {
    gigether-options {
        redundant-parent reth4;
    }
}
ge-5/0/5 {
    gigether-options {
        redundant-parent reth4;
    }
}
{% endraw %}
{% endhighlight %}

Under the reth interface we configure LACP:

{% highlight html %}
{% raw %}
reth4 {
    redundant-ether-options {
        redundancy-group 4;
        minimum-links 1;
        lacp {
            active;
        }
    }
    unit 0 {
        description INTERNET;
        family inet {
            address 203.24.22.50/29;
        }
    }
}
{% endraw %}
{% endhighlight %}

Behind the scenes this creates two distinct LACP sub-LAGs, one from each physical SRX node to the downstream device:

![SRX Reth LACP Topology]({{ site.url }}/images/reth-lacp-topology.png)

The ```minimum-links``` option is so that each LACP sub-LAG is considered to be up while ever there is still at least one port active.

Finally we assign the Reth to a redundancy-group that specifies the "Primary" node on which interfaces in the Reth will be active by way of the ```priority``` (higher being more preferred), and optionally ```preempt``` fails the RG back to the primary node when it becomes available.

{% highlight html %}
{% raw %}
redundancy-group 4 {
    node 0 priority 100;
    node 1 priority 50;
    preempt;
}
{% endraw %}
{% endhighlight %}

Something that isn't immediately obvious to newcomers is that given the above configuration, if both ge-0/0/4 and ge-0/0/5 are unplugged from the primary node, the minimum-links threshold will be crossed and the reth on the primary node will go down, however the redundancy-group will **NOT** fail over:

{% highlight html %}
{% raw %}
bdale@srx-lab-fw1# run show interfaces terse ge-[05]/0/[45]
Interface               Admin Link Proto    Local                 Remote
ge-0/0/4                up    down
ge-0/0/5                up    down
ge-5/0/4                up    up
ge-5/0/5                up    up


bdale@srx-lab-fw1# run show lacp interfaces
Aggregated interface: reth4
    LACP state:       Role   Exp   Def  Dist  Col  Syn  Aggr  Timeout  Activity
      ge-0/0/4       Actor    No   Yes    No   No   No   Yes     Fast    Active
      ge-0/0/4     Partner    No   Yes    No   No   No   Yes     Fast   Passive
      ge-0/0/5       Actor    No   Yes    No   No   No   Yes     Fast    Active
      ge-0/0/5     Partner    No   Yes    No   No   No   Yes     Fast   Passive
      ge-5/0/4       Actor    No   Yes    No   No   No   Yes     Fast    Active
      ge-5/0/4     Partner    No   Yes    No   No   No   Yes     Fast   Passive
      ge-5/0/5       Actor    No   Yes    No   No   No   Yes     Fast    Active
      ge-5/0/5     Partner    No   Yes    No   No   No   Yes     Fast   Passive
    LACP protocol:        Receive State  Transmit State          Mux State 
      ge-0/0/4            Port disabled    No periodic           Detached
      ge-0/0/5            Port disabled    No periodic           Detached
      ge-5/0/4            Current         Fast periodic          Collecting Distributing
      ge-5/0/5            Current         Fast periodic          Collecting Distributing


bdale@srx-lab-fw1# run show chassis cluster interfaces
....
Redundant-ethernet Information:     
    Name         Status      Redundancy-group
    reth0        Down        Not configured   
    reth1        Down        Not configured   
    reth2        Down        Not configured   
    reth3        Down        Not configured   
    reth4        Down        4                
    reth5        Up          5                
    reth6        Up          6                
    reth7        Down        Not configured   
...

bdale@srx-lab-fw1# run show chassis cluster status
Cluster ID: 1 
Node                  Priority          Status    Preempt  Manual failover

Redundancy group: 0 , Failover count: 0
    node0                   100         primary        no       no  
    node1                   50          secondary      no       no  

Redundancy group: 4 , Failover count: 0
    node0                   100         primary        yes      no  
    node1                   50          secondary      yes      no  

Redundancy group: 5 , Failover count: 0
    node0                   0           primary        yes      no  
    node1                   0           secondary      yes      no  

Redundancy group: 6 , Failover count: 0
    node0                   0           primary        yes      no  
    node1                   0           secondary      yes      no  
{% endraw %}
{% endhighlight %}

I will digress here for a moment and say that to this day, I still can't think of a single reason why this behaviour is ever desirable.  If you've got a topology that benefits somehow from completely losing a reth without failing over, I want to know about it! :) 

Each redundancy group has an in-built Threshold counter which determines when fail-over to the secondary node will occur - this value is set to be 255 under normal conditions.

Looking at our redundancy-group again, we now add in the ```interface-monitor``` statements, which specify a ```weight``` against physical interfaces.  

{% highlight html %}
{% raw %}
redundancy-group 4 {
    node 0 priority 100;
    node 1 priority 50;
    preempt;
    interface-monitor {
        ge-0/0/4 weight 128;
        ge-0/0/5 weight 128;
        ge-5/0/4 weight 128;
        ge-5/0/5 weight 128;
    }
}
{% endraw %}
{% endhighlight %}

Now whenever any of the four interfaces listed above goes down, their ```weight``` will be subtracted from the Redundancy-group threshold; when this threshold reaches 0, the redundancy-group will fail over and activate interfaces associated with reths associated with this redundancy group on the secondary node.

It should be noted that the physical interfaces being monitored don't have to be members of a Reth interface associated with this redundancy-group.

You can see the results of this using the hidden-until-recently command ```show chassis cluster information```

{% highlight html %}
{% raw %}
bdale@srx-lab-fw1# run show chassis cluster information 
node0:
--------------------------------------------------------------------------
Redundancy mode:
    Configured mode: active-active
    Operational mode: active-active

Redundancy group: 0, Threshold: 255, Monitoring failures: none
    Events:
        Jun 16 17:28:31.245 : hold->secondary, reason: Hold timer expired
        Jun 16 17:28:32.093 : secondary->primary, reason: Better priority (1/1)

Redundancy group: 4, Threshold: 255, Monitoring failures: none
    Events:
        Jun 18 03:28:54.773 : hold->secondary, reason: Hold timer expired
        Jun 18 03:28:54.809 : secondary->primary, reason: Remote yield (0/0)
{% endraw %}
{% endhighlight %}

In the example above, I've deliberately used a ```weight``` of 128 so that a single link loss will not cause fail-over, instead requiring that both links to a node go down before failing over - this configuration achieves the same thing as configuring ```minimum-links 1``` in the LACP bundle, except that it actually causes the fail-over to occur in a redundancy-group.

This seems somewhat (wait for it...) redundant to me. 

What I recently discovered however was that you can configure interface-monitor to monitor the Reth interface instead of the physical links that make it up eg:

{% highlight html %}
{% raw %}
redundancy-group 4 {
    node 0 priority 100;
    node 1 priority 50;
    preempt;
    interface-monitor {
        reth4 weight 255;
    }
}
{% endraw %}
{% endhighlight %}

With this deployed, if the LACP sub-LAG bundle falls below ```minimum-links```, it is taken down as before, but now ```interface-monitor``` will detect this and fail the redundancy-group over.

As an added bonus, the interface-monitor now has a dependency on the downstream device to be an active LACP participant, rather than just monitor physical link status - think of it as free BFD!