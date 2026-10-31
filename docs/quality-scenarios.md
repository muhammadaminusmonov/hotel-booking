# Quality scenarios

## Stakeholders

| Who | What they care about |
|---|---|
| Guest | Finds a free room quickly; never loses a booking |
| Receptionist | Never promises one room twice; can cancel at the desk |
| Hotel manager (domain expert) | Knows the rules; wants deposits and SMS soon |
| Development team | Can change one area without breaking the others |
| Email/SMS and payment providers | Outside systems that can be slow or down |

## S3 · Availability: a failed message keeps the booking

| Part | S3 |
|---|---|
| Source | The email provider |
| Stimulus | It stops answering |
| Environment | For 10 minutes, during normal bookings |
| Artifact | BookingService.Reserve and the notification sender |
| Response | Every valid booking is still Accepted and saved; the failed message is reported |
| Measure | 0 lost bookings; every failed message is counted |
| Checked in | P04 (R6 test, done) and P10 |

## S1 · Performance: many guests search at once

| Part | S1 |
|---|---|
| Source | 200 guests |
| Stimulus | They search free rooms for 12–14 Nov |
| Environment | Within one minute, during normal operation |
| Artifact | The free-room search in BookingService (`find_free_rooms`) |
| Response | Every guest gets the correct list of free rooms |
| Measure | p95 response ≤ 1 second (95 of every 100 searches answer in 1 second or less) |
| Checked in | P08 (cache) and P23 (load test) |

## S2 · Data integrity: two guests, one room, the same moment

| Part | S2 |
|---|---|
| Source | 100 pairs of guests |
| Stimulus | Both guests in each pair book the same room for the same nights |
| Environment | At the same moment (the race from P03), during normal operation |
| Artifact | BookingService.Reserve and the reservation store |
| Response | Exactly one guest of each pair is Accepted; the other gets Unavailable |
| Measure | Exactly 1 Confirmed reservation per pair; 0 double bookings |
| Checked in | P09 |

## S4 · Security: reading another guest's reservation

| Part | S4 |
|---|---|
| Source | A guest |
| Stimulus | Tries to read another guest's reservation, 100 times with different ids |
| Environment | Signed in as an ordinary guest, during normal operation |
| Artifact | The "view a booking" lookup (R5) |
| Response | Every attempt is refused; the guest only ever sees their own reservations |
| Measure | 0 reservations shown to the wrong guest out of 100 attempts |
| Checked in | P21 |

## S5 · Modifiability: adding SMS confirmations

| Part | S5 |
|---|---|
| Source | The hotel manager |
| Stimulus | Asks for SMS confirmations as well as email |
| Environment | During development, while the app is in use |
| Artifact | The notification sender and BookingService |
| Response | SMS is added as a new sender class; BookingService is not edited |
| Measure | 0 changes to BookingService; at most 2 files change |
| Checked in | P04 (SMS sender, done) |

## S6 · Sustainability: energy per search

| Part | S6 |
|---|---|
| Source | The development team |
| Stimulus | Changes the free-room search (for example, adds a cache) |
| Environment | The same 1,000 searches are run before and after the change |
| Artifact | The free-room search and the machine it runs on |
| Response | The same correct answers, using less energy |
| Measure | At least 20% fewer watt-hours per 1,000 completed searches |
| Checked in | P08 and P24 |
