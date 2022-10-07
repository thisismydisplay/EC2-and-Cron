# How to start an EC2 instance on AWS free tier & setting a crontab to ping your sleeping dynos.

---

# Sleepy dynos

When I was in bootcamp, I used free hosting services like Heroku to deploy my full stack applications.  Unfortunately, an annoying limitation of using Heroku is that when inactive, your apps will [go to sleep](https://blog.heroku.com/app_sleeping_on_heroku) after a short period of inactivity.  This can cause a site to have a 15-20 second load time when someone first accesses your site - not a good look when you're trying to get recruiters and potential employers to hire you.

This is because of how Heroku manages it's free dynos.  Dynos are isolated, virtualized Linux containers that are designed to execute code based on a user-specified command.  You can read more about them [here](https://www.heroku.com/dynos#:~:text=The%20containers%20used%20at%20Heroku,based%20on%20its%20resource%20demands.).

Two popular options to keep your apps 'awake' are [Kaffeine](https://kaffeine.herokuapp.com/) and [UptimeRobot](https://uptimerobot.com/).

However, with Heroku, free applications must sleep for 6 hours every day, and Heroku limits the total amount of awake hours across all your free hosting per month.  Now, if you just have one site, no problem - use Kaffeine, it takes 5 seconds, works great, and you can set a 'bedtime' for the required 6 hours of sleep.  But if you have more than one site you want awake, you are going to run out of hours before month's end.  UptimeRobot only provides scheduling with a paid plan.

---
# Some important notes before beginning

## Add your credit card to Heroku:

It's free to add your card info (they won't charge you for the free plan), and it will double your available hours of awake time.

## EC2 is optional

If you don't want to set up an EC2 instance, you can simply set up a crontab on your local computer! Just know that your local machine must be on, awake, and connected to the internet, or the cron jobs will not run.

If you want to go this route, Linux and Mac users can skip right to the [Setting up the crontab](#setting-up-the-crontab) section of this tutorial.

If you're on windows, presumably you could set this up from your WSL environment, but you'll have to make sure the cron daemon runs in the background when you boot windows if you don't want to always load WSL in the background on boot - I haven't tested this but a quick google search for 'crontab in wsl' looks [promising](https://www.howtogeek.com/746532/how-to-launch-cron-automatically-in-wsl-on-windows-10-and-11/).  Or you could just use windows task scheduler to solve the sleepy dyno problem however you want.

## Be careful with AWS:

AWS provides services that are meant to have constant runtime and be quickly scalable.  This is great if you need it, but if you don't know what you're doing, it is possible that you could rack up a hefty bill.

This tutorial assumes this is your first foray into EC2, in which case nothing in this tutorial will cost you anything for the first year.  However, you'll want to be mindful to terminate your instance before the 12th month ends - Amazon will warn you before this happens with plenty of time to shut down your instance.  If you don't want to deal with this, you can set up something local per the [EC2 is optional](#ec2-is-optional) section, or investigate other free services and set up something similar.  But, by then this may not be relevant because...

## Heroku free tier is going away:

"Starting November 28, 2022, free Heroku Dynos, free Heroku Postgres, and free Heroku Data for Redis® will no longer be available. You can learn more about these and other important changes from our GM, Bob Wise, on the [Heroku blog](https://blog.heroku.com/next-chapter).

Existing free dynos and Heroku data add-ons will be impacted, so action by you is required. To prevent any disruption to your apps or data using free plans, you will need to upgrade from a free plan to a paid plan before November 28, 2022."

It's unclear as of this writing what this means for students, hobbyists, and the relevance of this tutorial.

---

# OK, let's get started!

The EC2 portion of this tutorial is forthcoming by @nszad.  Until then, feel free to use Amazon's offical tutorial and docs to get it set up in this order:

1. [AWS Docs - Set up to use Amazon EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/get-set-up-for-amazon-ec2.html)
2. [AWS Docs - Tutorial: Get started with Amazon EC2 Linux instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html) *Make sure to use Ubuntu Focal Fossa LTS for your Linux distro for this tutorial
3. [AWS Docs - Connect to your Linux instance using SSH](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html) (You can stop just before "Transfer files to Linux instances using an SCP client")

---

# Cron and crontab


## What is cron?

Cron is a utility in Linux that allows for scheduling tasks to be run in the background at regular intervals.


## What is a cron job?

A cron job is simply execution instructions that specify a time/interval and command to execute.


## What is crontab?

Crontab is a file that contains the cron jobs to be run.

## Setting up the crontab:

First, I want to check what timezone my instance is set to (users setting up a crontab locally can skip to the ```crontab -l``` command below).  I can run
```
timedatectl
```
in the command line.  It will likely show UTC as the time zone and local time.

We can change our time zone to reflect our local time.  First, type
```
timedatectl list-timezones
```

We will need to use the long form of one of these strings as our parameter for the next command. For example, I am in Central Time, so I will take note of ```America/Chicago```. Now, I can set the local time on my instance with:
```
sudo timedatectl set-timezone America/Chicago
```

To sanity check, run
```
timedatectl
```
again and see that the local time has changed.

Great.  Now we can think about our cron job in terms of our local time.

First, we can type
```
crontab -l
```
in our terminal and hit enter to see current tasks.  Since we just spun up our instance, there shouldn't be any yet.

Now, type
```
crontab -e
```
and hit enter to edit the crontab.  Since this is the first time editing the crontab, we need to select our editor.  This tutorial will use nano, as it will be easiest for new users.  Select
```
1. /bin/nano
```
by pressing the 1 key.

This will open the crontab file in the nano editor:
```
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').
#
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
#
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
#
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
#
# For more information see the manual pages of crontab(5) and cron(8)
#
# m h  dom mon dow   command
```

Use the down-arrow key to scroll to the bottom of the file below the comments (lines starting with ```#``` are comments).

We are going to add our first cron job on the first empty line.  Here is the example cron job we will enter into our crontab.

```
0,30 6-19 * * 1-5 /usr/bin/curl -Ls MyApp.herokuapp.com >>/tmp/heroku_keepalive.log 2>&1
```

Make sure to edit the url to be the name of your site.

If we have more than one app we want cron to keep awake for us, we simply add another line below the first job with the second job.

```
0,30 6-19 * * 1-5 /usr/bin/curl -Ls My2ndApp.herokuapp.com >>/tmp/heroku_keepalive.log 2>&1
```

Now, hit
```
Ctrl + X
```
to exit and when prompted hit
```
Y
```
to save changes.  Hit enter to overwrite the crontab file we have just edited with the saved changes.

### Great!  Your crontab will now ping your sites for you.  But what did we just do?

---

# Let's go through the syntax.


## Scheduling Parameters:

The first parameter
```
0,30
```
is the minute value, and in this case I'm setting it to run on the hour and at 30 minutes after the hour.  The comma indicates that I am providing more than one value.

The second parameter
```
6-19
```
is the hour value.  Because I have several apps, and Heroku allows me 1000 hours of uptime per month using their free tier services, I have to be judicious in when I schedule my apps to be awake.  The dash between numbers indicates a range.

(When in doubt, [crontab.guru](https://crontab.guru/) is a nice resource for confirming your cron schedule expressions.)


## Our Schedule:

Let's say I'm in U.S. Central (CST) time and I want my app awake during normal work hours.  I also want to cover work hours on the west coast, mountain time, and the east coast.

By using 6-19, our apps will be awake by 7am for someone on the east coast (CST+1=7), and 19 means our apps will be awake until at least 5:30pm on the west coast (CST-2=17) - so these values cover 7am-5:30pm for the continental US.

The third and fourth parameter are numerical values representing day-of-the-month and month-of-the-year - we use the wildcard ```*``` here because we want this job always running.

The fifth parameter is a numerical value representing day-of-the-week (0 is Sunday).  1-5 tells cron to run this job Monday through Friday.


## Command to execute:

We're just going to use a simple ```curl``` command to ping our site.  We tell the cron job the full command path
```
/usr/bin/curl -Ls
```
and provide the -L and -s flags.  For a full list of curl flags, see the [curl man page](https://curl.se/docs/manpage.html).


## Logs and 2>&1:

```
>>/tmp/heroku_keepalive.log
```
tells the cron job to output the execution log to a file - in this case ```heroku_keepalive.log``` in our ```tmp``` directory (by default crontab will attempt to send an email).

```2>&1``` "redirects standard errors to standard out file descriptor, which in this case is already pointing to a file."

Explaining this in depth is beyond the scope of this guide, but [this article](https://www.adminschoice.com/what-does-21-mean-in-shell) explains it pretty well.

Basically we need this to redirect any error messages into our log file, because of how the Linux shell handles standard output and standard errors.
