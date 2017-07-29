---
layout: post
title: Creating MySQL databases in Scrutinizer CI
---

I absolutely love [Scrutinizer CI](http://scrutinizer-ci.com) for analysing my code, running tests and even for running my ansible deployment scripts. One thing that really sucks about Scrutinizer CI is that it has really lacking documentation. Config options are rarely discussed, and can be find by-change in examples. There's no explanation of the config options, you just have to learn through trial and error. From one of the documentation examples, we can work out how they recommend creating a MySQL database before running our tests:

```yaml
    # Run after dependencies
    project_setup:
        before:
            - mysql -e "CREATE DATABASE abc"
```

This seems to work, for the most part, except very intermittently you will get the following error:

    ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock' (2)

Annoyingly enough, if you run the inspection with SSH debugging turned on (which is incredibly useful, by the way) then it always seems to connect fine. After a bit of head scratching, I came to the conclusion that since it's booting up a fresh container for each inspection, it's probably a race condition between MySQL loading up and this command being ran.

Therefore, in an attempt to make sure MySQL is booted up, I've started using the following command:

```yaml
  # Run after dependencies
    project_setup:
        before:
            - sudo service start mysql || true
            - mysql -e "CREATE DATABASE abc"
```

There's probably a better way of doing it, but for all intents and purposes, this seems to fix the problem (although the annoying thing about intermittent issues is you can never truly be sure they're fixed!).

There's also another issue, which appears to be that Scrutinizer CI boots the application inside another container and runs composer install. If you use Symfony, you may find that it tries to run the post-install hooks and can't connect to the database. I'm still working on a fix for this one unfortunately!
