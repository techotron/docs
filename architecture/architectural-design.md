# Architectural Design
A collection of notes on architecture and design patterns.

## Contents

<!-- toc -->

- [Designs](#designs)
  * [Video Streaming](#video-streaming)
- [Infrastructure](#infrastructure)
  * [Caching](#caching)
  * [Message Queues](#message-queues)

<!-- tocstop -->

## Designs
### Video Streaming
- [Source](https://medium.com/@karan99/system-design-netflix-6962b4f6222)

#### Video Processing Pipeline
Raw content in -> Split file into chunks -> Content Filter -> Transcoder -> Quality Conversion

**File Chunker**

**Content Filter**

**Transcoder**

**Quality Conversion**


## Infrastructure Patterns
A succinct collection of notes on the use cases of the various technologies which are discussed in the designs section

### Caching
Redis

### Message Queues
Async 

### Pipelines
Eg content processing pipeline 

### State
Sharding
Relational (harder to scale)
NoSQL (easier to scale)
