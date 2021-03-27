+++
title = "Lazy Postgres"
description = "Starting postgresql when you want it with the magic of systemd sockets"
date = 2018-04-23
updated = 2020-09-26

[taxonomies]
categories = ["tech"]
tags = ["systemd", "postgres", "linux"]
+++

## Background

Sometimes I want to have local services for some development work outside of a
container but I don't want to run them if I'm not using them.

After using `systemd` for a while I got curious about the `.socket` services. I
knew that they could lazily start services when they are needed but not really
sure what supported them.

So reading the [documentation](https://www.freedesktop.org/software/systemd/man/systemd.socket.html) is a bit like drawing the [rest of the owl](https://www.reddit.com/r/restofthefuckingowl/). It kinda tells you exactly what it is but not much in a way of what it lets you do.

I still don't get exactly what is going on but my mental model for it all is:
 1. the `*.socket` system service will occupy the socket file or network port
     you want to listen to
 2. upon some initial connection `systemd` will start the prescribed service
 3. *???*
 4. profit

The documentation does elude to a initial connection hand over to the service
that has started for that _✨smooth transition✨_ but I have never been able to get to
work.

---

The recipe to success:

## Install

The pre-requisites are having `postgresql` and `systemd` installed.

* create `postgresql.socket` into `/usr/lib/systemd/system` with the content
    below

```
[Socket]
ListenStream=/run/postgresql/.s.PGSQL.5432

[Install]
WantedBy=sockets.target
```

* run `sudo systemctl enable postgresql.socket` to enable the socket service
* *Optional* run `sudo systemctl start postgresql.socket` to start listening
    now & try it out yourself


## Uninstall

* run `sudo systemctl disable postgresql.socket` to disable the socket service
* run `sudo systemctl stop postgresql.socket` to stop the listening socket

## Cavets

 * slow 1st time usage, may even fail 1st connection attempt
 * `ListenStream` needs to point to where postgres would be, e.g. port `5432` or a socket file.
 * This has only been tested on Arch Linux


> This post is largely based on an old [gist](https://gist.github.com/drbarnz/eb7b941740af6e993f1cb1ddeb95beca) I did a few years back, so take this as a [dramatic reenactment](https://www.youtube.com/watch?v=szUJSXBjpJM).
