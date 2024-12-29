# stole this from here: http://fellownerd.org/2021/edgerouter-config-dualwan/

Configuration
Initial Planning
WAN Performance Disparity
If you don’t have good, recent speed tests from your provider, now would be a great time to take them. Plug directly into each device from your provider and run an appropriate speed test. This is important for some of the configurations in part 2 so make sure you note the results. I recommend running the test a few times (and against different servers) and averaging your rresults for download speed (as this is what we’ll be concerned with later).

Port Definitions
For the purposes of this article, we’ll be using the following port assignments:

eth0 - WAN 1
eth1 - LAN
eth2 - WAN 2
You should ensure that your WAN connections are configured and working properly on the ports of your choice on your device. Adjust the configuration examples I provide appropriately for your setup.

Configuration Mode
The examples below will assume you are in configuration mode on your device. To enter configuration mode, type configure and press enter from your SSH/console session.

Configure a Basic Load-Balance Group
Start by configuring a load-balance group on your device that will provide the parameters for how the balancing should be done. We’re going to start with a very basic configuration:

set load-balance group G interface eth0 \
set load-balance group G interface eth2 \
set load-balance group G lb-local enable \
set load-balance group G sticky dest-addr enable \
set load-balance group G sticky source-addr enable

Let’s walk through each of these with an explanation:

set load-balance group G interface <interface>
This command is simply adding the interface to the group as a target. We’ll go back and define more here later.
set load-balance group G lb-local enable
This will tell our router to use this load-balance group for its own traffic as well. This ensures that things like dns-forwarding get balanced.
set load-balance group G sticky <type> enable
This set of rules (where we used dest-addr and source-addr) ensures that there is a relationship between which interface is used for traffic based on what has been sent previously. I chose to tie the dest-addr (on the Internet) and source-addr (on the LAN) together so that we could ensure that things like WebRTC will behave properly.
Basically anything that requires you to connect to multiple ports on the same remote address (some VPNs) has the opportunity to misbehave without this.
Setup Address Exclusions
Even though we will plan to load-balance most traffic, there are certain things we shouldn’t plan to load-balance. Amongst those things is our LAN network.

Create a network group that represents your LAN subnet (such as 192.168.1.0/24):

set firewall group network-group PRIVATE_NETS <your LAN subnet>
Setup Firewall Modify Rules
Now we need to specify the policy that will cause load-balancing to occur on the interface where it is assigned. Here’s the configuration we should put in place:

set firewall modify LOAD_BALANCE rule 10 action modify \
set firewall modify LOAD_BALANCE rule 10 destination group network-group PRIVATE_NETS \
set firewall modify LOAD_BALANCE rule 10 modify table main

set firewall modify LOAD_BALANCE rule 20 action modify \
set firewall modify LOAD_BALANCE rule 20 destination group address-group ADDRv4_eth0 \
set firewall modify LOAD_BALANCE rule 20 modify table main

set firewall modify LOAD_BALANCE rule 30 action modify \
set firewall modify LOAD_BALANCE rule 30 destination group address-group ADDRv4_eth2 \
set firewall modify LOAD_BALANCE rule 30 modify table main

set firewall modify LOAD_BALANCE rule 100 action modify \
set firewall modify LOAD_BALANCE rule 100 modify lb-group G

Let’s walk through each of these with an explanation:

set firewall modify LOAD_BALANCE rule <number> action modify
This instructs the rule to perform a modification on a routing table.
set firewall modify LOAD_BALANCE rule <number> destination group <specification>
This instructs the rule on when to be applied. In this instance, we’re saying with rules 10-30 that we want to be applied when the traffic is bound for either PRIVATE_NETS or the IPv4 address of eth0 or eth2.
set firewall modify LOAD_BALANCE rule <number> modify table main
This rule should modify the main routing table.
set firewall modify LOAD_BALANCE rule <number> modify lb-group G
After the preceeding rules, use the load-balance group G that we created earlier for the remaining traffic.
At this point in time, you have a basic load-balancing configuration. You should commit the configuration at this point but you’ll likely find some odd behaviors in the future. We’ll cover additional configuration in part 2.

Verify Load Balancing
You can validate that your load balancing is working properly using a few commands. First, to see the general status of the interfaces and determine their status, use this command (outside of configure mode):

show load-balance watchdog

You’ll receive a response similar to this:

Group G
  eth0
  status: OK
  pings: 19998
  fails: 63
  run fails: 0/2
  route drops: 1
  ping gateway: 8.8.8.8 - REACHABLE
  last route drop   : Tue Feb  2 01:54:07 2021
  last route recover: Tue Feb  2 02:05:27 2021

  eth2
  status: OK
  pings: 19979
  fails: 64
  run fails: 0/2
  route drops: 1
  ping gateway: 1.1.1.1 - REACHABLE
  last route drop   : Tue Feb  2 01:54:07 2021
  last route recover: Tue Feb  2 02:05:16 2021
From this screen, we can easily see that both of our interfaces are operational with pings running properly.

Next, we’ll show the flow of traffic across the load-balancing to ensure that our traffic is actually being balanced:

show load-balance status

You’ll receive a response similar to this:

Group G
    Balance Local  : true
    Lock Local DNS : false
    Conntrack Flush: true
    Sticky Bits    : 0x00000003

  interface   : eth0
  reachable   : true
  status      : active
  gateway     : <hidden>
  route table : 201
  weight      : 50%
  fo_priority : 100
  flows
      WAN Out   : 291K
      WAN In    : 0
      Local ICMP: 20019
      Local DNS : 0
      Local Data: 42013

  interface   : eth2
  reachable   : true
  status      : active
  gateway     : <hidden>
  route table : 202
  weight      : 50%
  fo_priority : 100
  flows
      WAN Out   : 382K
      WAN In    : 481
      Local ICMP: 20001
      Local DNS : 0
      Local Data: 17728
Specifically, the field under flows show the traffic on each interface. In my own experience, it can take some time for the balancing to be realized because of existing sessions. I usually just check back the next day to see balancing.

Summary
You now have a functional, generic load-balancing dual-WAN configuration for your EdgeOS device. If left in this configuration, you’ll probably start to notice little problems here and there that cause you with some service issues or frustration. Continue into part 2 to explore additional configurations that you should make to round-out your experience. Thanks for your time!


Configurations
Warnings
In my original article, I share a list of things that break when you implementing load-balancing on EdgeOS. You should have a look at these items and make an evaluation on the usefulness of this configuration based on what I share.

Change Default Route-Test Behavior
Background
By default, EdgeOS will ping your default gateway to determine the availability of a given WAN interface. This sounds good hypothetically but, in practice, doesn’t work as well. Doing this doesn’t protect against the scenario of an upstream failure within the provider’s network (which, with Comcast, is a common problem). Instead, we want to pick a public destination that indicates that we’re ready for “common service” and test this.

Implementation
Here are the configuration sections we’re going to add for each WAN interface:

set load-balance group G interface eth0 route-test type ping target 8.8.8.8 \
set load-balance group G interface eth0 route-test count failure 2 \
set load-balance group G interface eth0 route-test count success 2 \
set load-balance group G interface eth0 route-test initial-delay 10 \
set load-balance group G interface eth0 route-test interval 10

set load-balance group G interface eth2 route-test type ping target 1.1.1.1 \
set load-balance group G interface eth2 route-test count failure 2 \
set load-balance group G interface eth2 route-test count success 2 \
set load-balance group G interface eth2 route-test initial-delay 10 \
set load-balance group G interface eth2 route-test interval 10

Let’s walk through each of these with an explanation:

set load-balance group G interface <interface> route-test type ping target <address>
We’re going to use a ping against the specified address instead of the default gateway.
set load-balance group G interface <interface> route-test count [failure|success] 2
We want to wait for two failures or two successes before we decide an interface is unavailable.
set load-balance group G interface <interface> route-test initial-delay 10
Wait 10 seconds before we actually decide there is a failure with this. This helps prevent false-positives on a media reset for devices that might take a moment to initialize even after the media is up.
set load-balance group G interface <interface> route-test interval 10
Run our ping test every 10 seconds. We’re hitting public websites so we need to be considerate here.
Change Connection Weights (Optional)
Background
If you have imbalanced connections like I do, you’ll want to consider adjusting the weight of your load-balancing configuration. Here are my download speeds:

Comcast - 200Mbps down
AT&T - 50Mbps down
For me, the calculation of weight considers two factors:

Download speed of the connection
The reliability of the connection (primarily because of work)
Because the AT&T connection is so much slower than Comcast, I really don’t want to overly favor this (giving equal weight, for example). Instead, I split them based on speed initially (so 75% for Comcast, 25% for AT&T) and then consider that Comcast seems to go down at literally the most inopportune times. This results in 70% to Comcast, 30% to AT&T though I could probably stand to research this further.

Implementation
Here’s how you practically implement the weights:

set load-balance group G interface <interface> weight <weight>
The weight above is specified as an integer representing the percentage (so 70 or 30 respectively for my two connections).

Summary
You have now completed your load-balancing configuration and the additional configuration I recommend. As mentioned in the Things That Break section in part 1, from here things become more complicated in general for how to accomplish various configurations. I fully intend to document my configurations further as I explore these challenges and learn how to properly overcome them. Thanks for your time!

set load-balance group <group> flush-on-active enable
