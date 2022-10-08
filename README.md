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

# Launching an EC2 instance

## Signing in to AWS

NOTE: If you're starting from scratch, create an Amazon account by following the instructions [here.](https://aws.amazon.com/free/?all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc&awsf.Free%20Tier%20Types=*all&awsf.Free%20Tier%20Categories=*all)

If you already have an Amazon account, go to the [AWS Management Console](https://aws.amazon.com/console/). Select "Root User" to sign in as a root user.

Once logged in, find the the search bar at the top of the page, and search "EC2." Find "services," and select "EC2." This will take us to our EC2 dashboard.

## Preparing to launch

On the left-hand menu, click "Instances."

This will take us to a dashboard that displays instances we currently have running. We haven't launched our instance yet, so we won't see anything here.

In the top right corner, click "Launch Instances." 

This opens "Launch an Instance." Below the heading you willl see a box called "Name and tags." In the text box, type a descriptive name for your instance, e.g., "My first instance" or "My app server."


In the next section, "Application and OS Images (Amazon Machine Image)," select the Ubuntu AMI. This shows the latest LTS Ubuntu version (as of this writing, "Ubuntu Server 22.04 LTS (HVM), SSD Volume Type - Free tier eligible").

Scroll down to the next section, called "Instance type." By default, it will show 't2.micro' (or t3.micro, if t2.micro is unavailable in your region). Clicking the dropdown, you'll see that there are a several different types of instances we could launch.  In basic terms, we are  launching a cloud computer that will live in a data center owned by Amazon, and the different types of "instances" represent different specifications for our computer. In the future, you might want to upgrade to a different instance (you can do this dynamically with a few clicks later, so don't worry about it right now). The t2.micro is AWS free-tier eligible, so we're going to leave that as our selection for now and proceed.

## Creating a new key pair

The next section is called "Key pair (login)." Now, select "Create new key pair."

This will open a menu. Name your key pair in the text box. You can use the name of your app for now. For the other options, let's  leave 'RSA' selected for key pair type and '.pem' for private key file format.

Click "Create key pair." A file, MyAppName.pem, will be downloaded to your computer. Note the location—you will need to locate this file on your computer in the next section.

In "Network settings," leave the default selections or now and move on to "Configure storage."

At the time of this writing, free tier eligible customers can get up to 30 GB of EBS General Purpose (SSD) or Magnetic storage," so we're going to change the 8GB to 30GB.

Feel free to click the 'Advanced details' to get a sense of what options are available there, but we're not going to change anything right now.

## Lift off!

Click "Launch instance." 

After a brief loading screen, we should see a "Success" indicating that our instance has been successfully launched. 

Click the "View all instances."

We should now see a listing of our instance showing the state "Running" with a green check mark next to it. This indicates that we now have our own computer in the cloud Let's play with it!

# Configure your SSH (Secure Shell) connection


In the previous section, we downloaded a file called, "myapp.pem." By default, it will be ```/Downloads```. To secure the file from potential leaks, we're going to connect to our instance via ssh. Going forward, this guide will assume that your .pem file is located in ```~/Downloads```, but you can replace that with the file path where you've stored it.

If you run in to issues in this next part, see:
* [Connect to your Linux instance using SSH](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html)

## Change security permissions via the Terminal

Open your terminal. If you are a Mac user, there is a good chance your ssh directory has not been created. First, we'll check to see if your .ssh directory exists. In your terminal, type: 

```
cd ~ 
```
This will open your home directory. Once here, type:

```
ls -a
```
This will list all files and directories, including hidden directories. Now, try and locate the ```.ssh``` directory. If the ```.ssh``` directory already exists, we don't need to do anything here. If it does NOT exist, type: ```mkdir .ssh``` and press enter to create a ```.ssh``` directory inside your home directory.

Now that we've ensured our ```.ssh``` directory exists, we're going to move our .pem file there.

Still in the terminal, type:

```
cd ~/Downloads
```

This will navigate you to the ```/Downloads``` directory where the .pem is currently stored.

Now, type (using your own filename): 

```
mv myapp.pem ~/.ssh
```

This will move your .pem file to the ```.ssh``` directory.

Now, navigate to the ```.ssh``` directory:

```
cd ~/.ssh
```


Once we're in the proper directory, we're going to need to set the permissions properly.

First, let's take a look. Type:

```
ls -la
```

Next to myapp.pem, we will see something that looks like ```-rw-r--r---``` (Linux) or ```-rw-r--r--@``` (Mac)

Now, type:
```
chmod 600 myapp.pem
```
This will change the permissions on the .pem file such that the user has read and write permissions and group numbers and others do not. (For more information on ```chmod``` see the [man page.](https://man7.org/linux/man-pages/man1/chmod.1.html))

Linux users, that's it.
Mac users have to type one more command to remove the ```@```:
```
xattr -c myapp.pem
```

Now we can confirm our updated permissions by typing:
```
ls -la
```

Now we should see ```-rw-------``` to the left of our .pem file.

This means we have successfully changed the permissions and placed our .pem file in our ```.ssh``` directory, which means we can now connect to our EC2 instance via the terminal. 

## Using SSH to connect to our EC2 instance

Now the fun beings—we're going to use SSH to connect to our EC2 instance. First, you'll need the public IPv4 address of your new instance. This can be retrieved from the AWS console. 

Navigate to the EC2 dashboard and click "Instances."

Click on your Instance ID. This will look like a blue hyperlink under. "Instance ID."

This will take you to the Instance summary. Find the heading "Public IPv4 address" and copy the IP address listed below the heading to your clipboard. 

Now, go to your terminal. Make sure you are in the directory where the myapp.pem file is stored:
```
cd ~/.ssh
```

Now, type:

```
ssh -i myapp.pem ubuntu@myapp.<PUBLIC IPV4 ADDRESS>
```


(We could also provide the full path and ssh from any directory, like so):

```
ssh -i ~/.ssh/myapp.pem ubuntu@myapp.<PUBLIC IPV4 ADDRESS>
```

Be sure to substitute your app name and domain name. ```ubuntu``` is the default user.

You will see something like
```
The authenticity of host 'ec2-123-45-678-9.compute-1.amazonaws.com (123-45-678-9)' can't be established.
ECDSA key fingerprint is d8FT/adHym7kjsurWb7FRZltnUcJ56LrorpDofHQ7xAY.
Are you sure you want to continue connecting (yes/no)?
```

Type ```yes``` and hit enter.

Our shell with now show ubuntu@ip-OUR-ELASTIC-IP. This means we are now logged in as the default user, ubuntu, on our computer in the cloud. We can do whatever we want here.  Go ahead and type ```echo Hello Cloud!``` and hit ```Enter```.

Your computer just said Hello!

NOTE: If we hit ```Ctrl-D``` during our ssh session, it will disconnect the ssh session and our shell will be back on our local machine.  To reconnect, re-enter the ssh command from above (or just hit the up arrow, if you just disconnected the ssh command will be the last command you entered locally).
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
