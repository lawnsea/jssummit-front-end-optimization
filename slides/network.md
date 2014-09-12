## Deriving a model

We will now derive a model that can be used to analyze `T_scout_fetch` and
`T_app_fetch`.


### You wouldn't download a car

But if you *did*, how long would it take?

Notes:
'<- this is just to fix the stupid markdown syntax highlighting


### A simple model

If `S` is the size of the payload in bytes and `W` is the bandwidth of the
connection in bytes/second, then it will take

`T = S / W`

to download the payload.

Notes:
This is the intuitive mental model I had of downloading data


### Simple, but also wrong

Bandwidth measures how large a message a sender can transmit in a unit time. It
does not account for the time it takes the message to travel to the receiver.

The client has to transmit a request to the server and the server then has to
send the response. The time it takes these messages to travel is not accounted
for in our model.


### Propagation delay

The time it takes for a message to travel from sender to receiver is known as
**propagation delay**. It approaches the speed of light and, critically, is a
function of *distance* rather than the size of the payload.


### A slightly less simple model

Our earlier model can be refined to account for the time it takes the request
and response to travel between client and server by adding a new variable, `P`,
representing the propagation delay in seconds:

`T = 2 * P + S / W`


### A small example

```
S = 40,000 bytes
W = 21.2 megabits / second = 2,625,000 bytes/second
P = 0.0175 seconds

T = 2 * P + S / W
  = 2 * 0.0175 + 40,000 / 2,625,000
  = 0.035 + 0.015
  = 50ms
```

Notes:
Note that latency accounts for 70% of the total
W and P from terrestrial averages in http://www.fcc.gov/reports/measuring-broadband-america-2014


### Everything is TCPerrible

Notes:
This model is still not right
It does not account for the fact that browsers make HTTP requests over TCP
connections
The creators of TCP, Alex Sexton and Paul Irish, claim that...


### Taking Care of Packets

TCP provides reliable, connection-oriented delivery of packets across the
internet. Every HTTP request is made via a TCP connection.

Setting up a TCP connection requires a round trip to the server.


### Ok this model is getting a bit more complicated

The propagation delay incurred by a round trip to the server is known as the
**round trip time**. Let `RTT` be the round trip time, `2 * P`. Our model now
looks like so:

`T = 2 * RTT + S / W`

This adds another 35ms to the time required to fetch a 40kB file, for a total of
85ms, 82% of which is due to propagation delay.


### Slow start is slow

The sender sets a maximum number of unacknowledged packets that they will send,
called the *congestion window*, or `cwnd`. Typically, the congestion window is 4
to 10 packets, or 4-15kB.

For each acknowledgement the sender receives, the congestion window is increased
by 2 packets. This has the effect of roughly doubling `cwnd` every RTT.


The upshot is that early in the lifetime of a TCP connection, its usable
bandwidth is lower than the available bandwidth - sometimes *much* lower.

For example, during the first RTT, our example connection is throttled to
15kB / 35 ms, which is a little over 3.4Mb/s, or about **16% of the available
bandwidth**.


### Slow loris model

If we define the initial congestion window in bytes to be `C`, then we can find
the number of roundtrips to download a payload of size `S`:

`R = ceil(log_2(S / C + 1))`

Our model can be updated to account for slow start like so:

`T = RTT * (1 + R) + S / W`


Returning to our example, if `C` is 15,000 bytes, then the number of roundtrips
to fetch our payload is:

```
R = ceil(log_2(S / C + 1))`
  = ceil(log_2(40,000 / 15,000 + 1))
  = 2
```

Plugging that into our model:

```
T = RTT * (1 + R) + S / W`
  = 0.035 * (1 + 2) + 40,000 / 2,625,000
  = 120ms
```

Of that time, **87.5%** is due to propagation delay.


### Is this model real life?


LOL no. The internet is complicated.

![The Internet](./assets/internet.svg)

FIXME: make this attribution visible
By User:Ludovic.ferre (Internet_Connectivity_Overview.svg) [<a href="http://creativecommons.org/licenses/by-sa/3.0">CC-BY-SA-3.0</a>], <a href="http://commons.wikimedia.org/wiki/File%3AInternet_Connectivity_An_Overview.svg">via Wikimedia Commons</a>


However, this model is useful in that it approximates reality and makes
predictions that we can test.


## Instrumentation

Instrumenting network requests for third-party JavaScript applications is
challenging. It is not possible to instrument network requests directly across
all the browsers that we support.

Instead we record the time from the scout requesting the application JavaScript
until it actually begins to execute. We also record the time from the
application starting until it renders.

Notes:
Mention Resource Timing API


## Identifying an opportunity

While we fetch the CSS and JavaScript in parallel, we inject the `link` tag for
the CSS before the `script` tag for the JavaScript.

One more fact about TCP - that browsers reuse connections when possible - led us
to the following question:

Does it matter whether we request the JavaScript (~228kB) or the CSS (~45kB)
first?


### Model prediction

We start with the following values:

`C = 14,600 bytes`

`W = 625,000 bytes/second`

`RTT = 0.042 seconds`


#### Status quo

We expect that the CSS will be fetched over the existing connection, and thus
have a 30kB congestion window, while the JavaScript will be fetched over a
connection with a 15kB congestion window. Our model predicts the following:

```
T_fetch_CSS = 0.042 * ceil(log_2(45,000 / 29,200 + 1)) + 45,000 / 625,000
            = 0.042 * 2 + 45,000 / 625,000
            = 156ms
```

```
T_fetch_JS = 0.042 * (1 + ceil(log_2(228,000 / 14,600 + 1))) + 228,000 / 625,000
           = 0.042 * 6 + 228,000 / 625,000
           = 617ms
```


#### Swapped

With the requests swapped, our model predicts the following:

```
T_fetch_CSS = 0.042 * (1 + ceil(log_2(45,000 / 14,600 + 1))) + 45,000 / 625,000
            = 0.042 * 2 + 45,000 / 625,000
            = 198ms
```

```
T_fetch_JS = 0.042 * ceil(log_2(228,000 / 29,200 + 1)) + 228,000 / 625,000
           = 0.042 * 4 + 228,000 / 625,000
           = 533ms
```

So, we expect `T_fetch_CSS` to go up by 26% and `T_fetch_JS` to go down by 14%.


## The experiment


### Hypothesis

Injecting the application JavaScript tag before the CSS tag will decrease
`T_fetch_app`.


### Experiment design

I configured Charles proxy to map the domain our content is hosted on to a
web server running on my laptop and throttled the connection like so:

- Bandwidth down: 5Mbps
- Bandwidth up: 1Mbps
- RTT: 42ms
- MTU (maximum packet size): 1460 bytes

I then ran 10 trials in Chrome and recorded the Resource Timing results for the
application JavaScript and styles.

Notes:
Briefly discuss failed original design


### Results

 | Status Quo | Swapped | Delta | % Delta
-: | - | - | - | -
CSS | 178ms | 217ms | 39ms | +22%
JS | 650ms | 631ms | -19ms | -3%


## Analysis

It turns out that Chrome opens extra TCP connections, so the model predicts an
extra round trip for the second request. When this is taken into account, the
model predicts a more model 7% decrease for `T_fetch_JS`.

Because of the low risk associated with making the change, we went ahead and did
so without further experimentation (testing in other browsers, e.g.).
