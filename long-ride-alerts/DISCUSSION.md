<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# Lab Discussion: `ProcessFunction` and Timers (Long Ride Alerts)

(Discussion of [Lab: `ProcessFunction` and Timers (Long Ride Alerts)](./))

## How does this work?

The `main()` method for the job itself is straightforward: the stream of TaxiRides is keyed by rideId,
and processed by a `KeyedProcessFunction` called `MatchFunction`. The output of the `MatchFunction`
is printed out (or it's tested if the job is being run from the tests).

The business logic is all in the KeyedProcessFunction.

Looking first to see what keyed state is being used, it has a `ValueState<TaxiRide> rideState` --
in other words, it is remembering one `TaxiRide` object for each rideId. This is meant to be
an END event if the ride has ended, and otherwise it will be a START event. Looking into the
`processElement` method you can see that this is in fact what it does: if the incoming ride
is an END event it is definitely stored in the `rideState`. And if the incoming ride is a
START event it is only stored if the `rideState` is empty.

The `processElement` method always creates an event time timer set to fire 2 hours after the ride
started. 

When that timer fires, the ride in `rideState` is collected to the output
if it is a START event -- in other words, if an END event hasn't yet overwritten the state for this
rideId -- which is exactly what's required to generate an alert about a long ride. 

## What if something funny happens?

Most stream processing applications have to cope with real-world problems like missing, or
out-of-order data, and it's worth considering how this solution will behave in these cases:

* START event is missing: not a problem, the END will be saved and no alert will be emitted
* END event is missing: an alert will be emitted, as expected
* END arrives before the START: the START event will not overwrite the END already saved in `rideState`
* START or END event is duplicated: won't matter

You may have noticed that for each rideId, `registerEventTimeTimer` is called twice: once for
the START event, and again for the END event. Both are setting the timer for the same time: 2 hours
after the startTime of the ride, so these timers will be deduplicated (Flink only keeps one timer
per key and per timestamp). But even if we did somehow end up with multiple timers firing for
one rideId, the `onTimer` method would only emit one result and then delete the state for that
key. Subsequent calls to `onTimer` would then do nothing, except call `rideState.clear()` again,
which is harmless.

## Is this a good solution?

It's worth thinking about what makes for a good solution for a use case like this one.
Some possible criteria:

1. It should produce correct results
1. It should eventually clear any state it creates
1. It should be easy to understand and maintain

For correctness, there are tests.

As for point #2, can this job leak state? The only place that state is created is in
`processElement`, and whenever state is created, a timer is created for that same key.
And every time a timer fires, the state is cleared. In other words, every piece of
state is protected by a timer, and when the timer fires, the state is always cleared.

As for point #3, you can be the judge.

-----

[**Back to Labs Overview**](../LABS-OVERVIEW.md)
