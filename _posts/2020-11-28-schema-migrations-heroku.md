---
title: Running schema migrations on Heroku
author: g14a
date: 2020-11-28 11:33:00 +0800
categories: [tutorials]
tags: [databases, golang, heroku]
toc: true
---

This is a quick article on how schema migrations are run beforing deploying your Go application(or any another app) on Heroku. I've myself faced some problems due to lack of proper documentation with the Go buildpack. A bunch of SO threads have helped me pave the correct path. So let's start.

### What you need

* An application server

That's it. Preferrably add ```// +heroku goVersion go1.15``` after declaring your module in ```go.mod``` to get the version you want. I write Go code usually but there would be buildpacks for other languages as well.(I'm hoping)

I prefer [golang-migrate](https://github.com/golang-migrate/migrate) for handling database migrations since it supports a variety of databases, while being usable as a Linux binary or Go library.

Add your migrations to the ```db/migrations``` path as specified by the library. Once that is done, create a shell script ```migration.sh``` and add the following code:

```
wget -O /tmp/migrate.linux-amd64.tar.gz https://github.com/golang-migrate/migrate/releases/download/v4.14.1/migrate.linux-amd64.tar.gz &&
tar -xvf /tmp/migrate.linux-amd64.tar.gz -C /tmp &&
/tmp/migrate.linux-amd64 -database "${DATABASE_URL}" -path db/migrations up
exit_status=$?
RED='\033[0;31m'

if [ $exit_status -ne 0 ]; then
    echo "${RED}Migration did not complete smoothly. Rolling back..."
    /tmp/migrate.linux-amd64 -database "${DATABASE_URL}" -path db/migrations down --all
    exit 1;
fi
```

Although [golang-migrate](https://github.com/golang-migrate/migrate) supports for all kinds of platforms, we use the Linux binary as Heroku dynos are Linux containers. The above shell script gets the binary and stores it in the ```/tmp``` path to run it from. And you need your database URL in an environment variable which might already be set in your Heroku Config. 

The script performs a downward migration only if the upward migration exits with an `exit_status` other than `0`.

Once this is done, go ahead and create a ```Procfile``` in your root directory(assuming you don't have one already) and add
the release phase command.

```
release: chmod u+x migration.sh && ./migration.sh
web: bin/<your-app>
```

And voila! You're done. Go ahead and push your changes to your Heroku remote and see your migrations work like a charm!

Release phase are used for tasks like:

> * Sending CSS, JS, and other assets from your app’s slug to a CDN or S3 bucket
> * Priming or invalidating cache stores
> * Running database schema migrations

Checkout [Release Phase](https://devcenter.heroku.com/articles/release-phase) on Heroku for more information.

Thank you for reading and hope you're relieved from stresses due to deployments!