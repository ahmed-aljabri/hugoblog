---
title: "To ABC or not to ABC?"
date: 2026-04-05
description: "RFC 3465 never made the TCP standard, yet Linux, FreeBSD, and lwIP each quietly opted for their own variation of it."
tags: ["TCP", "congestion-control", "protocols", "networking"]
draft: false
---

As I've been working on my MPhil dissertation project - evaluating a prototype of [TARR](https://datatracker.ietf.org/doc/draft-ietf-tcpm-ack-rate-request/) in ns-3 (part of me wishes it was called [TARS](https://interstellarfilm.fandom.com/wiki/TARS)😁 ), I came across an interesting variation in how the sender congestion window is incremented during slow-start and congestion avoidance phases.

[RFC 3465](https://datatracker.ietf.org/doc/html/rfc3465) — *Appropriate Byte Counting* (ABC) — defines a modification to TCP's slow start and congestion avoidance phases. Instead of incrementing the congestion window by one segment per ACK received, ABC proposes incrementing based on the number of bytes acknowledged. This approach aims to address a real problem --- standard ACK-based incrementing rewards senders unfairly when ACKs are delayed or compressed. ABC aligns growth with actual data delivered. The RFC has been around since 2003 and it remains explicitly marked **Experimental** - it was never folded into the TCP standard.

When it comes to TARR - a new option that allows a TCP sender to request the receiver change their acknowledgment frequency (further than existing delayed-ack mechanisms - ACK every 2 segments - or immediately at times) - this has direct implications on the congestion window growth and subsequently, the throughput realised. Stretched ACKs are expected to slow down cwnd growth during slow start without ABC, and with ABC, create bursty flows (e.g. cwnd increases by 10xSMSS upon receiving a single ACK acknowledging 10 segments). Both of these expectations are undesired!

The latest [TARR draft](https://datatracker.ietf.org/doc/draft-ietf-tcpm-ack-rate-request/) specifically states:

> In order to avoid slow cwnd opening, a TCP sender SHOULD NOT use the
   TARR option to produce Stretch ACKs during Slow Start.

The draft also mentions that ABC may solve the slow congestion window opening problem, but is still only an experimental method. Using a basic simulation in ns3, I was able to confirm this expectation:

![ABC](/img/hs_cwnd_evolution.png)

In standard TCP slow-start phase (right), as expected, using stretched acks slows down the congestion window opening notably compared to the standard method or even existing delayed-ack approach. Likewise, with ABC enabled (left), stretched acks do not suffer as much! 

Motivated by the initial results and the discussion I had with two of TARR's authors (Prof. Jon Crowcroft and Carles Gomez), I was interested to see if any notable TCP stacks actually use ABC. Here is what I found so far:



## Linux

Looking at the kernel's slow-start path, they explicitly state:

>  We do not implement RFC3465 Appropriate Byte Counting (ABC) per se but something better;)

```c
__bpf_kfunc u32 tcp_slow_start(struct tcp_sock *tp, u32 acked)
{
	u32 cwnd = min(tcp_snd_cwnd(tp) + acked, tp->snd_ssthresh);

	acked -= cwnd - tcp_snd_cwnd(tp);
	tcp_snd_cwnd_set(tp, min(cwnd, tp->snd_cwnd_clamp));

	return acked;
}
```

This is ABC in essence (taking into account the bytes acknowledged `acked` when updating congestion window), just capped by the sender's slow-start threshold `snd_ssthresh` - with that it seems they allow the TCP flow cwnd to overshoot at first (ssthresh is set to a very large value at the start) before it is adjusted after experiencing loss/congestion.

## FreeBSD

FreeBSD, I have found, takes a stricter approach but closer to the actual ABC RFC recommendation, capping the byte increment by a variable - default is 2x SMSS (Sender Maximum Segment Size) - despite the total bytes acknowledged, the cwnd will not jump past the 2xSMSS cap (or the configurable variable) in one go.

```c
/*
	 * Regular in-order ACK, open the congestion window.
	 * The congestion control state we're in is slow start.
	 *
	 * slow start: cwnd <= ssthresh
	 *
	 * slow start and ABC (RFC 3465):
	 *   Grow cwnd exponentially by the amount of data
	 *   ACKed capping the max increment per ACK to
	 *   (abc_l_var * maxseg) bytes.
	 *
	 * slow start without ABC (RFC 5681):
	 *   Grow cwnd exponentially by maxseg per ACK.
	 */
	if (V_tcp_do_rfc3465) {
		/*
		 * In slow-start with ABC enabled and no RTO in sight?
		 * (Must not use abc_l_var > 1 if slow starting after
		 * an RTO. On RTO, snd_nxt = snd_una, so the
		 * snd_nxt == snd_max check is sufficient to
		 * handle this).
		 *
		 * XXXLAS: Find a way to signal SS after RTO that
		 * doesn't rely on tcpcb vars.
		 */
		uint16_t abc_val;

		if (ccv->flags & CCF_USE_LOCAL_ABC)
			abc_val = ccv->labc;
		else
			abc_val = V_tcp_abc_l_var;
		if (CCV(ccv, snd_nxt) == CCV(ccv, snd_max))
			incr = min(ccv->bytes_this_ack,
			           ccv->nsegs * abc_val * mss);
		else
			incr = min(ccv->bytes_this_ack, mss);
	}
	/* ABC is on by default, so incr equals 0 frequently. */
	if (incr > 0)
		return min(cw + incr, TCP_MAXWIN << CCV(ccv, snd_scale));
	else
		return cw;
}


```

## LwIP

On the IoT side of things, LwIP - a popular TCP stack for constrained devices - takes a similar approach. The sender's congestion window growth is capped at a hardcoded value: 2xSMSS (goes down to 1xSMSS if close to an RTO event).

```c
if (pcb->state >= ESTABLISHED) {
        if (pcb->cwnd < pcb->ssthresh) {
          tcpwnd_size_t increase;
          /* limit to 1 SMSS segment during period following RTO */
          u8_t num_seg = (pcb->flags & TF_RTO) ? 1 : 2;
          /* RFC 3465, section 2.2 Slow Start */
          increase = LWIP_MIN(acked, (tcpwnd_size_t)(num_seg * pcb->mss));
          TCP_WND_INC(pcb->cwnd, increase);
          LWIP_DEBUGF(TCP_CWND_DEBUG, ("tcp_receive: slow start cwnd %"TCPWNDSIZE_F"\n", pcb->cwnd));
        } 
```



## Takeaways

Despite ABC being an experimental feature, there are some notable stacks that implement variations of ABC out there! Which means that for new extensions to TCP like TARR, it would be useful to evaluate how TARR behaves with ABC enabled. My initial experiments so far show promising results but need to be evaluated more extensively before making a proper case. For instance, using stretched acks during slow-start may lead to reversed results if RTTs drift to a point that the acks are not received in time to maintain the feedback loop, with that potentially causing an RTO event at which the cwnd will be drastically reduced. 

Despite that, given how ABC seems to address the slow congestion window opening during slow-start, I could imagine cases where using stretched ACKs from the start, with ABC enabled, may not be a bad idea when compared against changing the ack ratio multiple times in a connection's lifetime (e.g. risk of middlebox interference) - however I am yet to observe and verify the cost of repetitive TARR requests to change the ack ratio during a connection.

Another thing worth noting is that although standardisation can often drive adoption, some ideas propagate far ahead of standardisation efforts - the variety of ABC-like implementations is an example of that. From an implementers' perspective, it makes it more challenging to evaluate new extensions (what ABC variant do we test with?) - possibly highlighting where standardisation catching up would help.



