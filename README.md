thrash-protect
=============

Simple-Stupid user-space program protecting a linux host from thrashing.

The script attempts to detect thrashing situations and temporary stop
rogue processes, hopefully before things get too much out of control,
hopefully giving a sysadm enough time to investigate and handle the
situation if there is a sysadm around, and if not - hopefully allowing
boxes to become just slightly degraded instead of completely thrashed,
all until the offending processes ends or the oom killer kicks in.

As of 2014-09, the development seems to have stagnated - for the very
simple reason that it seems to work well enough for me.

Problem
-------

It's common to add relatively much swap space to linux installations.
Swapping things out is good as long as the the swapped-out data is
really inactive.  Unfortunately, if actively used memory ends up being
swapped out (actively running applications using more memory than
what's available), linux has a tendency to become completely
unresponsive - to the point that it's often needed to reboot the box
through hardware button or remote management.

It can be frustrating enough when it happens on a laptop or a work
station; on a production server it's just unacceptable.

If asking around on how to solve problems with thrashing, the typical
answer would be one out of four:

* Install enough memory!  In the real world, that's not always
  trivial; there may be physical, logistical and economical
  constraints delaying or stopping a memory upgrade.  It may also be
  non-trivial to determinate how much memory one would need to install
  to have "enough" of it.  

* Disable swap.  Together with the advice "install enough memory" this
  is really a safe way (and the only safe way) to prevent thrashing.
  However, in many situations swap can be a very good thing - i.e. if
  having processes with memory leakages, aggressive usage of tmpfs,
  some applications simply expects swap (keeping large datasets in
  memory), etc.  Enabling swap can be a lifesaver when a much-needed
  memory upgrade is delayed.

* Tune the swap amount to prevent thrashing.  This doesn't actually
  work - even a modest amount of swap can be sufficient to cause
  severe thrash situations.

* Restrict your processes with ulimit.  In general it makes sense, but
  doesn't really help against the thrashing problem; if one wants to
  use swap one will risk thrashing.

Simple solution
---------------

This script will be checking the pswpin and pswpout variables in
/proc/vmstat on configurable intervals.  If both swap in and swap out
is detected within the interval, the program will STOP the most nasty
process.  The same will happen if there has been significant amounts
of swap in or swap out.  When the host has stopped swapping the host
will resume one of the stopped processes.  If the host starts swapping
again, the last resumed PID will be refrozen.

Finding the most "nasty" process seems to be a bit non-trivial, as
there is no per-process counters on swapin/swapout.  Perhaps it's
possible to check the delta of the total swap pages used and sum it
together with the number of page faults.  Currently three algorithms
have been implemented and the script uses them in this order:

* Last unfrozen pid, if it's still running.  Of course this can't work
  as a stand-alone solution, but it's a very cheap operation and just
  the right thing to do if the host started swapping heavily just
  after unfreezing some pid - hence it's always the first algorithm to
  run after unfreezing some pid.

* oom_score; intended to catch processes gobbling up memory without
  making significant amounts of page faults.  It has some drawbacks -
  it doesn't target the program behaviour "right now", and it will
  give priority to parent pids - when suspending a process, it may not
  help to simply suspend the parent process.

* Number of page faults.  This is non-ideal because a rogue process
  gobbling up memory and swap through write-only operations won't
  cause page faults.  Also, a "page fault" is not the same as swapin -
  it may also happen when a program wants to access data that the
  kernel has postponed loading from disk (typically program code -
  hence one typically gets lots of page fault when starting some
  relatively big application).  The worst problem with this approach
  is that it requires state about every process to be stored in
  memory, this memory may be swapped out, and if the box is really
  thrashed it may take forever to get through this algorithm.

The script creates a file on /tmp when there are frozen processes,
nrpe can eventually be set up to monitor the existance of such a file
as well as the existance of suspended processes.

Important processes (say, sshd) can be whitelisted, and processes
known to be nasty or unimportant can be blacklisted.  Note that the
"black/whitelisting" is done by weighting - randomly stopping
blacklisted processes may not be sufficient to stop thrashing, and a
whitelisted process may still be particularly nasty and stopped.

With this approach, hopefully the most-thrashing processes will be
slowed down sufficiently that it will always be possible to ssh into a
thrashing box and see what's going on.

Implementation
--------------

A prototype has been made in python - my initial thought was to
reimplement in C for smallest possible footstep, memory consumption
and fastest possible action - though I'm not sure if it's worth the
effort.

Tweaks implemented
------------------

I guess there are a lot of ways to tweak and tune this script, though
I'm worried that the simplicity will go down the drain if doing too
much tweaking.  After experimenting with the "thrash-bot.py"-script, I
realized some tweaking was necessary though.

I very soon realized that both a queue approach and a stack approach
on the frozen pid list has its problems (the stack may permanently
freeze relatively innocent processes, the queue is inefficient and
causes quite much paging) so I made some logic "get from the head of
the list sometimes, pop from the tail most of the times".

Up until and including 0.6 only delta maj_page_faults when deciding
which process to STOP.  More algorithms was implemented in 0.7.

If the script resumes some process and the host immediately starts
thrashing again, it's probably smart to refreeze the same process,
hence a third method for discovering which pid to freeze was created.
This is also a very cheap operation, so it's always the first method
run after unfreezing a process.

Finally, I found that I couldn't allow to do a full
sleep(sleep_interval) between each frozen process if the box was
thrashing.  I've also attempted to detect if there are delays in the
processing, and let the script be more aggressive.  Unfortunately this
change introduced quite some added complexity.

It's important to do some research to learn if the program would
benefit significantly from being rewritten into C before doing too
much tweaking on the python script.  If not, the python script should
be tweaked, refactored and optimized a bit (i.e. using re instead of
split, skip floats and use ints instead, look through and simplify the
algorithm where possible, garbage collection of old processes from the
pid/pagefault dict, read more options, improved log handling, etc).

Experiences
-----------

As of 2014-09, this script has been run on several production systems
and some workstations/laptops for almost a year without problems, it
has definitively saved us from several power-cyclings.  Best of all,
in most of the cases I haven't had to do anything - except, under some
circumstances, I had to add more swap space.  Instead of a whole
server or VM being thrashed beyond rescue some "badass" processes have
been peacefully and temporary suspended without anyone noticing, and
eventually the situation has resolved itself.  Hence I like to have it
running on any system having any kind of swap capacity mounted up.
Anyway, the script hasn't been through any thorough peer-review, and
it hasn't been deployed to many systems yet - don't blame me if you
start up this script and anything goes kaboom.

I would strongly recommend to give this script a shot as a temporary
stop-gap-solution if you have a server that have had thrashing problem
earlier, and where the problem cannot be solved (in a timely manner)
by adding more memory or shrinking the swap partition.

Drawbacks and problems
----------------------

* Some parent processes may behave unexpectedly when the children gets
  suspended - particularly, the suspension of interactive programs
  under the login shell (say, "less") may be annoying.  Observed
  situation: starting a minecraft server from an interactive bash
  session, if it gets suspended it cannot be resumed with "kill -CONT"
  as it depends on being run in the foreground (workaround: spawn
  minecraft i.e. from screen rather than bash).  I've also received
  one report that this script may have interphered with the condor job
  control system.

* I've observed situations where parent processes automatically have
  gone into suspend-mode as the children got suspended and been stuck
  there even as the child process got resumed.  I've done a quick
  work-around on this by always running SIGCONT on the session process
  id and group process id.  This may be harmful if you're actively
  using SIGSTOP on processes having children.

* This was supposed to be a rapid prototype, so it doesn't recognize
  any options.  Configuration settings can be given through OS
  environment, but there exists no documentation.  I've always been
  running it without any special configuration.

* Usage of mlockall should be made optional.  On a system with small
  amounts of RAM (i.e. half a gig) thrash_protect itself can consume
  significant amounts of memory.

Other thoughts
--------------

This should eventually be a kernel-feature - ultra slow context
switching between swapping processes would probably "solve" a majority
of thrashing-issues.  In a majority of thrashing scenarioes the
problem is too fast context switching between processes; this causes a
very insignificant amount of CPU cycles to be actually be spent on the
process, while the very most time is spent swapping between processes.

Roadmap
-------

Focus up until 1.0 is deployment, testing, production-hardening,
testing, testing, bugfixing and eventually some tweaking but only if
it's _really_ needed.  Some things that may be considered before 1.0:

* More "lab testing", and research on possible situations were thrash-bot wins over thrash-protect.  Verify that the mlockall() actually works.

* Tune for lower memory consumption

* look into init scripts, startup script and systemd script to ensure program is run with "nice -n -20"

* awareness of troublesome parents, i.e. bash.  When stopping a process running under a bash shell, the shell should be stopped first, and resumed after the process has been resumed.

* Look into init scripts, startup script and systemd script to allow for site-specific configuration

* proper logging

* Fix puppet manifest to accept config params

* look into the systemd service config, can the cgroup swappiness configuration be tweaked?  

* Reproduce parent suspension problems and fix properly

* Graceful handling of SIGTERM (any suspended processes should be reanimated)

* Recovery on restart (read status file and resume any suspended processes)

* Improved logging and error handling

* More work is needed on getting "make rpm" and "make debian" to work

* Package should include munin plugins

Things that eventually may go into 2.0:

* Replace floats with ints

* Rewrite to C for better control of the memory footprint

Donations
---------

Donations are not expected - but as long as this is a one-man hobby project it's non-problematic to receive donations.  Use bitcoin address 12jZRp4MVNySnQ9ypEP746G84jYmJATTVs eventually.
