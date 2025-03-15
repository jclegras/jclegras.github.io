+++
date = '2025-03-15T14:24:00+01:00'
draft = false
title = 'What is tail latency ?'
tags = ["performance", "tail latency", "percentile"]
toc = true
+++

## Context

**Tail latency** refers to the slowest response times experienced by a small percentage of requests in a system. These are typically measured at the **99th percentile (p99) or 99.9th percentile (p999)**.

## How tail latency arises in a fan-out model

In the **fan-out model** (where a request is split across multiple components and waits for all to complete), the **overall request latency** is determined by the **slowest** component.

Even if each component is **usually fast**, a **small probability** of slowness in each one can add up, making slow responses much more frequent when multiple components are involved.

### Probability equation

$$P(at least one slow) = 1 - P(fast request)^n = 1 - (1 - P(slow request))^n$$

## Why tail latency matters

* **Bad user experience**: Some users get slow responses, even if the average latency looks fine.
* **Scaling challenges**: Adding more components makes tail latency **worse** instead of better.
* **System reliability issues**: A few slow components can delay the entire system.

## Examples of tail latency in real life

* A cloud service like AWS Lambda may complete most requests very quickly, but some can take a longer time to execute because of [cold starts](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html#cold-start-latency)

> Cold starts typically occur in under 1% of invocations. The duration of a cold start varies from under 100 ms to over 1 second

* A search engine (like Google) may query multiple servers, but the overall search time is dictated by the slowest responding server.

## Application

**Problem 1**: If you fan out processing to 10 components and wait for all of them to complete what is the probability that a user request experiences the 99th percentile latency?

$$
\begin{aligned}
P(\text{at least one slow}) &= 1 - (1 - P(\text{slow request}))^{10} \\\
P(\text{at least one slow}) &= 1 - (1 - 0.01)^{10} \\\
P(\text{at least one slow}) &= 1 - 0.99^{10} \\\
P(\text{at least one slow}) &\approx 0.09
\end{aligned}
$$

If each component has a **1% chance** of being slow, a request fanned out to **10 components** will experience **at least one slow response** in **9% of cases**.

**Problem 2**: If you fan out processing to 10 components and wait for all of them to complete what is the probability that a user request experiences the 95th percentile latency?

$$
\begin{aligned}
P(\text{at least one slow}) &= 1 - (1 - P(\text{slow request}))^{10} \\\
P(\text{at least one slow}) &= 1 - (1 - 0.05)^{10} \\\
P(\text{at least one slow}) &= 1 - 0.95^{10} \\\
P(\text{at least one slow}) &\approx 0.4
\end{aligned}
$$

If each component has a **5% chance** of being slow, a request fanned out to **10 components** will experience **at least one slow response** in **40% of cases**.

### More cases on p99

| n (number of components)      | P(at least one slow) |
| ----------- | ----------- |
| 1 (no fan-out)      | 1%       |
| 2   | 1.99%       |
| 5   | 4.9%        |
| 10   | 9.56%        |
| 20   | 18.21%        |
| 50   | 39.5%        |

## Conclusion

* The more components in the fan-out, the higher the probability of hitting a slow response.
* This results in a long tail of slow responses, meaning some users will see significantly worse performance than the average user.

## Miscellaneous

### Why this probability equation

$$P(at least one slow) = 1 - P(fast request)^n = 1 - (1 - P(slow request))^n$$

Let’s try to breakdown it using [the complementary event](https://en.wikipedia.org/wiki/Complementary_event):

$$P(at least one slow) = 1 - P(all fast)$$

If 

$$P(fast request)$$ 
is the probability of a fast component, then:

$$P(fastrequest) = 1 - P(slowrequest)$$

Therefore:

$$
\begin{aligned}
P(\text{allfast}) &= P(\text{fastrequest}))^{n} \\\
P(\text{allfast}) &= (1 - P(\text{slowrequest}))^{n}
\end{aligned}
$$

So

$$P(at least one slow) = 1 - P(fast request)^n = 1 - (1 - P(slow request))^n$$

### How to calculate a worst case scenario

If you fan out processing to 5 components and wait for all of them to complete what is the probability that a user request experiences the worst case scenario which is all components are slow?

We suppose here the probability of a single component being slow is

$$P(slowrequest)=0.1$$

So:

$$
\begin{aligned}
P(\text{allcomponentsslow}) &= \prod_{i=1}^{5} P(\text{slowrequest}) = P(\text{slowrequest})^{5} \\\
P(\text{allcomponentsslow}) &= 0.1^{5} \\\
P(\text{allcomponentsslow}) &= 10^{-5} = 0.00001 = 0.001\% \\\
\end{aligned}
$$

If each component has a **10% chance** of being slow, a request fanned out to **5 components** will experience **a worst scenario response** in **0.001% of cases**.

## Resources

* [Patterns Of Low Latency](https://www.p99conf.io/session/patterns-of-low-latency/)
* [The tail at scale - Jeffrey Dean, Luiz André Barroso](https://dl.acm.org/doi/pdf/10.1145/2408776.2408794)
