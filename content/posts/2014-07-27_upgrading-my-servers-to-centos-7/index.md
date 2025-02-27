---
title: "Upgrading my servers to CentOS 7"
author: "Russ Mckendrick"
date: 2014-07-27T11:00:00.000Z
lastmod: 2021-07-31T12:31:42+01:00

tags:
 - Tech
 - Centos
 - Docker
 - El7
 - Puppet




aliases:
- "/upgrading-my-servers-to-centos-7-b38ea4c0571f"

---

==**PLEASE NOTE:** THESE INSTRUCTIONS ARE NOW OUTDATED, SEE [THIS POST FOR AN UPDATE](https://media-glass.es/2015/07/12/update-to-puppet-install-script/)==

After I dabbled with [CentOS 7](https://media-glass.es/2014/07/13/centos-7/) a few weeks ago I decided to take the plunge and upgrade the few servers I run at [DigitalOcean](https://www.digitalocean.com/?refcode=52ec4dc3647e) to CentOS 7. I run two machines, one is a Puppet and Salt Master, the second runs Docker.

#### Puppet Master Server

A while back I added a [few bash scripts to deploy both a Puppet Master & Agent](https://github.com/russmckendrick/puppet-install) to GitHub, as Puppet provide a repo for CentOS 7 I decided to just update the installation script to use this repo and then give ago. I use [stephenrjohnson/puppet](https://forge.puppetlabs.com/stephenrjohnson/puppet) to deploy the Puppet Master. the Puppet part worked, however it failed to install Passenger and its Apache module because there is no mod_passenger available for CentOS 7 yet. To get around this I grabbed the require packages from [The Foremans nightly builds](http://yum.theforeman.org/nightly/el7/x86_64/) repo. This turned the initial yum install line into the following;

[code gutter=”false”]
yum install -y vim-enhanced git [http://yum.puppetlabs.com/puppetlabs-release-el-7.noarch.rpm](http://yum.puppetlabs.com/puppetlabs-release-el-7.noarch.rpm)[https://anorien.csc.warwick.ac.uk/mirrors/epel/beta/7/x86_64/epel-release-7-0.2.noarch.rpm](https://anorien.csc.warwick.ac.uk/mirrors/epel/beta/7/x86_64/epel-release-7-0.2.noarch.rpm)[http://yum.theforeman.org/nightly/el7/x86_64/mod_passenger-4.0.18-9.5.el7.x86_64.rpm](http://yum.theforeman.org/nightly/el7/x86_64/mod_passenger-4.0.18-9.5.el7.x86_64.rpm)[http://yum.theforeman.org/nightly/el7/x86_64/rubygem-passenger-native-4.0.18-9.5.el7.x86_64.rpm](http://yum.theforeman.org/nightly/el7/x86_64/rubygem-passenger-native-4.0.18-9.5.el7.x86_64.rpm)[http://yum.theforeman.org/nightly/el7/x86_64/rubygem-passenger-native-libs-4.0.18-9.5.el7.x86_64.rpm](http://yum.theforeman.org/nightly/el7/x86_64/rubygem-passenger-native-libs-4.0.18-9.5.el7.x86_64.rpm)[http://yum.theforeman.org/nightly/el7/x86_64/rubygem-passenger-4.0.18-9.5.el7.x86_64.rpm](http://yum.theforeman.org/nightly/el7/x86_64/rubygem-passenger-4.0.18-9.5.el7.x86_64.rpm)[http://yum.theforeman.org/nightly/el7/x86_64/rubygem-rack-1.4.5-3.el7.noarch.rpm](http://yum.theforeman.org/nightly/el7/x86_64/rubygem-rack-1.4.5-3.el7.noarch.rpm)[http://yum.theforeman.org/nightly/el7/x86_64/rubygem-rack-doc-1.4.5-3.el7.noarch.rpm](http://yum.theforeman.org/nightly/el7/x86_64/rubygem-rack-doc-1.4.5-3.el7.noarch.rpm)
[/code]

#### Apache Configuration

Now everything was being installed correctly, however Apache refused to start due to the following error;

[code gutter=”false”]
Invalid command ‘RackAutoDetect’, perhaps misspelled or defined by a module not included in the server configuration
[/code]

This was due to the presence of RackAutoDetect Off and RailsAutoDetect Off in the puppet_passenger.conf.erb file. Once these two lines were removed everything worked.

The final script can be [found here](https://github.com/russmckendrick/puppet-install/blob/master/install-el7)

#### Puppet Configuration

I have had my base [Puppet Configration on GitHub](https://github.com/russmckendrick/puppet) for a while. So my next move was to grab a copy of the configuration and try it. For the most part this worked, there were a few errors though.

- man was not available so was removed
- jwhois was replaced with whois
- The firewall module was not able to use iptables-save even though iptables was installed and configured, installing iptables-services resolved this

The rest of the packages and configuration worked as expected.

#### Docker Server

This time I tried the Puppet Agent install script, it worked first time;

[code gutter=”false”]
curl -fsS [https://raw2.github.com/russmckendrick/puppet-install/master/agent-el7](https://raw2.github.com/russmckendrick/puppet-install/master/agent-el7) | bash -s puppet.master.com
[/code]

Once I had Puppet installed I ran puppet agent — test and was greeted with lots of NGINX errors, but thats all. Everything else worked as expected.

#### NGINX

The only issue here was the Puppet module I am using to install and configure NGINX was using [http://nginx.org/packages/rhel/6/](http://nginx.org/packages/rhel/6/) I updated it to use [http://nginx.org/packages/centos/7/](http://nginx.org/packages/centos/7/) and hey presto, server is back to working as before.

#### Conclusion

In all, other than having to tweak some packages to get services running it wasn’t as bad as I thought it would be. Considering the release is still less than a month old I expected to have to do a lot more work than I did to get everything up and running how I had it previously.

So would I run it production?

Yes & No. If I was running a service which didn’t anything other standard repository packages then I would have no problem running it in production, for some of the none standard stuff like the Puppet Master server, I would give it a few more weeks until the required packages start to filter down to more production ready repositories.
