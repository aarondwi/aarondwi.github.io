# Regarding Allocations on Web Apps

Typically, these are allocations happening on web app:

1. Thread/coroutine/goroutine/stack-like for each HTTP request
2. Alloc for network buffer
3. Web framework per request context
4. Business object from network
5. Creating DB queries from business objects
6. Waiting, receiving db results, and SerDe to business objects
7. Business objects to HTTP response
8. Intermediate state when SerDe-ing network/db buffer to business object/framework object

Basically, most web apps spent too much time doing allocs. CPU time spent allocating means time not used to do business logic. For managed language, more allocations also means more time needed when GC kicks in, causing another stalls

To solve it:

1. Reuse buffer/object (pooling) Because mostly have same shapes, and relatively close size (because mostly same type of request). Can use libs like fasthttp, etc. For typical static response (error object, OK without additional data, etc) initiate once, reuse
2. Use non-serializing lib (like jsonparser/fastjson), and reuse business objects
3. Need to start with storage (DB, queue, etc) clients that does less allocs. If possible, also native zero allocs API (flatbuffers/capnproto/custom)
