---
layout: post
title: Pacific++ 2017 Trip Report
---

The first [Pacific++] conference was held in Christchurch, New Zealand on
October 26th and 27th 2017. I was lucky enough to attend as a speaker so I got
the full conference experience.

## Location

Compared to other C++ conferences, the location of Pacific++ 2017 was very
convenient for me. I live in Auckland, NZ, which is only about an hour and a
half away from Christchurch by plane.

The conference was held at the Sudima Hotel, an easy 5 minute walk from the
airport. Being so close to the airport was very convenient for the attendees who
flew to Christchurch, which was the majority.

I stayed at the Sudima Hotel, which was both convenient and good for socializing
and networking with other attendees.

## Attendees

There were about 65 attendees at this year's conference. Most were from New
Zealand, split between Auckland, Wellington, Christchurch and other, and there
was a really good showing from Sydney.

There were a number of people I know from the [Auckland C++ Meetup] there,
including three of the other speakers.

## Programme

There were 10 talks split over the two days of the conference, with a keynote on
each morning. I won't go into the content of the talks in too much detail as you
can just watch them on the [Pacific++ YouTube channel].

Most of the talks were about 75 minutes long, which I found to be pretty good:
enough time to get into some detail and plenty of time for questions (and rants
disguised as questions) from the audience.

### Day 1

#### _LLVM: A Modern, Open C++ Toolchain_ --- [Chandler Carruth] (Google)

Chandler is always an entertaining speaker and his talk kicked off the
conference with lots of energy. He showed us how easy it is to clone and build
the LLVM toolchain and a few of the tools that are available to you if you build
the latest yourself rather than waiting for your OS to package it.

The things I personally took away from the talk were:

* Build the Release configuration and disable assertions. I knew about the
  former but not the assertion thing; apparently it makes for a significantly
  faster compiler at the end.
* Chandler's build machine is insane---it built LLVM and Clang from a fresh
  clone in under 5 minutes.
* I need to look into the various sanitizers (address, thread, memory, undefined
  behaviour). I was aware of their existence but have never used them. They look
  really useful.

[Chandler Carruth]: https://twitter.com/chandlerc1024

#### _An Introduction to the Coroutines TS_ --- [Toby Allsopp] (WhereScape Software)

This was my talk. Luckily there was a decent break between the end of Chandler's
and the start of mine, so hopefully the decrease in quality wasn't too jarring.

The points I was trying to get across included:

* Coroutines can simplify your application code
* Implementing coroutine library code is a pretty low-level exercise

I'm not sure I got the former across very well as most questions and comments
focused on the latter.

[Toby Allsopp]: https://twitter.com/toby_allsopp

#### _Can we make a faster linked list? Yes, basically yes. Also ignorance is bliss._ --- [Matt Bentley]

Matt talked about his experiences designing a near-drop-in replacement for
`std::list`, which he calls [plf::list]. By dropping support for partial
splicing, his list performs much better than a traditional linked list while
preserving all of the pointer validity and algorithmic complexity guarantees of
`std::list`.

Although I have no use at the moment for any kind of linked list, I really
enjoyed this talk.

[Matt Bentley]: https://twitter.com/xolvenz
[plf::list]: http://plflib.org/list.htm

#### _Debugging with LLVM XRay_ --- [Dean Michael Berris] (Google)

[XRay] is a tool that inserts useless bits of code into certain points in your
executable, for example function entry and exit, and allows them to be patched
with instrumentation code at runtime. Dean explained how this works and went
over some applications of the technology.

The audience was very concerned about the security implications of deploying
executables with these instrumentation hook points in place. This appears to be
an area in need of further research.

[Dean Michael Berris]: https://twitter.com/deanberris
[XRay]: https://llvm.org/docs/XRay.html

#### _Using tasks to simplify concurrency in modern C++_ --- [Christian Blume] (Serato)

Christian talked about his [transwarp] library and how it can simplify setting
up static (known at compile-time) dependency graphs of tasks that can then be
automatically parallelized.

Some of the questions were about how to integrate this approach with a more
fine-grained, dynamic parallelism. In particular, one of Christian's examples
included a task that performed a sum over the square of the differences between
two sequences of numbers, a problem that is, as Christian put it, embarrassingly
parallel. I would expect that this kind of problem would be best addressed using
SIMD instructions or a GPU, completely orthogonal to the task-based concurrency
that transwarp addresses.

[Christian Blume]: https://github.com/bloomen
[transwarp]: https://github.com/bloomen/transwarp

### Day 2

#### _Rethinking Exceptions_ --- [Jason Turner] (CppCast)

Jason was the second keynote speaker. He spoke about his experience adding
`noexcept` specifications to his [ChaiScript] library and how carefully
considering which of his functions could throw exceptions and why led to the
fixing of some performance bugs in his code.

I enjoyed the back-and-forth with the audience; this kind of talk is much better
experienced in person.

This was one of the most relevant talks to what we do at my work and I think it
will help my team to make some progress on the discussions we've been having
about error handling.

[Jason Turner]: https://twitter.com/lefticus
[ChaiScript]: http://chaiscript.com/

#### _Low Latency C++ for Fun and Profit_ --- [Carl Cook] (Optiver)

Carl talked about his experiences optimizing code for high-frequency trading,
where the code you need to be extremely fast is executed extremely rarely. This
presents some unique challenges due to the behaviour of the hardware, which is
mostly concerned with making the frequently-executed code fast.

Some of the techniques that Carl described were shocking to many people with
experience in other kinds of optimization. For example, he talked about
disabling all but one of the cores on a high-end Xeon processor in order to have
exclusive access to the L3 cache.

[Carl Cook]: https://twitter.com/ProgrammerCarl

#### _Type-safe state machines with C++17 std::variant_ --- [Nick Sarten] (Trimble)

This was a talk of two halves, or rather a three-quarters and a quarter. In the
first part of the talk, Nick described the features of `std::variant` and
`boost::variant`, showing how they are used. In the second part of the talk, he
implemented an example state machine in three different ways, one of which being
the use of `std::variant`, and compared the verbosity of the code, type safety
and performance of the different approaches.

One very persistent questioner was concerned about how much of the performance
advantage of the variant-based version over the classic OO-based version was due
to dynamic allocation vs. virtual function dispatch. Nick [tweeted] some updated
performance numbers (just) after the conference which indicated that removing
the dynamic allocation improved the performance of the OO version but that it
remained significantly slower than the variant version.

[Nick Sarten]: https://twitter.com/NickSarten
[tweeted]: https://twitter.com/NickSarten/status/923861082090782720

#### _Postcards from the Cross-platform Frontier_ --- [Sarah Smith] (Smithsoft Pty Ltd)

Sarah told us a very interesting story of her experiences with mobile
development around the time that the iPhone was first released. She then
launched into a live coding demo showing us how easy it is to write a
cross-platform mobile app in C++ using Qt.

I was seriously impressed that Sarah managed to write an app to use the Twitter
API to fetch tweets and display them live on stage in less than 30 minutes.
There was a lot that could have gone wrong!

[Sarah Smith]: https://twitter.com/sarah_j_smith

#### _Equivalence in cross-compilation compiler warnings_ --- [Tom Isaacson] (Navico)

As Tom pointed out at the beginning of his talk, he wasn't talking about
cross-compilation as much as compiling using multiple different toolchains. In
particular, he was comparing Microsoft Visual C++ with GCC.

I will need to go back and look at Tom's list of MSVC warnings that he
recommends that are not included in `/W4` to see if any are useful at work.

[Tom Isaacson]: https://twitter.com/parsley72

## Speaking

This was the second conference presentation I've given, the first being at
[CppCon] 2017 just a month earlier. Although I was speaking on the same topic as
I did at CppCon, the Coroutines TS, I decided to approach it from a slightly
different angle this time, which meant I had a lot more preparation to do than I
had anticipated.

I can highly recommend speaking at a conference. I'm not a naturally confident
public speaker, but I have found the experience to be very rewarding. Preparing
a talk is a lot of work but learning a topic well enough to explain it to others
is a great way to gain a deeper understanding than that needed to feel as though
I understand it myself.

I also find that speaking at a conference, or a user group for that matter,
causes me to get more out of the event as it gives other attendees a reason to
talk to me.

## Not just the talks

The talks that I got to listen to (all but my own) were great, but anyone can
watch them on YouTube. The added value from actually attending a conference
comes from what happens before, between and after the talks.

The gaps between the talks were generous, and snacks were provided, so there was
a lot of mingling and random discussion. This was often a direct follow-on from
the previous talks, but not always.

Lunch was included in the conference ticket on both days and was a buffet in the
hotel restaurant. This provided a slight but important change of scene and the
discussion over lunch was much more social. It was quite nice that the
conference was small enough that it felt like we were all having lunch together.

On the first night there was a Meet the Speakers dinner which attendees could
pay extra for. A perk of being a speaker is that I didn't have to pay to go to
this dinner. The dinner was a lot of fun and I hope the paying guests felt that
it was value for money.

After the end of the conference on the Friday evening, a number of the attendees
loitered at the hotel bar. The crowd gradually diminished as rides or flights
had to be caught. I really enjoyed sharing wedges with the [Wargaming] folks
from Sydney and Dean Michael Berris from Google.

## Conclusion

Pacific++ 2017 was a huge success. I have to take my hat off to Phil Williams
for organizing it and for sticking his neck out because there was no guarantee
of success and he could have been left personally out-of-packet.

I really hope this conference happens again (and I've heard some encouraging
signs). I also hope that, encouraged by the success of the first one, more
companies step up to offer sponsorship and to send employees to attend.

Great job, Phil!

[Pacific++]: https://pacificplusplus.com
[Auckland C++ Meetup]: https://www.meetup.com/Auckland-C-Meetup/
[Pacific++ YouTube channel]: https://www.youtube.com/channel/UCrRR5mU5aqvtZAuEGYfdTjw
[CppCon]: https://cppcon.org/
[Wargaming]: http://wargaming.com/
