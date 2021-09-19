# Four Golden Signals
These are the **minimum** metrics you should gather for all services.

## Source
- Google Site Reliability Engineering Book

## Latency
- Time it takes to service a request
- Ensure you distinguish between latency of successful requests and failed requests
- Track error latency rather than filter out errors

## Traffic
- A measure of how much demand is being placed on the system:
- This will differ per type of service eg:
  - **Web Service:** HTTP requests per second
  - **Audio Streaming:** Network I/O rate or concurrent sessions
  - **Key/Value storage systems:** Transactions and retirevals per second

## Errors
- Rate of requests that fail eg either explicitly (5XXs) or implicitly (2XX but wrong content or slower response time than commitment)
- Could be made up of custom metrics that represent an error to the application

## Saturation
- How "full" the service is, in a memory contrained system, show memory; in an I/O constrained system, show networking.
- Latency increases are often a leading indicator of saturation. Measuring the 99th percentile will help to discover these types of issues.
