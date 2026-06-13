Here are the complete requirements for Challenge 1 — no services, no technology, just what the system must do.

---

## Business context

A ride-sharing platform operates across three geographic regions — India, Southeast Asia, and the Middle East. The platform connects drivers and riders in real time. When demand for rides in an area exceeds the number of available drivers, the platform charges a higher fare multiplier — called surge pricing — to balance supply and demand. The system described here is responsible for computing and serving that multiplier.

---

## What the system must do

### Ingestion — collecting driver location data

Every driver using the platform has a mobile app that broadcasts their current GPS location continuously while they are active. The system must receive these location broadcasts and record them reliably.

- The system must accept location updates from all active drivers across all three regions simultaneously
- Each update contains: a driver identifier, a latitude coordinate, a longitude coordinate, a timestamp, and the driver's current status — available for a ride, currently on a trip, or offline
- At peak hours the platform has enough active drivers that the system receives up to **500,000 location updates per minute** across all regions combined
- Every update must be received and processed — losing location data means the system has an inaccurate picture of where drivers are, which leads to wrong pricing
- The system must acknowledge receipt to the driver app quickly so the app is not blocked waiting for a response

### Geographic aggregation — understanding supply per area

Raw GPS coordinates are not useful for pricing on their own. The system needs to know how many drivers are available within a meaningful geographic area — not at a precise point.

- The system must divide each city into geographic zones of approximately 1–2 square kilometres each
- For each zone, the system must maintain a continuously updated count of how many drivers are currently available within that zone
- A driver moving from one zone to an adjacent zone must be reflected in the counts of both zones promptly
- Driver counts must never be stale by more than **30 seconds** — if a surge situation develops, the system must detect it within 30 seconds of it forming
- The system must handle zones with very high driver density — a busy commercial area might have hundreds of drivers in a single zone at peak

### Demand tracking — understanding ride requests per area

The surge multiplier is based on the ratio of demand to supply. The system must also track how many ride requests are being made per zone.

- Every ride request submitted by a rider must be associated with the geographic zone of the pickup location
- The system must maintain a count of ride requests per zone over a rolling time window
- This count must be as current as the driver count — no more than 30 seconds stale

### Pricing computation — calculating the multiplier

Once the system knows the supply and demand per zone, it must compute a fare multiplier for that zone.

- The multiplier represents how many times the base fare a rider will be charged — for example 1.0x means normal fare, 2.5x means two and a half times the base fare
- The multiplier must never drop below **1.0x** and must be capped at **4.0x** regardless of how extreme the supply/demand imbalance is
- The multiplier for every active zone must be recomputed regularly — at most every **20–30 seconds** — so pricing stays current
- A zone with no recent driver or rider activity should return to 1.0x automatically — the system must not hold a stale high multiplier for a zone that has gone quiet
- The computation must account for both drivers currently available AND drivers currently on a trip nearby, weighted appropriately

### Serving pricing — delivering the multiplier to riders

When a rider opens the app and requests a fare estimate, the system must return the surge multiplier for their pickup location.

- The system must respond to a pricing query — given a latitude and longitude — within **80 milliseconds at the 99th percentile** globally
- This means a rider in Dubai, Mumbai, or Singapore must all get a response within 80ms
- The system serves millions of pricing queries per day — at peak, a single busy city can generate tens of thousands of pricing queries per minute
- The multiplier returned to a rider must never be more than **30 seconds out of date** — this is the staleness SLA
- If the exact zone has no recent data, the system must return 1.0x rather than an error

---

## Reliability requirements

### Availability

- The system must be available **99.99%** of the time — this translates to a maximum of approximately 52 minutes of downtime per year
- This availability target applies to both the ingestion side and the pricing read side independently — a failure in one must not bring down the other
- The platform has business-critical moments — major public holidays, monsoon season, late-night events — where the system must handle 3× normal peak load without degradation

### Data durability

- Every GPS location ping received must be stored permanently — it is used later for training the machine learning models that improve pricing accuracy and for resolving driver payment disputes
- Every pricing decision made — what multiplier was applied to which zone at what time — must be stored permanently for regulatory compliance and financial audit
- No GPS ping or pricing decision may be silently dropped — if the system cannot process a record it must preserve it for later reprocessing

### Fault tolerance

- If the component responsible for computing prices fails, the system must continue serving the last known price rather than returning errors to riders
- If the component responsible for receiving GPS pings fails, the pricing system must continue serving stale but reasonable prices — 1.0x is acceptable as a safe fallback
- No single component failure must cause a complete system outage
- Recovery from any single component failure must happen automatically without human intervention

---

## Operational requirements

### Multi-region

- The system must operate independently in three regions: India, Southeast Asia, and the Middle East
- Data for each region must be processed within that region — a GPS ping from a driver in Bangalore must not travel to Singapore for processing
- Despite regional independence, the architecture must be consistent across regions — not three bespoke designs

### Cost

- Total infrastructure cost across all three regions must remain under **$8,000 per month** at peak load
- The system must scale down during off-peak hours — 3 AM in each region has a fraction of the peak traffic and should cost a fraction of the peak price

### Observability

- The business must be able to see in real time: how many GPS pings are being received per minute, how many zones currently have surge pricing active, and what the current multiplier is in any given zone
- The engineering team must be alerted automatically if: GPS ping ingestion falls behind by more than 30 seconds, the pricing computation stops updating, or the read latency exceeds 80ms

### Security

- Driver location data is personally identifiable — it must be encrypted in transit and at rest
- The pricing read endpoint is public-facing — it must be protected against abuse, such as a single client making millions of requests to scrape pricing data

---

## What the system explicitly must NOT do

- It must not require a human to restart any component after a failure
- It must not serve a pricing error to a rider — 1.0x is always better than an HTTP 500
- It must not allow a pricing decision to be based on driver data older than 30 seconds
- It must not store driver location data in a way that allows one driver's data to be accessed when querying another driver's data
- It must not charge a rider more than 4.0x the base fare regardless of demand

---

These are the constraints. Everything else — how you store it, how you compute it, what triggers what — is an implementation decision you make to satisfy these requirements within the cost ceiling.
