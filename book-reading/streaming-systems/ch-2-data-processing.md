https://learning.oreilly.com/library/view/streaming-systems/9781491983867/ch02.html

concept of windowing (i.e., partitioning a dataset along temporal boundaries), which is a common approach used to cope with the fact that unbounded data sources technically might never end. Some simpler examples of windowing strategies are fixed and sliding windows, but more sophisticated types of windowing, such as sessions (in which the windows are defined by features of the data themselves; for example, capturing a session of activity per user followed by a gap of inactivity) also see broad usage.

#### Triggers
A trigger is a mechanism for declaring when the output for a window should be materialized relative to some external signal. Triggers provide flexibility in choosing when outputs should be emitted. In some sense, you can think of them as a flow control mechanism for dictating when results should be materialized. Another way of looking at it is that triggers are like the shutter-release on a camera, allowing you to declare when to take a snapshots in time of the results being computed.

Triggers also make it possible to observe the output for a window multiple times as it evolves. This in turn opens up the door to refining results over time, which allows for providing speculative results as data arrive, as well as dealing with changes in upstream data (revisions) over time or data that arrive late (e.g., mobile scenarios, in which someone’s phone records various actions and their event times while the person is offline and then proceeds to upload those events for processing upon regaining connectivity).


#### Watermarks
A watermark is a notion of input completeness with respect to event times. A watermark with value of time X makes the statement: “all input data with event times less than X have been observed.” As such, watermarks act as a metric of progress when observing an unbounded data source with no known end. 

#### Accumulation
An accumulation mode specifies the relationship between multiple results that are observed for the same window. Those results might be completely disjointed; that is, representing independent deltas over time, or there might be overlap between them. Different accumulation modes have different semantics and costs associated with them and thus find applicability across a variety of use cases.

-------------------------------------------------------------------------------------------------------------------

Also, because I think it makes it easier to understand the relationships between all of these concepts, we revisit the old and explore the new within the structure of answering four questions, all of which I propose are critical to ***every unbounded data processing problem:***


1) ***What results are calculated?*** This question is answered by the types of transformations within the pipeline. This includes things like computing sums, building histograms, training machine learning models, and so on. It’s also essentially the question answered by classic batch processing

2) ***Where in event time are results calculated?*** This question is answered by the use of event-time windowing within the pipeline. This includes the common examples of windowing from Chapter 1 (fixed, sliding, and sessions); use cases that seem to have no notion of windowing (e.g., time-agnostic processing; classic batch processing also generally falls into this category); and other, more complex types of windowing, such as time-limited auctions. Also note that it can include processing-time windowing, as well, if you assign ingress times as event times for records as they arrive at the system.

3) ***When in processing time are results materialized?*** This question is answered by the use of triggers and (optionally) watermarks. There are infinite variations on this theme, but the most common patterns are those involving repeated updates (i.e., materialized view semantics), those that utilize a watermark to provide a single output per window only after the corresponding input is believed to be complete (i.e., classic batch processing semantics applied on a per-window basis), or some combination of the two.

4) ***How do refinements of results relate?*** This question is answered by the type of accumulation used: discarding (in which results are all independent and distinct), accumulating (in which later results build upon prior ones), or accumulating and retracting (in which both the accumulating value plus a retraction for the previously triggered value(s) are emitted).


----------------------------------------------------------------------------------------------------------------------

### Batch Foundations: What and Where

### What: Transformations
The transformations applied in classic batch processing answer the question: “What results are calculated?” Even though you are likely already familiar with classic batch processing, we’re going to start there anyway because it’s the foundation on top of which we add all of the other concepts.

https://learning.oreilly.com/library/view/streaming-systems/9781491983867/assets/stsy_0203.mp4


### Where: Windowing
As discussed in Chapter 1, windowing is the process of slicing up a data source along temporal boundaries. Common windowing strategies include fixed windows, sliding windows, and sessions windows

https://learning.oreilly.com/library/view/streaming-systems/9781491983867/assets/stsy_0205.mp4

--------------------------------------------------------------------------------------------------------------------

### Going Streaming: When and How
We just observed the execution of a windowed pipeline on a batch engine. But, ideally, we’d like to have lower latency for our results, and we’d also like to natively handle unbounded data sources. Switching to a streaming engine is a step in the right direction, but our previous strategy of waiting until our input has been consumed in its entirety to generate output is no longer feasible. Enter triggers and watermarks.


### When: The Wonderful Thing About Triggers Is Triggers Are Wonderful Things!
Triggers provide the answer to the question: “When in processing time are results materialized?” Triggers declare when output for a window should happen in processing time (though the triggers themselves might make those decisions based on things that happen in other time domains, such as watermarks progressing in the event-time domain, as we’ll see in a few moments). Each specific output for a window is referred to as a pane of the window.

Though it’s possible to imagine quite a breadth of possible triggering semantics,3 conceptually there are only two generally useful types of triggers, and practical applications almost always boil down using either one or a combination of both:

1) ***Repeated update triggers
These periodically generate updated panes for a window as its contents evolve. These updates can be materialized with every new record, or they can happen after some processing-time delay, such as once a minute. The choice of period for a repeated update trigger is primarily an exercise in balancing latency and cost.

2) ***Completeness triggers
These materialize a pane for a window only after the input for that window is believed to be complete to some threshold. This type of trigger is most analogous to what we’re familiar with in batch processing: only after the input is complete do we provide a result. The difference in the trigger-based approach is that the notion of completeness is scoped to the context of a single window, rather than always being bound to the completeness of the entire input.



Repeated update triggers are the most common type of trigger encountered in streaming systems. They are simple to implement and simple to understand, and they provide useful semantics for a specific type of use case: repeated (and eventually consistent) updates to a materialized dataset, analogous to the semantics you get with materialized views in the database world.

Completeness triggers are less frequently encountered, but provide streaming semantics that more closely align with those from the classic batch processing world. They also provide tools for reasoning about things like missing data and late data, both of which we discuss shortly (and in the next chapter) as we explore the underlying primitive that drives completeness triggers: watermarks.

https://learning.oreilly.com/library/view/streaming-systems/9781491983867/assets/stsy_0206.mp4

You can see how we now get multiple outputs (panes) for each window: once per corresponding input. This sort of triggering pattern works well when the output stream is being written to some sort of table that you can simply poll for results. Any time you look in the table, you’ll see the most up-to-date value for a given window, and those values will converge toward correctness over time.

One downside of per-record triggering is that it’s quite chatty. When processing large-scale data, aggregations like summation provide a nice opportunity to reduce the cardinality of the stream without losing information. This is particularly noticeable for cases in which you have high-volume keys; for our example, massive teams with lots of active players. Imagine a massively multiplayer game in which players are split into one of two factions, and you want to tally stats on a per-faction basis. It’s probably unnecessary to update your tallies with every new input record for every player in a given faction. Instead, you might be happy updating them after some processing-time delay, say every second, or every minute. The nice side effect of using processing-time delays is that it has an equalizing effect across high-volume keys or windows: the resulting stream ends up being more uniform cardinality-wise.

