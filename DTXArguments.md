# Arguments for Distributed Transactions

Currently, to achieve atomicity of transactions across microservices, synchronous distributed transaction (Some called it orchestration, typicaly in forms Saga, XA/2PC, AT, TCC, etc) are considered a bad idea and choreography (using CDC tools, like [debezium](https://debezium.io) to read logs or outbox pattern) should be used, as written [here](https://martin.kleppmann.com/2015/05/27/logs-for-data-infrastructure.html).

While this is mostly true, there are reasons why choreography is not always the best solution:

1. CDC, the implementation of choreography, also has caveats, as its prime feature is **ONLY** durability, as I argued [here](https://github.com/aarondwi/notes/blob/main/CDCCaveats/Caveats.md)
2. Choreography patterns assume static configuration for transactions, while in reality there may be optional service.
3. Choreography also assumes the caller doesn't need to know the result of the respondents.
4. It is hard to check the status of all transactions/propagations, especially with optional service used.

## Case

It is taking an example of an OTA, which has the most visible, well-known needs for distributed transactions.

Let's say this big OTA has 6 kinds of different verticals, all separated into separate service: `Flight`, `Accomodation`, `Train`, `Bus`, `Car renting`, `Attractions` (amusement park, soccer matches, etc).
Each of this vertical interfaces with both internal service (such as recommendation based on user id), and possibly many 3rd party partners (usually to share resource with other OTAs).
Beside that, it also has `typical` service, such as user service (for metadata), fraud detection service, reward service for frequest customer, payment service, and insurance service. Insurance itself usually are different for one vertical to the others.

### The 1st case

A couple of newlyweds wanna do their honeymoon by watching the final or world cup. They decided to use the big OTA's **trip planner** feature, as budget is not a problem. Their requirements:

1. To get ticket for the final match, possibly on lower seats so can watch closer.
2. They also want to have time for sightseeing before and after the match, so add additional 3 days to both, a total of 7 days.
3. They need flight tickets to go.
4. They gonna need car to sightseeing there.
5. They also need a hotel to stay. They prefer those with closer to the match, and has rating 4 or 5.
6. Additionally, they also want to go to few tourism spots, such as mountain, beach, etc.
7. For the flight, match, and hotel, they decided to add insurance. But not for car renting and tourism spots, they decided not to use insurance, as they can use uber there and go to another tourism spots in case something happens with their appointments.

### The 2nd case

An executive need to travel to give talks. His secretary decided to schedule all the needs with the same OTA as the 1st case. Their requirements:

1. The executive gonna need a flight ticket and back.
2. The executive need a hotel for 3 days and 2 night.
3. He also need a car rent, to go from airport to hotel and vice-versa.
4. Due to the company being a frequent customer, it can use specific discount for overall trip.

Note that even with all the custom search needs, the search can be separated from the actual transaction processing itself. All that matters now is that to fulfill the request, it needs to update data in multiple services. And whether the transaction is submitted to the first service, or to `order` service which accept the request and gonna propagate it, also does not matter, as the result would still be the same. The explanations below assumes using `order` service.

## Choreography Solution

From the 1st case, we would need to pass the message from flight service to the attractions services, twice. While in 2nd case, there is no need for such service. And because the typical car renting database design will only record consecutive date only (`from_date` and `to_date`), the 2nd case would need 2 separate transactions, while the 1st case only need one. There is also reward service for the 2nd case, while none for the 1st case.

If we were to use CDC, we would force it to be static, i.e. a transaction will be passed to all services, regardless of whether it is used. Each service then will check whether they need to do something, and if not, gonna pass straight **OK** response to the next one. There will also be a `sink` service that concludes whether the transaction success or not. Another option is that we gonna need to write to lots of queue atomically, one for each needed service, which basically back to orchestration.

Beside that, if we need to check the status of the transaction with its details, we gonna need to query **ALL** the services, and combine those included in the message. The more the number of services needed, the heavier the query becomes.

## Orchestration Solution

The order service accept the request, start the distributed transaction (usually by creating global unique id, and saving the request data) and start calling services needed only. No need to call unneeded services. To check the status of the transaction, we can simply query the distributed transaction database.

## Conclusion

If you need to know the result of all the respondent transactions, the respondents may be dynamic, and occasionally check the status of ongoing transaction + its metrics (how many currently running, where is blocking, etc), go for orchestration.

All that said, choreography still is the best solution for more static propagations and cases which correctness criteria only whether it is applied or not, such as cache filling/invalidation, pipeline to feed search engine, etc. It usually has higher throughput, as no contention by default and everything can be parallel, just read the log.
