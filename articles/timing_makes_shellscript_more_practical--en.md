---
title: "Improve Your Shell Script's Timing Precision, and It Suddenly Gets Fun"
emoji: "🚴"
type: "idea"
topics: ["unix", "linux", "shellscript", "iot", "posix"]
published: true
---

（日本語版は[こちら](timing_makes_shellscript_more_practical)）

# Introduction

Shell scripts are handy for organizing files or kicking off commands, but a lot of people never use them for much more than that. Once you can control time with high (sub-second) precision, though, things suddenly get interesting — precisely timing when data flows, or precisely timing when commands fire.

I built a collection of UNIX commands for exactly this, so let me introduce it and show off some fun examples along the way.

# Preparation (Installation)

The command collection is called [tokideli](https://github.com/ShellShoccar-jpn/tokideli), and here's how to install it.

## For Homebrew Users

I've set up a Homebrew tap (a mechanism for building unofficial Homebrew packages), so you just need to run these two commands.

```
$ brew tap ShellShoccar-jpn/tap
$ brew install tokideli
```

## For Everyone Else

Clone the source from GitHub, build it, and add it to your PATH. I've written a script that handles the build and PATH setup after cloning, so just run the following.

```
# For installing with administrator privileges
$ git clone https://github.com/ShellShoccar-jpn/tokideli.git
$ sudo tokideli/INSTALLIN.sh /usr/local/tokideli

# For installing into your own home directory
$ git clone https://github.com/ShellShoccar-jpn/tokideli.git
$ tokideli/INSTALLIN.sh $HOME/tokideli
```

To uninstall, just delete the directory you gave to `INSTALLIN.sh`, and remove the PATH entry it added.

# Let's Just Play Around With It

Rather than talking theory, you'll get it faster by just using it and playing around, so let's dive straight into examples.

## Reliving the Days of Dial-Up

Until roughly the 1990s, computers connected over phone lines for mostly text-based communication, at speeds like 300bps, 1200bps, or 9600bps, where characters trickled across the screen one by one. Anyone who grew up on gigabit internet probably can't picture it, so let's recreate that speed with a one-liner using the [`valve` command](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/valve.man.en.md) (valve, as in a flow-control valve). (If you remember it firsthand, feel free to relive the nostalgia too.)

```sh:
$ fortune | valve 300bps
$ fortune | valve 1200bps
$ fortune | valve 9600bps
```

![An example of showing a fortune message nostalgically](https://raw.githubusercontent.com/ShellShoccar-jpn/tokideli-assets/main/article_figs/images/fortune_valve_demo.gif)

If you don't have the [`fortune` command](https://en.wikipedia.org/wiki/Fortune_(Unix)), install it with `apt` or `brew`, or just `cat` any text file you have handy into [`valve`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/valve.man.en.md) instead.

## Making SL Crawl

UNIX has a command called [`sl`](https://www.tkl.iis.u-tokyo.ac.jp/~toyoda/). If you typo `ls` as "sl", a steam locomotive comes through and you can't do anything until it's passed — it's basically a prank command. (Again, install it with `apt` or `brew` if you don't have it.)
![sl command (running normally)](https://raw.githubusercontent.com/ShellShoccar-jpn/tokideli-assets/main/article_figs/images/sl_demo.gif)

Running the next command limits the pipe's transfer rate to 9600bps, so SL crawls across the screen instead and blocks the terminal even longer.
```sh:
$ sl | valve 9600bps
```
![sl command (crawling)](https://raw.githubusercontent.com/ShellShoccar-jpn/tokideli-assets/main/article_figs/images/sl_valve_demo.gif)

You might think "that just makes it even more annoying!" — fair enough, but you can interrupt it with [Ctrl]+[C] once you've had enough. (Just run `reset` afterward to fix up the garbled screen.)

## Downloading Without Getting Rate-Limit Banned

Here's a slightly more practical example.

Sometimes you need to download a large batch of sequentially-numbered files (you know the feeling, right?). But download them too fast and you might get banned for hitting a rate limit, so you want to respect the rate limit while downloading. That's another job for the [`valve` command](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/valve.man.en.md).

Say there's a website that will rate-limit-ban you if you download files more often than once every 3 seconds. Write a one-liner like this, and the loop keeps a steady 3-second interval even if the command inside `while`...`done` takes a varying amount of time to run — so you stay within the rate limit. (Careful though: if the command inside takes longer than 3 seconds, the next round gets shorted by that much 😉.)
```sh:
$ seq -f '%02.0f' 1 50 | valve -l 3s | while read n; do wget "https://example.com/file_${n}.jpg"; done
```
Here, the [`valve` command](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/valve.man.en.md)'s `-l` option means "insert a wait after every line," so in this example one line comes out every 3 seconds.

Running it for real, you can see downloads firing off at a nice, steady 3-second interval, just like in the image below.
![Downloading a series of numbered files with valve + wget](https://raw.githubusercontent.com/ShellShoccar-jpn/tokideli-assets/main/article_figs/images/wget_valve_demo.gif)

This example used the [`wget` command](https://www.gnu.org/software/wget/) for file downloads, but the same idea applies to any one-liner or shell script: write it this way and you can run any command repeatedly at a precise interval.

## A New Use for the `yes` Command (a Pulse Generator)

Ask someone to explain what the [`yes` command](https://en.wikipedia.org/wiki/Yes_(Unix)) is for, and most people will say "it's a lazy command for answering `y` to repeated `(y/n)?` prompts, and nothing else." But combine it with the [`valve` command](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/valve.man.en.md), and it's suddenly reborn **as a pulse generator for embedded hardware**.

Connect a [Raspberry Pi](https://www.raspberrypi.com/), an oscilloscope, and a reference pulse generator, as shown below.
![Connection diagram: Raspberry Pi, oscilloscope, and reference pulse generator](https://raw.githubusercontent.com/ShellShoccar-jpn/tokideli-assets/main/valve_as_wave_generator_on_raspi_fig01.png)
Then run the following one-liner on the [Raspberry Pi](https://www.raspberrypi.com/), and a square wave at the specified frequency comes out of the GPIO pin.
```sh:
# Using the classic (sysfs) way to control GPIO
$ yes "$(printf '0\n1')" | valve -l TIME > /sys/class/gpio/gpio2/value

# Using the modern (gpiod v2) way to control GPIO
$ yes toggle | valve -l TIME | gpioset -i GPIO2=0
```

The actual measurements (taken using the sysfs method) can be seen in the videos below.
| Frequency | Value given to valve | Video of the measurement |
|---|---|---|
| 100Hz | ≒ 5ms | <https://youtu.be/e8Qx46Gbz-s> |
| 1kHz | ≒ 0.5ms | <https://youtu.be/1WZZLyjXYvg> |
| 10kHz | ≒ 50us | <https://youtu.be/a-X0ydGcugY> |

Even on a cheap computer like the [Raspberry Pi](https://www.raspberrypi.com/), you can see it produces a practical pulse train up to around 1kHz. **The source of the pulse isn't the [`valve` command](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/valve.man.en.md) — yes, that's right, it's the [`yes` command](https://en.wikipedia.org/wiki/Yes_(Unix)).**

For a detailed write-up of this experiment, see [Using the valve Command as a Square-Wave Generator on a Raspberry Pi](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/valve_as_wave_generator_on_raspi.info.en.md).

## Recording and Replaying "hello, world!" Keystrokes

[tokideli](https://github.com/ShellShoccar-jpn/tokideli) has 18 commands, and it'd be a waste to only show off the [`valve` command](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/valve.man.en.md), so let's look at a few more: [`typeliner`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/typeliner.man.en.md), [`linets`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/linets.man.en.md), and [`tscat`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/tscat.man.en.md).

Among programmers, starting anything new traditionally means printing ["hello, world!"](https://en.wikipedia.org/wiki/%22Hello,_World!%22_program). Let's add a twist here and capture the actual timing of typing "hello, world!" on the keyboard, then play it back.

First, let's record. Type the following one-liner, and it'll start recording each character you type — along with its timing — into a file (`hw.log`). Once it starts, type "hello, world!" at whatever pace you like, then finish by pressing [Ctrl]+[D].
```sh:
$ typeliner -e | linets -3 > hw.log
(type "hello, world!" here)
(finish by pressing [Ctrl]+[D])
```
A bit more about these two commands:
* [`typeliner`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/typeliner.man.en.md)
  * Every time you type a character, it sends that character to standard output immediately, with a trailing newline, turning one character into one line. Without this command, nothing reaches standard output until you press [Enter].
  * `-e` makes it echo the typed character locally, so you can see what you just typed.
* [`linets`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/linets.man.en.md)
  * Prepends a timestamp to each incoming line. (The name comes from attaching a timestamp — "ts" — to a "line".)
  * `-3` means "use millisecond precision" (3 digits after the decimal point).
  * With no other options given, the timestamp comes out in `YYYYMMDDhhmmss.nnn` format.

For reference, here's what the resulting `hw.log` looks like:
```text:
20261014004947.408 h
20261014004947.559 e
20261014004947.809 l
20261014004947.956 l
20261014004948.252 o
20261014004948.527 ,
20261014004949.057
20261014004950.255 w
20261014004950.441 o
20261014004950.650 r
20261014004950.818 l
20261014004950.901 d
20261014004951.235 !
20261014004951.803
```

Now for playback. Just run the following one-liner.
```sh:
$ tscat -yZ hw.log
```
More about this command:
* [`tscat`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/tscat.man.en.md)
  * Takes text with a leading timestamp on each line, waits until the time that timestamp indicates, then strips the timestamp and writes that line to standard output — the reverse of [linets](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/linets.man.en.md). (The name means "a `cat` command that understands timestamps — ts".)
  * `-Z` means "base the output timing of later lines on the first line's timestamp." Since recorded timestamps are of course always in the past, without this every line would print immediately.
  * `-y` restores data that was originally character-by-character. Without it, each character would print on its own line (stacked vertically).

So, here's a video of recording and replaying the author's own "hello, world!" typing.
![hello, world! with timing](https://raw.githubusercontent.com/ShellShoccar-jpn/tokideli-assets/main/article_figs/images/hw_typing_demo.gif)

## Recording and Replaying "Dun-dun Dun Dun-Dun" (at Double and Half Speed, Too)

"hello, world!" was just a warm-up. Next, let's type along to the rhythm of the "Dun-dun dun dun-dun" sting from the *Terminator 2* theme, play it back at the same speed, and then also play it back at double speed and half speed.

First, the recording. We'll use a slightly different one-liner this time.
```sh:
$ typeliner -e | linets -3Z > t2.log
(tap any key to the rhythm of "Dun-dun dun dun-dun" here)
(finish by pressing [Ctrl]+[D])
```
This time we gave [`linets`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/linets.man.en.md) the `-Z` option, which means "use a timestamp format showing elapsed time, with the very first incoming line treated as time zero." You'll see why once we get to the trick playback.

Now for playback. First at normal speed — this one-liner adds a beep so the rhythm is easy to follow.
```sh:
$ cat t2.log | sed $(printf 's/$/\007/') | tscat -zZ
```
This time [`tscat`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/tscat.man.en.md) also gets a `-z` option, which tells it that the timestamp format produced by [`linets`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/linets.man.en.md) is elapsed time relative to the first line.

Before showing off the playback, here are the double-speed and half-speed one-liners too:
```sh:
$ cat t2.log | awk '{print $1/2,$2}' | sed $(printf 's/$/\007/') | tscat -zZ
$ cat t2.log | awk '{print $1*2,$2}' | sed $(printf 's/$/\007/') | tscat -zZ
```
The trick is that `awk` halves or doubles the timestamp in the first column. That's exactly why we used elapsed seconds relative to the first line — you can't do this with a calendar-time or UNIX-time timestamp format, right?

So, here's a video covering the whole thing, from recording to playback — you can see the screen flash in time with the "Dun-dun dun dun-dun" rhythm (that's the beep, made visible).
![Recording, playback, double-speed, and half-speed playback of "Dun-dun dun dun-dun"](https://raw.githubusercontent.com/ShellShoccar-jpn/tokideli-assets/main/article_figs/images/t2_typing_demo.gif)

## Echo, Echo! with the `delay` Command

There's a command called [`delay`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/delay.man.en.md) that holds incoming data on standard input for a fixed number of seconds before sending it on to standard output. Let's use it for some quick fun: a mountain echo.

First, type the following one-liner. Once it's running, quickly type "yo-ho!" and press [Enter].
```sh:
$ typeliner -t "" | (tee /dev/stderr | delay 3s) 2>&1
(type "yo-ho!" and [Enter] here...)
(it comes echoing back)
```

Stretch the one-liner further, and the echo doubles up.
```sh:
$ typeliner -t '' | (( tee /dev/stderr | delay 3s ) 2>&1 | tee /dev/stderr | delay 6s) 2>&1
(type "yo-ho!" and [Enter] here (= echo #0))
(echo #1 comes back)
(echo #2 comes back)
(echo #3 comes back)
```

Here's a video of what this looks like:
![Echo](https://raw.githubusercontent.com/ShellShoccar-jpn/tokideli-assets/main/article_figs/images/delay_echo_demo.gif)

By the way, the [`typeliner` command](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/typeliner.man.en.md)'s `-t ''` option tells it to stop inserting a newline after every character, and instead insert whatever string you give it. Here that string is empty, so effectively it means "don't insert anything at all."

## Fixing the "`stdbuf` Doesn't Work" Problem

Finally, let me introduce the [`ptw`](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/ptw.man.en.md) command. In short, it's a souped-up version of [`stdbuf`](https://www.gnu.org/software/coreutils/manual/html_node/stdbuf-invocation.html).

So why does [`stdbuf`](https://www.gnu.org/software/coreutils/manual/html_node/stdbuf-invocation.html) exist in the first place? Many commands, in the name of efficiency, wait until a certain amount of text data has piled up before processing it. People do the same thing: if you had to stamp approval on 100 documents and seal each one into its own envelope, you'd probably lay all 100 out on the desk and stamp them all first, then seal the envelopes afterward, rather than stamping and sealing one at a time. But when a command does that, it wrecks the carefully-timed data you sent it. `stdbuf` exists to prevent exactly that.

Except there are plenty of cases where [`stdbuf`](https://www.gnu.org/software/coreutils/manual/html_node/stdbuf-invocation.html) just doesn't work. Everyone's favorite, Python, is one of them — applying `stdbuf` to the `python3` command has no effect.

For example, save the following Python script (it pulls out high-scoring posts from [Hacker News](https://news.ycombinator.com/))...
```python:
from datetime import datetime
import requests

top_ids = requests.get(
  "https://hacker-news.firebaseio.com/v0/topstories.json"
).json()

for story_id in top_ids[:30]:
  story = requests.get(
    f"https://hacker-news.firebaseio.com/v0/item/{story_id}.json"
  ).json()

  if story.get("score", 0) >= 100:
    score = story["score"]

    # UNIX epoch to YYYY-MM-DD HH:MM:SS
    posted_at = datetime.fromtimestamp(
      story["time"]
    ).strftime("%Y-%m-%d %H:%M:%S")

    title = story["title"]

    print(f"[{score:3d}] {posted_at} {title}")
```

Remember how it behaves when you run the next one-liner:
```
$ python3 best_articles_on_hackernews.py
```
It hits the API, and articles trickle in gradually.
![The original Python script's behavior](https://raw.githubusercontent.com/ShellShoccar-jpn/tokideli-assets/main/article_figs/images/ptw_demo_part1.gif)

Now pipe that output through `grep` to narrow it down to today's articles, and the buffering kicks in — everything dumps out in one big burst at the end.
```
$ python3 best_articles_on_hackernews.py | grep $(date +%Y-%m-%d)
```

So, to suppress the buffering, try applying [`stdbuf`](https://www.gnu.org/software/coreutils/manual/html_node/stdbuf-invocation.html) to `python3` — but the behavior doesn't change at all.
```
$ stdbuf -o L python3 best_articles_on_hackernews.py | grep $(date +%Y-%m-%d)
```

But the [`ptw` command](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/ptw.man.en.md) does the trick.
```
$ ptw python3 best_articles_on_hackernews.py | grep $(date +%Y-%m-%d)
```

Here's a video capturing all three behaviors — none, stdbuf, and ptw — side by side.
![The behavior with none, stdbuf, and ptw](https://raw.githubusercontent.com/ShellShoccar-jpn/tokideli-assets/main/article_figs/images/ptw_demo_part2.gif)

Pretty handy, right? Explaining exactly why it works would take a while, so if you're curious, check out [What Is PTW Command for?](https://github.com/ShellShoccar-jpn/tokideli/blob/main/manual/ptw.info.en.md).

# Wrapping Up

Programmers tend to obsess over raw speed, but there are plenty of situations where timing matters more. However high-end a CD player is, playing 74 minutes of music in one second would make no sense, and even today's buzzworthy physical-AI robots will stumble if they move their limbs too fast while ignoring proper timing.

That's why being able to control timing precisely suddenly makes things more practical. [tokideli](https://github.com/ShellShoccar-jpn/tokideli) is a command collection built exactly for that — and shell scripts that pick it up suddenly become a lot more practical too.

This article leaned into the fun, entertaining side of things, but every command here does something genuinely useful, so put them to work in development, debugging, maintenance, and all sorts of other practical situations.
