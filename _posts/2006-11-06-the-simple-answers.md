---
layout: post
title: "Don't overlook the simple answers"
categories: misc
---

Today, I spent a good part of the day troubleshooting an Oracle 10g database whose 
`db_recovery_file_dest` kept filling up. Now, I'm not a DBA by trade, just a technical 
generalist with a penchant for Googling.  I increased the size of the `db_recovery_file_dest` 
and 4 hours later it was full again.  I could not for the life of me figure out why the 
archiving and log rotation RMAN scripts weren't working. I ran them manually, and voila! 
problem fixed again, for a limited time.

That's when it occured to me to look in /var/cron/log. Sure enough, I found the answer 
to all my problems. Well, not ALL of my problems, but enough of the ones I was dealing 
with today that I rated today a success.

The oracle user's password had expired.

That was it. The root cause of two database outages due to the recovery log destination 
filling up, and the database refusing connections, and hours of troubleshooting.

An expired password.

This brings me to a lesson I know well, but often forget.

Never overlook the simple answers!

Sadly, I often forget that, and try to solve a complex problem that's not complex.
