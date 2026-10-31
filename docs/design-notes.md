# P03 design notes

## Why design before code
Low-level design decides how the core program works at the level of each class: which classes exist, what data each one keeps, which methods each one has, and which class calls which. If the project is built in Go or Rust, which have no classes, the same decisions are made for structs and functions.

## Where the methods live
Cancel() lives in the Reservation class, because cancelling only changes the reservation's own status, and only the reservation knows its status.
HasOverlap() lives in the ReservationStore class, because only the store can see all the reservations of a room. It asks each reservation Blocks(), and each reservation uses DateRange.Overlaps() to compare the two pairs of dates.

## Activity traces
- Omar asks for R7 on 13–15 November, while Maya has 12–14. First diamond: is check-out after check-in? Yes. Second diamond: does a Confirmed booking share a night? Yes, the 13th. The request ends at "Unavailable · save nothing".
- Nora asks for R7 on 14–16 November, while Maya has 12–14. First diamond: is check-out after check-in? Yes. Second diamond: does a Confirmed booking share a night? No, Maya leaves on the 14th and Nora's first night is the 14th. The reservation is saved as Confirmed, the confirmation email is sent, and the request ends at "Accepted + reservation id" (R102).
- Omar asks for R8 on 15–15 November. First diamond: is check-out after check-in? No, both dates are the 15th, so there is no night at all. The request ends at "InvalidDates · save nothing"; the second diamond is never reached.

## Prediction
Nora books R7 14-16 Nov: Accepted R102. She shares no night with Maya (12–14), and Omar's request was refused, so the store holds only R101 and the next id is R102.
Result: the program printed "Nora books R7 14-16 Nov: Accepted R102", the same as the prediction.

## The race
Prediction: R7 will have 2 Confirmed reservations for 12–14 November, because both requests will check the store while it is still empty.

What happened: both Maya and Omar got "Accepted" (R101 and R102), and the store ended with 2 Confirmed reservations for R7 on the same nights, so rule R3 is broken. The problem appears between the two calls in Reserve: `store.has_overlap(room, stay)` (the check) and `store.save(reservation)` (the act), because the other request runs its check in the gap before the first one saves.

# P04 design notes

## Refactoring
In a refactoring, the way the code is organised changes (names, files, which class depends on which), but what the app does must not change at all.

Must stay the same:
- The tests from Step 2: all 6 still pass after every move.
- The demo output: the same nine lines as in P03.
- The race demo: still "Double booking!", because refactoring does not fix bugs.

## What our code breaks
- BookingService asks for exactly ReservationStore (D: dependency inversion).
  In P07 we would have to edit BookingService just to use a database.
- BookingService asks for exactly EmailSender (D and O: dependency inversion and open/closed).
  When the hotel wants SMS confirmations we would have to open BookingService and change its constructor and Reserve(), and in tests we cannot swap in a sender that fails, so we cannot prove that a failed email keeps the booking.

## Proof
- Tests: 6 passed after the refactoring (`python3 -m unittest -v` → Ran 6 tests, OK); 8 passed after Step 7.
- Demo: the same nine lines as in P03, line by line (`python3 main.py`).
- Race: still double-booked (`python3 main.py --race` → Maya: Accepted R101, Omar: Accepted R102, 2 Confirmed reservations, "Double booking!").
