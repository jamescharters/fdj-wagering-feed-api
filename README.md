# Wagering Stats API

Thank you for reviewing my code for the "Wagering Feed Consumer" task, written to satisfy the requirements as specified in the provided PDF. I've named the subsequent application the "Wagering Stats API" as, though it consumes both the FDJ wagering feed and Customer API, it presents a combined view to its own consumers. I suppose it's just a matter of naming semantics and personal taste!

Instructions to run the application as well as a high level development diary are found in this document. Please also refer to the source code for a variety of specific developer commentary/musings marked with `// DEVNOTE: ...`

## Prerequisites

This application was developed and tested on macOS. Ensure that the following are installed:

- Docker

## Code

```bash
git clone https://github.com/jamescharters/fdj-wagering-stats-api
cd fdj-wagering-stats-api
```

## Run (Docker)

```bash
# Build the image
docker build -t wagering-stats-api .

# Run the container (use real CANDIDATE_ID)
docker run -p 5000:8080 -e WageringFeed__CandidateId=CANDIDATE_ID wagering-stats-api
```

Or using Docker Compose:

```bash
# Set your candidate ID and run
CANDIDATE_ID=your-id-here docker compose up
```

The app will be available at `http://localhost:5000/customer/<customerId>/stats`.

For example, to obtain the information for `Barbara Rodriguez` while the wagering data stream is being consumed by the API, execute

```bash
curl -X GET http://localhost:5000/customer/114587/stats
```

This should return a payload of the expected form

```
{"customerId":114587,"name":"Barbara Rodriguez","totalStandToWin":2095.45}
```

## Design Notes

### Assumptions

- Implementation using traditional API structure with distinct controllers, repositories, services etc for testability and composability
- No hard requirement of using a design pattern such as CQRS: the goal here was not to completely overengineer the initial solution; though the code here would be amenable to such a pattern should it be desired
- "This will only work for the duration of your session" is assumed to mean the API HTTP GET endpoint `/customer/<customerId>/stats` only returns a response payload while the WebSocket session itself is active. In effect, this means the API only provides HTTP 200 range responses for valid customer IDs for about 3 minutes after start. Be quick!
- Further to the point above, the service however is not automatically terminated upon receipt of EndOfFeed message
- TotalStandToWin in the response has no currency conversion, and decimal values rounded to 2 places (i.e. the value corresponds to dollars-and-cents, Euro-and-cents etc)
- Automatic connection to web socket to receive messages is appropriate on application start, and once EndOfFeed is received (or our own hard timelimit expires), there is no automatic reconnection or even mechanism to re-establish the websocket subscription
- "Fixture" WebSocket messages don't seem to be needed for the customer stats endpoint, so they're ignored
- Repository `Clear()` only gets called at startup before the WebSocket connects, so no locking needed there
- A unified repository for Customer and associated data could be plausible, but these concerns are kept separate in this implementation

### Limitations

- In-memory data storage, therefore everything is lost on app restart
- Customer data, once obtained, is cached for application lifetime and therefore oblivious to changes at the source of truth (e.g. a name update for a customer)
- Only works as a single instance (multiple instances would each have their own separate data, leading to potentially inconsistent responses basded on which server answers the client query)
- API returns HTTP 503 once the feed completes (dis/gracefully), which might not be ideal depending on actual real world requirements
- No auth, rate limiting, or HTTP request/response caching (see TODOs/DEVNOTE remarks in the code). These, among other things, would require consideration for real-world usage
- No end to end / functional tests, nor performance (which might be important in this context to ensure highly performant API behaviour). Deemed out of scope for this proof of concept

### Potential Extensions

- Persistent storage such as Redis so data survives restarts and permit shared state for horizontal scaling (eg. Redis as backing store)
- More intelligent caching of customer data, i.e. awareness of expiration/changes to source of truth or sliding window expiration of local copy for example
- The handling of wagering data (WageringDataRepository) is based on a concurrent dictionary of the customer ID and the incrementally updated StandToWin value. The value of the dictionary would certainly be a more complex data structure (likely requiring slim locking logic) if anything above this trivial StandToWin data was required by the API. The repository is intentionally basic to meet the basic needs of the application as written to spec
- Internal health check endpoints for more robust container orchestration (and updates to Docker files to use these)
- Metrics/tracing (Azure AppInsights, Prometheus etc)
- Rate limiting middleware
- Auth (JWT, client API keys, or some other form)
- Secrets management for sensitive config
- Caching of requests/responses at the API edge
- Depending on customers and data, split based on tenant/region rather than a "kitchen sink" API for all customers (speculative!)
- ... others I am sure!