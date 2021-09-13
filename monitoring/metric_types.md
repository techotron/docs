# Metric Types
A brief description of different metric types and what they can be used for

## Sources
- [Prometheus Metrics](https://tomgregory.com/the-four-types-of-prometheus-metrics/)

## Counter
- For any value that increases (eg error or request count)
- Should **not** be used for any value that decreases
- When you want to record a value that only increments
- When you want to query _how fast_ the value is increasing (rate)

**Use Cases**
- request count
- tasks completed
- error count

## Gauges
- For values which go up or down (eg memory utilisation or items in a queue)
- Don't use this if you need to query the rate of change

**Use Cases**
- memory usage
- queue size
- number of requests in progress

## Histograms
- Measures the frequency of value observations that fall into specific pre-defined buckets
- Eg, number request durations which fall into .005, .01, .1, .5, .75, 1, 5, 10 buckets.
- When you want to use measurments of a value to calculate averages or percentiles. 
- When you're happy with an approximate (this may be a Prometheus consideration)
- You know the range of values will be (in order to setup the bucket definitions beforehand)
- Quantiles are calculated on the Prometheus server (rather than on the application server)

**Use Cases**
- request duration
- response size

## Summaries
- Preceeded histograms in Prometheus and are very similar to them. Recommendation is to use Historgrams where possible.
- Summaries are calculated on the application server so values aren't aggregated across application servers.
- Histograms require knowledge of the range of values up front. If you don't know these, then you can use Summaries instead.
- When you want to use measurments of a value to calculate averages or percentiles. 

**Use Cases**
- request duration
- response size
