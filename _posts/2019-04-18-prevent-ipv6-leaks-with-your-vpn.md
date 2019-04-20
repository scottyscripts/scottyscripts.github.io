---
layout: post
title:  "Prevent IPv6 Leaks With Your VPN"
---

## Background

I have been using VPNs for the past few years. I like the added sense of security and anonymity that a VPN provides. However, the other day I realized that my VPN isn't protecting me as much as I had thought. I was configuring openvpn to work with my VPN provider on my new installation of Linux. When I got things up and running I wanted to verify that the VPN was actually doing what it should.

As I expected, `ifconfig` and my routing tables (`route`) showed a tun0 interface was present. I then went to https://whatismyip.com. My IPv4 address matched the VPN server that I had connected to, but the IPv6 address was still my own IP.

A quick search led me to discover that my VPN service (and many others) only support IPv4. This means that if I'm using a site that uses IPv6, any requests I made would not use my VPN tunnel. This is known as an IPv6 leak.

My first thought was that I should look for new VPN service. This is an important feature in my eyes, especially since adoption of IPv6 is on the uprise. In the meantime, I decided that I could just disable IPv6 when using my VPN.

## Dynamically Enabling / Disabling IPv6

*NOTE:* This solution is for my machine running Kali Linux. I would expect this to also work on other Debian based Linux distros though.

My first challenge was being able to dynamically enable / disable IPv6 from my terminal. After some quick research, I found out that IPv6 would be disabled via the kernel. I knew that I would be using `sysctl`, Linux's tool to modify kernel parameters at runtime. I would have to modify these variables using the `sysctl` tool. (I found this after some Googling)

{% highlight shell %}
net.ipv6.conf.all.disable_ipv6
net.ipv6.conf.default.disable_ipv6
net.ipv6.conf.lo.disable_ipv6
{% endhighlight %}

`sysctl --help` provided more insight to the flages I would want to use.

{% highlight shell %}
-w, --write          enable writing a value to variable
{% endhighlight %}

I knew I would have to modify the value of the above 3 variables, but I decided to start with just one.

I checked the value of `net.ipv6.conf.all.disable_ipv6` and discovered it was 0.

{% highlight shell %}
sysctl net.ipv6.conf.all.disable_ipv6
=> net.ipv6.conf.all.disable_ipv6 = 0
{% endhighlight %}

To disable it, the value would need to be 1. This could be done using `sysctl` with the `--write` / `-w` flag:

{% highlight shell %}
sysctl -w net.ipv6.conf.all.disable_ipv6=1
{% endhighlight %}

After checking the value again, I noticed that it was now 1! This could be done for all 3 values that needed to be set to 1. I now knew how to dynamically disable ipv6, but I also wanted a way of permanently disabling it as well.

## Persisting Enabling / Disabling of IPv6

I knew that `sysctl` configuration lived at `/etc/sysctl.conf`. I thought about directly adding config for my net.ipv6* variables in there. After noticing the directory /etc/sysctl.d/ and reading more about `sysctl.d` (`man sysctl.d`) I discovered configuration specified in `sysctl.d` would be applied early on boot. There was also already a file at /etc/sysctl.d/ipv6.conf. I decided any permanent changes to IPv6 configuration would be persisted via this file.

I knew that adding my desired configuration to this file (in /etc/sysctl.d/) would apply changes on boot, but I didn't like the idea of needing to reboot every time I made a change to this file. I wanted to:
  1. disable ipv6 on subsequent system boots
  2. AND wanted those changes to take place immediately.

To handle case 2, I would utilize `sysctl --load` which allows `sysctl` to read values from a file.

I now knew where and how to make changes for ipv6 config AND how to make those changes take effect immediately.

## Putting It All Together: Script Time
{% highlight shell %}
#!/usr/bin/bash

# declare variables to colorize output
red='\033[0;31m'
green='\033[0;32m'
reset_color='\033[0m'

# output information for how to use script
# I named this script ipv6, but feel free to change its name!
print_help() {
  echo -e "Usage:
  ipv6
  ipv6 [-s | --save] on
  ipv6 [-s | --save] off"
}

# print whether ipv6 was enabled / disabled by reading current config
print_ipv6_status(){
  # determine whether ipv6 is enabled / disabled
  ipv6_is_off=$(cat /proc/sys/net/ipv6/conf/all/disable_ipv6)
  if [[ "$ipv6_is_off" -eq 1 ]]; then
    echo -e "ipv6 is ${red}disabled${reset_color}"
  else
    echo -e "ipv6 is ${green}enabled${reset_color}"
  fi
}

# if -s or --save is passed as first argument to script,
#   this config will be saved and systemctl will load this new config
# this regex checks if first argument is '-s' or '--save'
if [[ "$1" =~ ^-s$|^--save$ ]]; then
  if [[ "$2" = "off" ]]; then
    disabled=1
  elif [[ "$2" = "on" ]]; then
    disabled=0
  else
    echo -e "${red}invalid command${reset_color}"
    print_help
    exit 1
  fi
  # store net.ipv6* variable config to sysctl.d to be applied during subsequent boots
  # WARNING: this currently overwrites anything in /etc/sysctl.d/ipv6.conf
  echo -e "net.ipv6.conf.all.disable_ipv6 = ${disabled}
net.ipv6.conf.default.disable_ipv6 = ${disabled}
net.ipv6.conf.lo.disable_ipv6 = ${disabled}" > /etc/sysctl.d/ipv6.conf
  # load newly added config from file, pipe output of command to /dev/null
  sysctl --load /etc/sysctl.d/ipv6.conf > /dev/null
  # custom output about status of ipv6
  print_ipv6_status
else
  if [[ "$1" = "off" ]]; then
    disabled=1
  elif [[ "$1" = "on" ]]; then
    disabled=0
  elif [[ -z "$1" ]]; then
    print_ipv6_status
    exit
  else
    echo -e "${red}invalid command${reset_color}"
    print_help
    exit 1
  fi
  # manually write config at run time, pipe output of command to /dev/null
  sysctl -w net.ipv6.conf.all.disable_ipv6=${disabled} \
         -w net.ipv6.conf.default.disable_ipv6=${disabled} \
         -w net.ipv6.conf.lo.disable_ipv6=${disabled} > /dev/null
  # custom output about status of ipv6
  print_ipv6_status
fi
{% endhighlight %}

## Running It

*Reminder:* before running this script, make sure you make it executable `chmod +x <path_to_script>`. (I also recommend adding it to your `PATH`)

There are a few ways to run this script (assuming I added it to my `PATH` and named the script `ipv6`):

- `ipv6` - return colorized output about if ipv6 is enabled or disabled

- `ipv6 [on|off]` - temporarily enable / disable ipv6 via `sysctl`

- `ipv6 [-s|--save] [on|off]` - enable / disable ipv6, also save and persist config on subsequent boots by adding configuration to /etc/sysctl.d/ipv6.conf

## In Conclusion

I can now use my script to enable / disable IPv6. I now run `ipv6 off` in my terminal before using my VPN to ensure all my traffic is using IPv4 and my VPN tunnel. It is satisfying to now see that my information is once again anonymous when I check a site like https://whatismyip.com while using my VPN.

This process taught me a ton about IPv4 vs IPv6, how VPNs are actually working, and Linux. This also taught me to never feel completely secure, even when using something like a VPN.
