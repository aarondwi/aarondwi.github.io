# Distributed System's Semantic For Monoliths

Most projects written as monolith right now are close to distributed system, even when more capacity is not needed, if not already is.
This is because even when business logics can all reside in a single database, few functionalities need to go outside, mainly to 3rd parties, such as:

1. Payments, usually via gateway, not directly to banks.
2. Emails, notifications, etc.
3. Or probably for loads, such as caching, read from replica, etc.

Because of that, distributed systems' semantics need to be considered, such as:

1. At-least/at-most once delivery. Is it better to send notifs/emails twice, or better losing it? What about payment calls to vendor?
2. Bulkhead pattern to prevent overloading DB/upstream services. This mostly already built-in to DB connection pool, HTTP libraries, etc.
3. Rate Limiting, even on single node is better than nothing. Or when 3rd party dependencies give RPS limit.
