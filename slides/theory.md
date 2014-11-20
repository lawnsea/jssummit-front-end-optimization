# Theory and method of performance optimization

Notes:
- Now lay out the method that I take when approaching a performance project


<div class="center-image">
![Systems Performance: Enterprise and the Cloud](./assets/systems-performance-brendan-gregg.jpeg)
</div>

Notes:
- Brendan Gregg, currently of Netflix, formerly of Joyent and Sun


## What is performance optimization?


At its core, performance optimization is the process of identifying inefficient
use of system resources and improving the situation.

It is both an art *and* a science. It requires creativity, persistence,
discipline, and attention to detail.


## How is performance optimized?

The major activities of optimization include

- setting goals
- modeling
- instrumentation
- identifying optimization opportunities
- experimentation
- analysis

Notes:
- I will go into detail for each of these
- At a high level, you
  - decide what you want to improve
  - gather data
  - try something
  - see if it worked
  - repeat
- I will be talking in terms of improving the speed of a program, but these
  techniques also apply to other perf concerns, such as reducing memory usage


## Setting goals

The first step is to decide what you want to accomplish.

Goals must be attainable.

Goals should not be arbitrary, they should be driven by business requirements,
user experience, and other real-world motivations.

Goals must be measurable.

Notes:
- an unnamed client wanted us to load in 30ms, which is less than SF<->NYC
  latency


## Modeling

A **model** is an abstract, mathematical representation of a system. Models are
never completely precise or accurate, but good ones allow us to analyze and make
predictions about changes to the system.

Notes:
gonna need math, sorry not sorry


## Instrumentation

**Instrumentation** is the process of modifying an application to gather and
report performance measurement data.


### User Timing API

The [User Timing API](http://caniuse.com/user-timing) provides a means to
measure durations with sub-millisecond precision. It exposes a simple but
powerful API that is easily
[polyfillable](https://www.npmjs.org/package/usertiming).

Many development tools - including the Chrome Dev Tools and WebPagetest - detect
and display measurements captured with the User Timing API.


### Collection

This is the hard part: getting performance data from the browser to a server for
later analysis. There are three broad approaches to the problem:

- Fully-hosted (Google Analytics, Splunk, e.g.)
- Self-hosted off the shelf software (statsd, e.g.)
- Roll your own

Any of these will work, but solving this problem in a manner that is reliable
and scalable is *hard*.


## Identifying Optimization Opportunities


### Amdahl's Law cannot be bargained with

The maximum performance improvement to a computation that can be achieved by
optimizing a portion of that computation is bounded by the current performance
of that portion.

For example, if a particular function takes 100ms to execute and 10ms of that
time is spent in a helper function, even making that helper *infinitely fast*
will garner at most a 10% improvement.

Notes:
- often discussed in the context of parallelization


## Experimentation


### Experimental method

- Ask a question
- Form a hypothesis
- Test the hypothesis
- Analyze the collected data

A good experiment is *focused* and *repeatable*.

Notes:
- Q: will spriting my images reduce load time?
- H: spriting my images will reduce median load time by 25%
- T: load the page 30x each in Firefox, Chrome, and IE, with images sprited,
  then repeat with images unsprited
- A: Find the median of each dataset and compare


## Analysis

<div class="center-image">
![A typical timing distribution](./assets/distribution.png)
</div>

Performance data rarely conforms to a standard distribution, so mean and
standard deviation are typically not very helpful.

Instead, examine the minimum and maximum, as well as the median and two or three
other percentiles, such as the 75th, 95th, and 99th.

Notes:
- distributions
- probability density
- modal vs multimodal
- min, max, median and percentiles sted of mean


Plotting experimental results can be very helpful.

Useful visualization tools include spreadsheet programs like Excel or
LibreOffice Calc, gnuplot, and R.

Another excellent visualization is the waterfall diagram found in most browser
developer tools.

I've built a little tool called [Sapa](https://github.com/lawnsea/sapa) that
tranforms User Timeline data into the Chrome Timeline format.

Notes:
- waterfall
- chrome timeline
- sapa


## Closing the loop

After analysis is complete, it is time to begin the process again.

Analysis may drive refinements to your model, suggest instrumentation points, or
highlight new opportunities for optimization.
