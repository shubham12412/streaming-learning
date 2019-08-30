https://learning.oreilly.com/library/view/streaming-systems/9781491983867/ch03.html


***Consider any pipeline that ingests data and outputs results continuously. We wish to solve the general problem of when it is safe to call an event-time window closed, meaning that the window does not expect any more data. To do so we would like to characterize the progress that the pipeline is making relative to its unbounded input.***

One naive approach for solving the event-time windowing problem would be to simply base our event-time windows on the current processing time. As we saw in Chapter 1, we quickly run into trouble—data processing and transport is not instantaneous, so processing and event times are almost never equal. Any hiccup or spike in our pipeline might cause us to incorrectly assign messages to windows. Ultimately, this strategy fails because we have no robust way to make any guarantees about such windows.

Another intuitive, but ultimately incorrect, approach would be to consider the rate of messages processed by the pipeline. Although this is an interesting metric, the rate may vary arbitrarily with changes in input, variability of expected results, resources available for processing, and so on. Even more important, rate does not help answer the fundamental questions of completeness. Specifically, rate does not tell us when we have seen all of the messages for a particular time interval. In a real-world system, there will be situations in which messages are not making progress through the system. This could be the result of transient errors (such as crashes, network failures, machine downtime), or the result of persistent errors such as application-level failures that require changes to the application logic or other manual intervention to resolve. Of course, if lots of failures are occurring, a rate-of-processing metric might be a good proxy for detecting this. However a rate metric could never tell us that a single message is failing to make progress through our pipeline. Even a single such message, however, can arbitrarily affect the correctness of the output results.

We require a more robust measure of progress. To arrive there, we make one fundamental assumption about our streaming data: each message has an associated logical event timestamp. This assumption is reasonable in the context of continuously arriving unbounded data because this implies the continuous generation of input data. In most cases, we can take the time of the original event’s occurrence as its logical event timestamp. With all input messages containing an event timestamp, we can then examine the distribution of such timestamps in any pipeline. Such a pipeline might be distributed to process in parallel over many agents and consuming input messages with no guarantee of ordering between individual shards. Thus, the set of event timestamps for active in-flight messages in this pipeline will form a distribution

Messages are ingested by the pipeline, processed, and eventually marked completed. Each message is either “in-flight,” meaning that it has been received but not yet completed, or “completed,” meaning that no more processing on behalf of this message is required.

https://learning.oreilly.com/library/view/streaming-systems/9781491983867/assets/stsy_0301.mp4

***There is a key point on this distribution, located at the leftmost edge of the “in-flight” distribution, corresponding to the oldest event timestamp of any unprocessed message of our pipeline. We use this value to define the watermark:***

***The watermark is a monotonically1 increasing timestamp of the oldest work not yet completed.***


There are two fundamental properties that are provided by this definition that make it useful:

1) ***Completeness***
If the watermark has advanced past some timestamp T, we are guaranteed by its monotonic property that no more processing will occur for on-time (nonlate data) events at or before T. Therefore, we can correctly emit any aggregations at or before T. In other words, the watermark allows us to know when it is correct to close a window.

2) ***Visibility***
If a message is stuck in our pipeline for any reason, the watermark cannot advance. Furthermore, we will be able to find the source of the problem by examining the message that is preventing the watermark from advancing.

--------------------------------------------------------------------------------------------------------------------


### Source Watermark Creation
Where do these watermarks come from? To establish a watermark for a data source, we must assign a logical event timestamp to every message entering the pipeline from that source. As Chapter 2 informs us, all watermark creation falls into one of two broad categories: perfect or heuristic. 

Notice that the distinguishing feature is that perfect watermarks ensure that the watermark accounts for all data, whereas heuristic watermarks admit some late-data elements.

After the watermark is created as either perfect or heuristic, watermarks remain so throughout the rest of the pipeline. As to what makes watermark creation perfect or heuristic, it depends a great deal on the nature of the source that’s being consumed. To see why, let’s look at a few examples of each type of watermark creation.

### Perfect Watermark Creation
Perfect watermark creation assigns timestamps to incoming messages in such a way that the resulting watermark is a strict guarantee that no data with event times less than the watermark will ever be seen again from this source. Pipelines using perfect watermark creation never have to deal with late data; that is, data that arrive after the watermark has advanced past the event times of newly arriving messages. However, perfect watermark creation requires perfect knowledge of the input, and thus is impractical for many real-world distributed input sources. Here are a couple of examples of use cases that can create perfect watermarks:

1) ***Ingress timestamping***

A source that assigns ingress times as the event times for data entering the system can create a perfect watermark. In this case, the source watermark simply tracks the current processing time as observed by the pipeline. This is essentially the method that nearly all streaming systems supporting windowing prior to 2016 used.

Because event times are assigned from a single, monotonically increasing source (actual processing time), the system thus has perfect knowledge about which timestamps will come next in the stream of data. As a result, event-time progress and windowing semantics become vastly easier to reason about. The downside, of course, is that the watermark has no correlation to the event times of the data themselves; those event times were effectively discarded, and the watermark instead merely tracks the progress of data relative to its arrival in the system.

2) ***Static sets of time-ordered logs***

A statically sized2 input source of time-ordered logs (e.g., an Apache Kafka topic with a static set of partitions, where each partition of the source contains monotonically increasing event times) would be relatively straightforward source atop which to create a perfect watermark. To do so, the source would simply track the minimum event time of unprocessed data across the known and static set of source partitions (i.e., the minimum of the event times of the most recently read record in each of the partitions).

Similar to the aforementioned ingress timestamps, the system has perfect knowledge about which timestamps will come next, thanks to the fact that event times across the static set of partitions are known to increase monotonically. This is effectively a form of bounded out-of-order processing; the amount of disorder across the known set of partitions is bounded by the minimum observed event time among those partitions.

Typically, the only way you can guarantee monotonically increasing timestamps within partitions is if the timestamps within those partitions are assigned as data are written to it; for example, by web frontends logging events directly into Kafka. Though still a limited use case, this is definitely a much more useful one than ingress timestamping upon arrival at the data processing system because the watermark tracks meaningful event times of the underlying data.



-----------------------------------------------------------------------------------------------------------------

### Heuristic Watermark Creation
Heuristic watermark creation, on the other hand, creates a watermark that is merely an estimate that no data with event times less than the watermark will ever be seen again. Pipelines using heuristic watermark creation might need to deal with some amount of late data. Late data is any data that arrives after the watermark has advanced past the event time of this data. Late data is only possible with heuristic watermark creation. If the heuristic is a reasonably good one, the amount of late data might be very small, and the watermark remains useful as a completion estimate. The system still needs to provide a way for the user to cope with late data if it’s to support use cases requiring correctness (e.g., things like billing).

For many real-world, distributed input sources, it’s computationally or operationally impractical to construct a perfect watermark, but still possible to build a highly accurate heuristic watermark by taking advantage of structural features of the input data source. 

Following are two example for which heuristic watermarks (of varying quality) are possible:

1) ***Dynamic sets of time-ordered logs***

Consider a dynamic set of structured log files (each individual file containing records with monotonically increasing event times relative to other records in the same file but with no fixed relationship of event times between files), where the full set of expected log files (i.e., partitions, in Kafka parlance) is not known at runtime. Such inputs are often found in global-scale services constructed and managed by a number of independent teams. In such a use case, creating a perfect watermark over the input is intractable, but creating an accurate heuristic watermark is quite possible.

By tracking the minimum event times of unprocessed data in the existing set of log files, monitoring growth rates, and utilizing external information like network topology and bandwidth availability, you can create a remarkably accurate watermark, even given the lack of perfect knowledge of all the inputs. This type of input source is one of the most common types of unbounded datasets found at Google, so we have extensive experience with creating and analyzing watermark quality for such scenarios and have seen them used to good effect across a number of use cases.

2) ***Google Cloud Pub/Sub***

Cloud Pub/Sub is an interesting use case. Pub/Sub currently makes no guarantees on in-order delivery; even if a single publisher publishes two messages in order, there’s a chance (usually small) that they might be delivered out of order (this is due to the dynamic nature of the underlying architecture, which allows for transparent scaling up to very high levels of throughput with zero user intervention). As a result, there’s no way to guarantee a perfect watermark for Cloud Pub/Sub. The Cloud Dataflow team has, however, built a reasonably accurate heuristic watermark by taking advantage of what knowledge is available about the data in Cloud Pub/Sub. The implementation of this heuristic is discussed at length as a case study later in this chapter.



Consider an example where users play a mobile game, and their scores are sent to our pipeline for processing: you can generally assume that for any source utilizing mobile devices for input it will be generally impossible to provide a perfect watermark. Due to the problem of devices that go offline for extended periods of time, there’s just no way to provide any sort of reasonable estimate of absolute completeness for such a data source. You can, however, imagine building a watermark that accurately tracks input completeness for devices that are currently online, similar to the Google Pub/Sub watermark described a moment ago. Users who are actively online are likely the most relevant subset of users from the perspective of providing low-latency results anyway, so this often isn’t as much of a shortcoming as you might initially think.



With heuristic watermark creation, broadly speaking, the more that is known about the source, the better the heuristic, and the fewer late data items will be seen. There is no one-size-fits-all solution, given that the types of sources, distributions of events, and usage patterns will vary greatly. But in either case (perfect or heuristic), after a watermark is created at the input source, the system can propagate the watermark through the pipeline perfectly. This means perfect watermarks will remain perfect downstream, and heuristic watermarks will remain strictly as heuristic as they were when established. This is the benefit of the watermark approach: you can reduce the complexity of tracking completeness in a pipeline entirely to the problem of creating a watermark at the source.

--------------------------------------------------------------------------------------------------------------------


### Watermark Propagation
So far, we have considered only the watermark for the inputs within the context of a single operation or stage. However, most real-world pipelines consist of multiple stages. Understanding how watermarks propagate across independent stages is important in understanding how they affect the pipeline as a whole and the observed latency of its results.

---------------------------------------------------------------------------------------------------------------------
***PIPELINE STAGES***

Different stages are typically necessary every time your pipeline groups data together by some new dimension. For example, if you had a pipeline that consumed raw data, computed some per-user aggregates, and then used those per-user aggregates to compute some per-team aggregates, you’d likely end up with a three-stage pipeline:

One consuming the raw, ungrouped data

One grouping the data by user and computing per-user aggregates

One grouping the data by team and computing per-team aggregates

-------------------------------------------------------------------------------------------------------------------------


Watermarks are created at input sources, as discussed in the preceding section. They then conceptually flow through the system as data progress through it.3 You can track watermarks at varying levels of granularity. For pipelines comprising multiple distinct stages, each stage likely tracks its own watermark, whose value is a function of all the inputs and stages that come before it. Therefore, stages that come later in the pipeline will have watermarks that are further in the past (because they’ve seen less of the overall input).



