# Hotel contexts

## Words

| Word | Meaning here | Name in our code |
|---|---|---|
| Stay | Check-in to check-out, half-open: 12–14 Nov = the nights of the 12th and 13th | DateRange (check_in, check_out) |
| Reservation | Guest + room + stay + status, with an id such as R101 | Reservation (id, guest, room, stay, status) |
| Confirmed | The room is promised to this guest for these nights | ReservationStatus.CONFIRMED |
| Guest | The person who sleeps in the room, with a name and an email: Maya, maya@example.com | Guest |
| Room | A room the hotel sells, with a number and a type: R7 Standard, R8 Deluxe | Room (number, type) |
| Room type | The kind of room, which decides its price: Standard or Deluxe | Room.type |
| Nightly price | What one night in a room costs, set by its type: Deluxe costs more than Standard | not in code yet |
| Out of service | The room is not on sale, for example R8 during repairs next week | not in code yet |
| Cancelled | The promise is withdrawn; the reservation is kept but no longer blocks the nights | ReservationStatus.CANCELLED |
| Book | Ask for a room for a stay and, if no Confirmed reservation shares a night, get a reservation id | BookingService.reserve(guest, room, stay) |
| Cancel | Withdraw a reservation by its id: Maya cancels R101 and R7 is free again for 12–14 Nov | BookingService.cancel(reservation_id) |
| Free rooms | Rooms that have no Confirmed reservation sharing a night with the stay: 12–14 Nov → R7, R8 | BookingService.find_free_rooms(stay) |
| Deposit | 30% of the stay's price, taken when the guest books | not in code yet |
| Paid | The deposit was taken successfully | not in code yet |
| Declined | The payment provider refused the deposit | not in code yet |
| Confirmation message | The email or SMS that tells the guest the booking is Accepted, with its id | NotificationSender.send_confirmation(reservation) |
| Sent | The confirmation message was delivered to the provider | not in code yet |
| Failed | The confirmation message could not be sent; the booking stays Accepted (R6) | not in code yet (today only BookingResult.notification_failed) |

## Rooms

Job: knows which rooms the hotel sells.

| Word | Meaning here |
|---|---|
| Room | A room the hotel sells, with a number and a type: R7 Standard, R8 Deluxe |
| Room type | The kind of room, which decides its price: Standard or Deluxe |
| Nightly price | What one night in a room costs, set by its type: Deluxe costs more than Standard |
| Out of service | The room is not on sale, for example R8 during repairs next week |

Rules this context owns:
- A room that is out of service is not offered to guests.
- The nightly price depends on the room type (Deluxe costs more than Standard).

Needs from other contexts: nothing.
Tells other contexts: nothing (it only answers Reservations' question "which rooms exist?").

## Reservations

Job: decides whether a guest gets a room for some nights, and keeps every reservation.

| Word | Meaning here |
|---|---|
| Stay | Check-in to check-out, half-open: 12–14 Nov = the nights of the 12th and 13th |
| Reservation | Guest + room number + stay + status, with an id such as R101 |
| Confirmed | The room is promised to this guest for these nights |
| Cancelled | The promise is withdrawn; the nights are free again |
| Guest | The person the reservation is for, with a name and an email |
| Book | Ask for a room for a stay; Accepted with an id, or Unavailable, or InvalidDates |
| Cancel | Withdraw a reservation by its id; it is kept as Cancelled |
| Free rooms | Rooms with no Confirmed reservation sharing a night with the stay |

Rules this context owns:
- A room never has two Confirmed reservations that share a night.
- Check-out must be after check-in.
- Cancelling a reservation frees its nights for new bookings.

Needs from other contexts: the list of rooms (Rooms); from P17, a deposit taken (Payments).
Tells other contexts: ReservationConfirmed (Notifications).

## Payments

Job: takes deposits and records whether they were paid.

| Word | Meaning here |
|---|---|
| Deposit | 30% of the stay's price, taken when the guest books |
| Paid | The deposit was taken successfully |
| Declined | The payment provider refused the deposit |

Rules this context owns:
- The deposit is 30% of the stay's price.
- Only a Paid deposit can be refunded.

Needs from other contexts: the reservation id and the amount to take, sent by Reservations with the "take a deposit" question (the price behind the amount comes from Rooms, rule card 6).
Tells other contexts: nothing; it answers Reservations' "take a deposit" question with Paid or Declined.

## Notifications

Job: tells guests what happened, by email or SMS.

| Word | Meaning here |
|---|---|
| Confirmation message | The email or SMS that tells the guest the booking is Accepted, with its id |
| Sent | The message was delivered to the provider |
| Failed | The message could not be sent; it is recorded, and the reservation does not change |

Rules this context owns:
- A failed message is recorded as Failed and never changes the reservation.

Needs from other contexts: the ReservationConfirmed event from Reservations (it never asks for anything).
Tells other contexts: nothing.

## Same word, different meaning

| Word | Rooms | Reservations | Payments | Notifications |
|---|---|---|---|---|
| Status | In service / out of service | Confirmed / Cancelled | Paid / Declined | Sent / Failed |
| Room | The full room: R8, Deluxe, its nightly price, in or out of service | Only the number R8 written on a reservation | Not used | Just text in the message: "Room R8" |
| Cancel | Take a room off sale | Withdraw a reservation; it stays as Cancelled and frees the nights | Refund a deposit | Not used |

If Notifications could change a reservation's status to Failed, then a failed SMS to Maya would overwrite her Confirmed R101, and Maya would lose room R7 just because a message did not arrive, which breaks rule R6 (a failed message must never undo a saved booking).

## Rule owners

| Card | Rule | Owner | Why this owner |
|---|---|---|---|
| 1 | A room never has two Confirmed reservations that share a night. | Reservations | It keeps every reservation of the room, so it can check them all together. |
| 2 | Check-out must be after check-in. | Reservations | The stay is its own word, and it must refuse wrong dates before it looks at anything else. |
| 3 | Cancelling a reservation frees its nights for new bookings. | Reservations | It keeps the reservation's status, and only Confirmed reservations block a room. |
| 4 | A room that is out of service is not offered to guests. | Rooms | It keeps which rooms are on sale and which are out of service. |
| 5 | The nightly price depends on the room type (Deluxe costs more than Standard). | Rooms | It keeps each room's type and its nightly price. |
| 6 | The deposit is 30% of the stay's price. | Payments | Money rules belong to Payments; it asks Rooms for the nightly price. |
| 7 | Only a Paid deposit can be refunded. | Payments | It keeps each deposit's status (Paid or Declined). |
| 8 | A failed message is recorded as Failed and never changes the reservation. | Notifications | It keeps each message's status; Reservations' status is not its data (rule R6). |

## Between contexts

### Reservations → Notifications: ReservationConfirmed (event)

Sent when: a reservation is saved as Confirmed.

Fields:
- reservationId: R101
- guestEmail: maya@example.com
- roomNumber: R7
- checkIn: 2026-11-12
- checkOut: 2026-11-14

Reservations does not wait for an answer. A failed message never changes R101 (rule R6).

Never crosses: Reservations' list of reservations. Notifications never reads it.

### Reservations → Rooms: which rooms exist? (question)

Sent when: a guest searches free rooms or books a room, so Reservations knows which rooms it can offer.

What Reservations sends: nothing but the question (later it may add the stay, so Rooms can leave out rooms that are out of service on those dates).

The answer and its fields:
- roomNumber: R7, R8
- roomType: Standard, Deluxe

Only rooms that are in service are returned (rule card 4).

Never crosses: Reservations' list of reservations and the guests. Rooms never decides whether a room is free on some nights; it only says which rooms exist.

### Reservations → Payments: take a deposit (question, from P17)

Sent when: Reservations has accepted a booking and needs the deposit taken (from P17).

What Reservations sends:
- reservationId: R101
- amount: 30% of the stay's price, for example 60.00 for a 200.00 stay

The answer and its fields:
- reservationId: R101
- status: Paid or Declined

Never crosses: card numbers and other payment details (they stay in Payments), and the list of reservations. Payments never decides whether a room is free.
