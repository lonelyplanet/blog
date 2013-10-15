---
layout: post
title: Chef staging environment
categories: []
tags:
- chef
- ci
- lxc
- staging
- test
- toft
status: publish
type: post
published: true
meta:
  draftfeedback_requests: a:1:{s:30:"mark.jennings@lonelyplanet.com";a:3:{s:3:"key";s:13:"5130c95a8c0c5";s:4:"time";s:10:"1362151770";s:7:"user_id";s:8:"43262790";}}
  _publicize_pending: '1'
  _elasticsearch_indexed_on: '2013-03-08 11:07:23'
---
<strong><span style="line-height:16px;">Current situation</span></strong>

We are using <a title="Chef" href="http://www.opscode.com/chef/" target="_blank">Chef</a>in order to deploy our apps. It is a really great tool, associated with Git it allow us to work all together an change the configuration to our live environment by a simple push.

In order to achieve that we have the following process :

Git commit --&gt; Jenkins --&gt; Build agent --&gt; Chef Server
<ol>
	<li><span style="line-height:1.5;">Someone does a commit on GIT</span></li>
	<li><span style="line-height:1.5;">Jenkins monitor the repo, if a commit has been done it will pull the commit on a build-agent which will test the commit (<a href="http://acrmp.github.com/foodcritic/" target="_blank">Foodcritic</a>) </span></li>
	<li><span style="line-height:1.5;">If the test succeed the build-agent runs a script on the Chef Server which pull the change then load it into Chef</span></li>
</ol>
It worked pretty well, the problem was that usually you will pass the Footcritic test but something else will failed during deployment, something you cannot detect without running chef-client on a server. Because we only had a production environment, every time someone broke chef, it impacted all our server. It was not a major problem as we don't run chef-client as adaemon so until someone manually ran chef-client on a server nothing was impacted. It was almost true, because we are using AWS autoscaling which can trigger the build of a new server using chef.

As you can imagine it started to be a big issue when we were working on chef while some new code was pushed to our server (we are usingcontinuousintegration).

<strong><span style="font-size:medium;line-height:16px;">What do we want to fix ?</span></strong>
<ul>
	<li><span style="line-height:16px;">Detect all possible issues before to deploy our chef changes</span></li>
</ul>
The only way we have to test Chef in a real environment is run it on a server, you cannot really use the same server each time because some chef role could conflict with each other, and you would have to uninstalleverythingbefore each new test run. Therefore we need to boot a brand new server. Possible solutions to automatically boot a new server ?
<ol>
	<li><span style="line-height:16px;">AWS EC2 : That's what we use for production, the only issue is that you have to pay 1 hour every time you boot an instance.</span></li>
	<li>Another cloud service : AWS works very well for us, we don;t want to bother with another one</li>
	<li>Our own virtualisation product : It has to be easy to manipulate from our current infrastructure and it's better if it is opensource</li>
</ol>
We chose to use <a href="http://lxc.sourceforge.net/">LXC</a>, it is built in the Linux kernel, the overhead is minimal, and it is really easy to script as a VM is as simple as a folder
<ul>
	<li><span style="line-height:1.5;">Do not impact production when we work on chef changes</span></li>
</ul>
Possible options?
<ol>
	<li><span style="line-height:16px;">Use the chef ENVIRONMENT feature : chef has a build in environment feature, it allow us to use a specific environment, so we can choose on our server if we want to useproductionor staging, for example. The problem is that it is not as isolated as we would like. First of all when you deploy a cookbook in the staging environment it will simply fix the cookbook version in the environment. So you have to bump cookbook version and be sure you don't update the production cookbook version. Second issue, you don't have version on the Roles or Databags, even if you can define attributes per environment (which is really nice) you cannot change itwithoutimpacting the production role. It was the first solution we chose, however its implementation was quite complex, and it was really easy to break production by playing with staging.</span></li>
	<li>Use a second chef server : It is perfect, it iscompletelyisolated from production, and we can change the server we want to use on a client by editing /etc/chef/client.rb. The only problem is that we need to pay for a new server, server which will require monitoring and maintenance</li>
</ol>
We chose the second solution, to be sure at 100% that we will not break production.
<ul>
	<li><span style="line-height:16px;">Automatically boot a LXC server using our staging chef server, and be sure the change succeed before to push it to production</span></li>
</ul>
There are not a lot of ways to test chef, we could have developed our own tools to run a LXC environment running chef-client then a series of test on this LXC server, however wepreferredto use <a href="https://github.com/exceedhl/toft">Toft</a>which is a Ruby library based on Cucumber, Rspec and LXC. Toft handles the launch of the LXC container and the Cucumber test against this containers.

<strong>The solution</strong>

We knew our tools, <strong>Chef, Jenkins, LXC, Toft</strong>.

We then had to chose a process that was as simple as possible, as transparent as possible for the users (us) and as automated as possible. After different test we chose the following process =&gt;

Git commit --&gt; Jenkins --&gt; Build agent --&gt; Chef Staging Server --&gt; Toft --&gt; LXC container --&gt; Jenkins -- Build agent --&gt; Chef Production Server
<ol>
	<li>Someone does a commit on GIT</li>
	<li>A first Jenkins job monitors the repo, if a commit has been done it will pull the commit on a build-agent which will test the commit (<a href="http://acrmp.github.com/foodcritic/" target="_blank">Foodcritic</a>)</li>
	<li>If the test succeeds the build-agent runs a script on the Chef <strong>STAGING</strong> Server</li>
	<li>Toft will be run on our LXC server, it will boot a LXC container with the roles we want, then it will test the results (files expected, process running etc...)</li>
	<li>If the Toft run and test succeeds, run a second Jenkins job</li>
	<li>A second Jenkins job will retrieve the last successful build (validated by toft), then it will tell to a build-agent to pull this specific successful build on the chef <strong>PRODUCTION</strong> server and to load it into Chef</li>
</ol>
That's it, a fully automated test solution for Chef.
