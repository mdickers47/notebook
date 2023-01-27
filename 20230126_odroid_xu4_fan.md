                           _________________

                             ODROID XU4 FAN

                            Mikey Dickerson
                           _________________


<2023-01-26 Thu>

This should be a short one.  I have an "ODROID XU4" that I bought a
couple years ago at the peak of my frustration with the design problems
of the Pi 4.  I never used it for anything, but it seems worth figuring
out now (2023), since all models of Raspberry Pi are apparently extinct
forever.  Some things I notice right away:

+ There's a normal, full sized HDMI port.  Yay!
+ The single-core performance is 6% faster than the Pi 4b on the
  arbitrary sysbench test I ran.  Yay!
+ Also there's like, a whole lot of cores?  "4 big and 4 small."  I will
  have to give it a lot of work to find out if this matters.  Yay
  question mark?
+ No wifi.  I don't like to use it anyway but it certainly limits your
  options.  Boo.
+ I don't think anyone is serious about the GPIO pins.  They are
  on a smaller and denser old-PC style header, partly blocked by the
  fan, completely inaccessible in the default plastic case I got, and
  the only information you will find on them is from internet randoms,
  not the manufacturer.  Boo.
+ There's nothing special about the 5V/4A power supply.  That's a "yay"
  compared to the bizarrely botched fake-USB power on Raspberry Pis.
+ Price: $53 is not bad in 2023 dollars, and more importantly, it
  looks like you can actually buy one without a 400% Amazon price
  gouger markup.  Yay!
+ It's probably already discontinued: discounted at the manufacturer and
  production guaranteed "to the Mid-2023."  Boo.
+ The documentation is really hot to get you to run it on an "eMMC"
  storage device.  I have never heard of "eMMC" and from some searches,
  it seems to be a dead evolutionary sibling to SD cards that now only
  exists in ODROID.  But it also can boot from a micro SD card like
  everything else, so ... meh.
+ It has a silly green blinkenlight that I think is supposed to resemble
  a heartbeat?  It speeds up and slows down with temperature or CPU or
  something.  Neither yay nor boo.
+ It boots an Arch Linux ARM image (armv7l) with no particular trouble.
  Yay for not being yet another new architecture to worry about.

Overall it looks like a pretty good Raspberry Pi upgrade if you were
only doing regular computer stuff and not using the GPIO or other
Pi-specific interfaces.  But it had one very annoying feature out of the
box: While totally idle, its little fan spins up, and stops, and spins
up, and stops, on like a 10 second cycle.  It's maybe not "loud" as
these things go, but it's very noticeable, and the constant toggling on
and off is way more annoying than if it would just pick one or the
other.

There's some ODROID wiki pages that tell you how to change the
temperature/fan map:

<https://www.hardkernel.com/blog-2/how-to-control-odroid-xu4-cooling-fan/>

It's the sorta-standard Linux thermal device and PWM fan control, with 5
"thermal zones" reporting in.  Internet randoms say that the 5 sensors
are for the 4 big CPU cores and one GPU; sure, seems like it could be
true.  Let's watch their output for a few minutes:

``` 
[alarm@xu4 ~]$ ( while sleep 3 ; do cat -vn
/sys/devices/virtual/thermal/thermal_zone?/temp ; done ) | tee templog
```

We see that yes, thermal zone "2" is bouncing off the 60C threshold,
triggering the fan, back to 59C, every few seconds.  Presumably
zone 2 is the "big core" drew the short straw and is running the
kernel idle threads.

The normal solution to this is hysteresis, meaning, once the fan
turns on at 60C, it continues to run until some lower bound (say
55C), rather than turn off again at 59.99.  The Linux 4.14 kernel
on here has this feature:

```
[alarm@xu4 ~]$ ls /sys/devices/virtual/thermal/thermal_zone2/trip_point_0_*
/sys/devices/virtual/thermal/thermal_zone2/trip_point_0_hyst
/sys/devices/virtual/thermal/thermal_zone2/trip_point_0_temp
/sys/devices/virtual/thermal/thermal_zone2/trip_point_0_type
```

It's documented the same way as all Linux features, which is not
at all, except by random mailing list posts and the source code.
Anyway it does not work.  Writing numbers to `trip_point_0_hyst`
has no effect.  In random scattered bug reports there are people
asking for this feature, which the developers repeatedly misunderstand,
thinking that we want the fan speed to slowly ramp up or down between
set points.  (I would call that "damping," but nobody needs to call
it anything, because nobody cares about that non-feature.  This fan
has a plastic 3x3cm blade, I don't think its inertia is a big
concern.)

For each "trip point" there is an associated fan set-point, which
determines how much power to send the fan (via PWM) and therefore its
speed.  These can be read out of another device node, which is for some
reason in a different place not near the thermal nodes:

```
[alarm@xu4 ~]$ cat /sys/devices/platform/pwm-fan/hwmon/hwmon0/fan_speed
0 120 180 240
```

"Trip point 1" is where we go from 0 to 120/255ths, which is
noticeably loud.  So I tried raising the threshold for trip point
1 to 65C, on the hope that the idle temperature might stabilize
somewhere below 65, such that the fan is never needed.  No good.
With no fan the temperature on the hot zone just seems to increase
without bound, at least past 70, which is probably too hot to run
all the time.  (Side observation that I overrode the fan speed on
a cheap and *very* loud 4x4k video card and ran it that way for
years, and now it is unstable and regularly freezes its bus and the
whole machine, and these may be related.)

Thinking on this a minute, I realized that hysteresis isn't really a
solution anyway.  It would slow down the cycle time, but the toggling
between 0 and 120 will still be annoying.  This is apparently a CPU that
can't be passively cooled, so there is no choice about the fact that the
fan is going to run.  So a better idea is to never let it go to 0:

```
[alarm@xu4 ~]$ echo "60 120 180 240" | \
   doas tee /sys/devices/platform/pwm-fan/hwmon/hwmon0/fan_speed
```

At 60/255ths the fan is spinning; I can see it but not hear it.  Now at
idle, the hottest zone doesn't go above 57, and a louder speed is never
necessary.  So this looks like the permanent solution.  The echo can go
in /etc/rc.local or some horrible systemd equivalent, so it is
reinstated at boot time.

That's all.  I just wrote this down because it seems like a common
complaint and there are no good Google search results that tell you what
to do.  Maybe in the fulness of time this ascii file will get indexed
and it will solve somebody's problem.

Have fun!
