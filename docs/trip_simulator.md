# Trip activity simulator

Run against the single, unpartitioned development database:

```sh
bin/rails 'simulate:app_activity[10]'
```

The task reuses 10 simulator drivers, 30 riders, and four predefined locations.
It creates a request, assigns an idle driver, records pickup and interpolated
position samples, completes the ride, then assigns a random rating from 1 to 5.
The existing rating field has no separate rating-author column. A completed
driver returns to the idle pool. Requests, trips and position history accumulate;
the actor roster does not grow on repeated runs.

```sh
SIM_DRIVERS=2 SIM_RIDERS=5 SIM_SAMPLES=10 SIM_INTERVAL=0.1 SIM_SEED=42 \
  bin/rails 'simulate:app_activity[20]'
```

`SIM_INTERVAL` is seconds between position samples; zero runs without sleeping.
`SIM_SEED` controls participant, route and rating choices. Set `SIM_SQL_LOG=1`
to include SQL logs alongside lifecycle events. Routes are straight
lines between stored coordinates, without geocoding or a map service.

The service runs sequential rides through real Active Record writes and
`TripCreator`; it does not require an HTTP server, UI or background worker.
Idle means no trip with a null `completed_at`. Assignment locks the request and
an idle driver, preventing duplicate assignment and simultaneous use of a driver.
A busy simulator pool raises an error without creating another request. Existing
unrelated users and trips are never added to the simulator pool.

Trip creation is transactional; position updates and completion are separate
commits so the activity can be observed. An interrupted or failed ride remains
unfinished and its driver stays busy; the task does not pretend it completed.
The simulator is development-only. Tests exercise the lifecycle in the test DB.
