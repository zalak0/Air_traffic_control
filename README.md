# Air Traffic Control Tracker

A command-line air traffic control system written in C. It reads a live stream of aircraft broadcasts (ADS-B packets) from standard input and tracks every aircraft in a 350 km operating radius. It also answers controller queries about current positions, how many aircraft are in the airspace, and predicted conflicts between aircraft (separation issues).

Built as a first-year university programming project (USyd). It uses only the C standard library.

## Features

- **ADS-B packet parsing**: decodes broadcasts carrying each aircraft's ID, time, position, altitude, heading and speed, and keeps the latest state for each aircraft.
- **Dynamic aircraft tracking**: stores aircraft in a heap-allocated structure that grows as new aircraft appear. Aircraft are removed once their predicted path takes them out of the operating area, so memory use stays bounded even if the program runs for days.
- **Position estimation**: dead-reckons each aircraft's current position from its last broadcast, using constant heading and speed.
- **Separation checking**: predicts whether an aircraft will come within a given horizontal distance of any other aircraft flying within 750 m of its altitude. If it will, the program reports where the first conflict occurs.

## How it works

1. **Parsing**: each input line is either an ADS-B packet (starting with `#ADS-B:`) or a controller request (starting with `*time:`). Fields are pulled out of the comma-delimited string and stored in structs.
2. **Kinematics**: speed and heading are converted into north and east velocity components. A future position is then the last known position plus velocity multiplied by elapsed time.
3. **Airspace management**: before answering a request, the program projects each tracked aircraft to the request time. Any aircraft beyond 350 km of the station is dropped from the list.
4. **Conflict detection**: for each pair of aircraft at similar altitudes, the program works out their relative position and velocity at the check time. It then solves a circle–line intersection problem as a quadratic in time.
   - If the discriminant is negative, the paths never come too close.
   - Otherwise the two roots give the times the aircraft enter and leave the minimum separation distance. Looking at the signs of the roots shows whether the conflict is in the future, happening now, or already in the past.
   - Across all other aircraft, the earliest future conflict is reported, provided it falls inside the airspace.

## Build and run

```bash
make
./<executable_name> < input.txt
```

The program reads from standard input line by line until it receives a `close` request.

### Supported requests

| Request | Format | Response |
|---------|--------|----------|
| Estimate position | `*time:HH:MM,est_pos,<ID>` | Estimated N/E position in km, or a not-in-area message |
| Count contacts | `*time:HH:MM,num_contacts` | Number of aircraft currently in the airspace |
| Separation check | `*time:HH:MM,check_separation,<ID>,<min_km>` | Position of the first predicted conflict, or `No separation issues` |
| Close | `*time:HH:MM,close` | Prints `closing` and exits |

### Example

```
#ADS-B:6,time:09:23,N:39.7,E:23.1,alt:6500,head:-37.5,speed:270.0
#ADS-B:7285,time:09:24,N:187.0,E:187.4,alt:8000,head:-155.0,speed:210.0
*time:09:25,est_pos,7285
Aircraft (ID:7285): Estimated Position: N:181.7,E:176.0
*time:09:25,num_contacts
Currently tracking 2 aircraft
*time:09:26,close
closing
```

## Notes

- Predictions assume constant heading and speed between broadcasts. A real system would also account for turns, climbs and uncertainty in the measurements.
- All calculations use consistent SI units internally (metres and m/s). Positions are converted to km for display.
