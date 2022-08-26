Forthcoming tutorial/guide to spinning up an EC2 instance on AWS free tier and setting a Cron job to ping your sleepy dynos.

```0,30 11-23 * * 1-5 /usr/bin/curl -Ls MyApp.herokuapp.com >>/tmp/heroku_keepalive.log 2>&1```
