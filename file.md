

"we need to implement procedures instead of using table names directly in PostgreSQL"

This means:

No direct Train.objects.filter(...)

Instead call PostgreSQL stored procedures like:

SELECT * FROM get_available_trains(origin, destination, travel_date);
 2. How to write a PostgreSQL stored PROCEDURE / FUNCTION
Use FUNCTION (returns data) not PROCEDURE (does not return rows).

Example: Returning trains list:

📌 PostgreSQL Function Example
CREATE OR REPLACE FUNCTION get_available_trains(
    p_origin TEXT,
    p_destination TEXT,
    p_date DATE
)
RETURNS TABLE (
    id INT,
    train_number TEXT,
    train_name TEXT,
    origin TEXT,
    destination TEXT,
    departure_time TIME,
    arrival_time TIME
)
AS $$
BEGIN
    RETURN QUERY
    SELECT 
        t.id,
        t.train_number,
        t.train_name,
        s1.name AS origin,
        s2.name AS destination,
        t.departure_time,
        t.arrival_time
    FROM trains_train t
    JOIN trains_station s1 ON t.origin_id = s1.id
    JOIN trains_station s2 ON t.destination_id = s2.id
    WHERE s1.name ILIKE p_origin
      AND s2.name ILIKE p_destination;
END;
$$ LANGUAGE plpgsql;
 3. Calling PostgreSQL Stored Function in Django
Django can run SQL functions via:

✔️ Option A – connection.cursor()
from django.db import connection

def call_get_available_trains(origin, destination, travel_date):
    with connection.cursor() as cursor:
        cursor.execute("""
            SELECT * FROM get_available_trains(%s, %s, %s);
        """, [origin, destination, travel_date])
        rows = cursor.fetchall()
    return rows
 4. Convert your API to stored procedure usage
🔵 Example 1: Train Search API (Stored Procedure version)
Step 1: Create a PostgreSQL function
CREATE OR REPLACE FUNCTION search_trains(
    p_origin TEXT,
    p_destination TEXT,
    p_date DATE
)
RETURNS TABLE (
    train_id INT,
    train_number TEXT,
    train_name TEXT,
    origin TEXT,
    destination TEXT,
    departure_time TIME,
    arrival_time TIME
)
AS $$
BEGIN
    RETURN QUERY
    SELECT 
        t.id,
        t.train_number,
        t.train_name,
        s1.name AS origin,
        s2.name AS destination,
        t.departure_time,
        t.arrival_time
    FROM trains_train t
    JOIN trains_station s1 ON t.origin_id = s1.id
    JOIN trains_station s2 ON t.destination_id = s2.id
    WHERE s1.name ILIKE p_origin
      AND s2.name ILIKE p_destination;
END;
$$ LANGUAGE plpgsql;
Step 2: Django API calling stored procedure
from django.db import connection
from rest_framework.views import APIView
from rest_framework.response import Response

class TrainSearchAPIView(APIView):
    def get(self, request):
        origin = request.query_params.get('origin')
        destination = request.query_params.get('destination')
        travel_date = request.query_params.get('date')

        with connection.cursor() as cursor:
            cursor.execute(
                "SELECT * FROM search_trains(%s, %s, %s);",
                [origin, destination, travel_date]
            )
            rows = cursor.fetchall()

        result = []
        for r in rows:
            result.append({
                "id": r[0],
                "train_number": r[1],
                "train_name": r[2],
                "origin": r[3],
                "destination": r[4],
                "departure_time": str(r[5]),
                "arrival_time": str(r[6]),
            })

        return Response(result)
🔵 Example 2: Available Origins API using stored procedure
PostgreSQL Function
CREATE OR REPLACE FUNCTION get_available_origins(p_destination TEXT)
RETURNS TABLE(origin TEXT)
AS $$
BEGIN
    RETURN QUERY
    SELECT DISTINCT s1.name
    FROM trains_train t
    JOIN trains_station s1 ON t.origin_id = s1.id
    JOIN trains_station s2 ON t.destination_id = s2.id
    WHERE s2.name ILIKE p_destination;
END;
$$ LANGUAGE plpgsql;
Django View
class AvailableOriginsView(APIView):
    def get(self, request):
        destination = request.query_params.get("destination")

        with connection.cursor() as cursor:
            cursor.execute("SELECT * FROM get_available_origins(%s)", [destination])
            rows = cursor.fetchall()

        origins = [r[0] for r in rows]
        return Response(origins)
🔵 Example 3: Booking Create API using stored procedure
PostgreSQL Function
CREATE OR REPLACE FUNCTION create_booking(
    p_user_id INT,
    p_train_id INT,
    p_class TEXT,
    p_seats INT,
    p_total NUMERIC
)
RETURNS INT
AS $$
DECLARE new_id INT;
BEGIN
    INSERT INTO bookings_booking(
        user_id, train_id, travel_class, number_of_seats, total_fare, status, booking_date
    ) VALUES (
        p_user_id, p_train_id, p_class, p_seats, p_total, 'Booked', NOW()
    )
    RETURNING id INTO new_id;

    RETURN new_id;
END;
$$ LANGUAGE plpgsql;
Django API
class BookingCreateView(APIView):
    def post(self, request):
        user = request.user
        train_id = request.data['train']
        cls = request.data['travel_class']
        seats = request.data['number_of_seats']
        total = request.data['total_fare']

        with connection.cursor() as cursor:
            cursor.execute("""
                SELECT create_booking(%s, %s, %s, %s, %s);
            """, [user.id, train_id, cls, seats, total])
            booking_id = cursor.fetchone()[0]

        return Response({"booking_id": booking_id})
🎯 Final Notes
✔️ You can convert ALL your APIs like this
✔️ Django ORM will no longer touch tables directly
✔️ PostgreSQL procedures manage business logic
✔️ Safer, faster, and easier to maintain

 If you want, I can generate:
✔️ Stored procedures for every API
✔️ Full SQL + Django integration
✔️ Step-by-step migration plan
✔️ Optimization + indexing for speed
Would you like complete stored procedure conversion for the whole project?

You said:
yes we need to do that
 said:
Great — I’ll convert the project to use PostgreSQL functions for the core operations and show you the SQL and the Django-side calls. I’ll include concurrency-safe booking logic (seat checks + row locking), search/list APIs, CRUD for trains/stations, bulk station upload, booking cancel/history, and helper functions for seats/fare. I can’t run these here, so treat them as ready-to-use templates you should test in a dev DB before production.

Plan (what I’ll deliver)
Stored functions (PL/pgSQL):

fn_search_trains(...) — search trains by origin/destination/date and return seats/fare.

fn_get_available_origins(destination)

fn_get_available_destinations(origin)

fn_get_train_by_id(train_id)

fn_get_available_seats(train_id) — returns seats per class

fn_reserve_booking(...) — transactional create booking with seat validation & locking, returns booking id or raises

fn_cancel_booking(booking_id, user_id) — cancels if allowed and returns success/failure

fn_get_booking_history(user_id) — returns bookings for user

fn_create_train(...), fn_update_train(...), fn_delete_train(...)

fn_bulk_create_stations(text[]) — bulk create stations

Django wrappers calling the functions safely (connection.cursor()), using transaction.atomic() where needed.

Notes on locking, indexes, testing, and migration steps.

Conventions / Assumptions
Tables:

trains_train (id, train_number, train_name, origin_id, destination_id, departure_time, arrival_time, travel_duration, fare_per_class JSONB, available_seats JSONB, ...)

trains_station (id, name)

bookings_booking (id, user_id, train_id, travel_class, number_of_seats, total_fare, status, booking_date, travel_date, ...)

fare_per_class and available_seats are stored as jsonb in trains_train (e.g. {"SL": 100, "3A": 40}).

Use ILike for case-insensitive partial matching.

You should create DB indexes on trains_train(origin_id), trains_train(destination_id), trains_station(name), bookings_booking(train_id, travel_date, status) for performance.

1) Helper: available seats function
CREATE OR REPLACE FUNCTION fn_get_available_seats(p_train_id INT)
RETURNS TABLE (class TEXT, seats INT)
LANGUAGE plpgsql AS $$
DECLARE seats_json jsonb;
BEGIN
    SELECT available_seats INTO seats_json
    FROM trains_train WHERE id = p_train_id;

    IF seats_json IS NULL THEN
        RETURN;
    END IF;

    RETURN QUERY
    SELECT key, (value::text)::INT
    FROM jsonb_each_text(seats_json);
END;
$$;
2) Search trains (with seats adjusted for existing bookings on a travel_date)
CREATE OR REPLACE FUNCTION fn_search_trains(
    p_origin TEXT,
    p_destination TEXT,
    p_travel_date DATE
)
RETURNS TABLE (
    train_id INT,
    train_number TEXT,
    train_name TEXT,
    origin TEXT,
    destination TEXT,
    departure_time TIME,
    arrival_time TIME,
    travel_duration TEXT,
    available_classes TEXT[],
    available_seats JSONB,
    fare_per_class JSONB
)
LANGUAGE plpgsql AS $$
DECLARE seats_json jsonb;
    cls TEXT;
    remaining jsonb := '{}';
    booked_total INT;
    k TEXT;
    v TEXT;
BEGIN
    FOR train_id, train_number, train_name, origin, destination, departure_time, arrival_time, travel_duration, seats_json, fare_per_class
    IN
        SELECT t.id, t.train_number, t.train_name, s1.name, s2.name, t.departure_time, t.arrival_time, t.travel_duration::text, t.available_seats, t.fare_per_class
        FROM trains_train t
        JOIN trains_station s1 ON t.origin_id = s1.id
        JOIN trains_station s2 ON t.destination_id = s2.id
        WHERE (s1.name ILIKE p_origin OR t.id IN (
            SELECT ts.train_id FROM trains_stop ts JOIN trains_station st ON ts.station_id = st.id WHERE st.name ILIKE p_origin
        ))
        AND (s2.name ILIKE p_destination OR t.id IN (
            SELECT ts.train_id FROM trains_stop ts JOIN trains_station st ON ts.station_id = st.id WHERE st.name ILIKE p_destination
        ))
    LOOP
        -- copy seats_json into mutable 'remaining'
        remaining := '{}'::jsonb;
        IF seats_json IS NOT NULL THEN
            remaining := seats_json;
            -- subtract booked seats on this travel date
            FOR k, v IN SELECT key, value FROM jsonb_each_text(seats_json)
            LOOP
                SELECT COALESCE(SUM(number_of_seats), 0)
                INTO booked_total
                FROM bookings_booking
                WHERE train_id = train_id
                  AND travel_date = p_travel_date
                  AND travel_class = k
                  AND status = 'Booked';

                remaining := jsonb_set(remaining, ARRAY[k], to_jsonb(GREATEST((v::int - booked_total), 0)));
            END LOOP;
        END IF;

        RETURN NEXT (
            train_id,
            train_number,
            train_name,
            origin,
            destination,
            departure_time,
            arrival_time,
            travel_duration,
            ARRAY(
                SELECT key FROM jsonb_each_text(remaining) WHERE (value::int) > 0
            ),
            remaining,
            fare_per_class
        );
    END LOOP;
END;
$$;
Note: This function returns available_classes + available_seats (a JSONB object of remaining seats). Adjust types/columns if your model fields differ.

3) Origins / Destinations functions
CREATE OR REPLACE FUNCTION fn_get_available_origins(p_destination TEXT)
RETURNS TABLE(origin TEXT)
LANGUAGE sql AS $$
    SELECT DISTINCT s1.name
    FROM trains_train t
    JOIN trains_station s1 ON t.origin_id = s1.id
    JOIN trains_station s2 ON t.destination_id = s2.id
    WHERE s2.name ILIKE p_destination;
$$;

CREATE OR REPLACE FUNCTION fn_get_available_destinations(p_origin TEXT)
RETURNS TABLE(destination TEXT)
LANGUAGE sql AS $$
    SELECT DISTINCT s2.name
    FROM trains_train t
    JOIN trains_station s1 ON t.origin_id = s1.id
    JOIN trains_station s2 ON t.destination_id = s2.id
    WHERE s1.name ILIKE p_origin;
$$;
4) Booking reservation (transactional + row lock)
This is the most critical piece. It handles:

Locking the train row (or another seat allocation row) using SELECT ... FOR UPDATE.

Re-checking available seats (taking booked seats into account).

Decrementing seats in trains_train.available_seats (optional — sometimes you keep static capacity and rely on bookings table; but below we adjust JSONB directly).

Inserting booking in bookings_booking.

Returns booking id.

CREATE OR REPLACE FUNCTION fn_reserve_booking(
    p_user_id INT,
    p_train_id INT,
    p_travel_date DATE,
    p_travel_class TEXT,
    p_seats INT
)
RETURNS INT
LANGUAGE plpgsql AS $$
DECLARE
    seats_json jsonb;
    current_seats INT;
    booked_sum INT;
    new_seats INT;
    booking_id INT;
    fare_per_class jsonb;
    fare_per_unit NUMERIC;
    total_fare NUMERIC;
BEGIN
    IF p_seats <= 0 THEN
        RAISE EXCEPTION 'Invalid seat count';
    END IF;

    -- Lock the train row to avoid race conditions
    SELECT available_seats, fare_per_class
    INTO seats_json, fare_per_class
    FROM trains_train
    WHERE id = p_train_id
    FOR UPDATE;

    IF seats_json IS NULL THEN
        RAISE EXCEPTION 'No seat data for train %', p_train_id;
    END IF;

    -- current_seats from JSON
    SELECT (seats_json ->> p_travel_class)::INT INTO current_seats;
    IF current_seats IS NULL THEN
        RAISE EXCEPTION 'Class % not available', p_travel_class;
    END IF;

    -- seats already booked on that date (committed bookings)
    SELECT COALESCE(SUM(number_of_seats),0)
    INTO booked_sum
    FROM bookings_booking
    WHERE train_id = p_train_id
      AND travel_date = p_travel_date
      AND travel_class = p_travel_class
      AND status = 'Booked';

    new_seats := current_seats - booked_sum;
    IF new_seats < p_seats THEN
        RAISE EXCEPTION 'Only % seats available in class %', new_seats, p_travel_class;
    END IF;

    -- fare
    fare_per_unit := (fare_per_class ->> p_travel_class)::NUMERIC;
    IF fare_per_unit IS NULL THEN
        fare_per_unit := 0;
    END IF;
    total_fare := fare_per_unit * p_seats;

    -- Optionally: reduce available_seats base value (so next calls see reduced capacity)
    -- NOTE: If you prefer to keep original capacity and rely solely on bookings table, skip this update.
    seats_json := jsonb_set(seats_json, ARRAY[p_travel_class], to_jsonb((current_seats - p_seats)));
    UPDATE trains_train SET available_seats = seats_json WHERE id = p_train_id;

    -- Create booking record
    INSERT INTO bookings_booking (user_id, train_id, travel_date, travel_class, number_of_seats, total_fare, status, booking_date)
    VALUES (p_user_id, p_train_id, p_travel_date, p_travel_class, p_seats, total_fare, 'Booked', now())
    RETURNING id INTO booking_id;

    RETURN booking_id;
EXCEPTION WHEN others THEN
    RAISE;
END;
$$;
Important: The approach above decreases the available_seats base by p_seats. If you instead want to keep available_seats static and compute remaining seats by subtracting bookings, remove the update to trains_train. Both approaches are valid — choose one.

5) Cancel booking function
CREATE OR REPLACE FUNCTION fn_cancel_booking(p_booking_id INT, p_user_id INT)
RETURNS BOOLEAN
LANGUAGE plpgsql AS $$
DECLARE b_status TEXT;
    t_id INT;
    t_class TEXT;
    t_seats INT;
    seats_json jsonb;
    current_in_json INT;
BEGIN
    SELECT status, train_id, travel_class, number_of_seats
    INTO b_status, t_id, t_class, t_seats
    FROM bookings_booking
    WHERE id = p_booking_id AND user_id = p_user_id
    FOR UPDATE;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'Booking not found';
    END IF;

    IF b_status = 'Cancelled' THEN
        RETURN FALSE;
    END IF;

    -- mark cancelled
    UPDATE bookings_booking SET status = 'Cancelled' WHERE id = p_booking_id;

    -- If you modified trains_train.available_seats during booking, restore them
    SELECT available_seats INTO seats_json FROM trains_train WHERE id = t_id FOR UPDATE;
    IF seats_json IS NOT NULL THEN
        SELECT (seats_json ->> t_class)::INT INTO current_in_json;
        IF current_in_json IS NULL THEN
            current_in_json := 0;
        END IF;
        seats_json := jsonb_set(seats_json, ARRAY[t_class], to_jsonb(current_in_json + t_seats));
        UPDATE trains_train SET available_seats = seats_json WHERE id = t_id;
    END IF;

    RETURN TRUE;
END;
$$;
6) Booking history
CREATE OR REPLACE FUNCTION fn_get_booking_history(p_user_id INT)
RETURNS TABLE (
    id INT,
    train_id INT,
    travel_date DATE,
    travel_class TEXT,
    number_of_seats INT,
    total_fare NUMERIC,
    status TEXT,
    booking_date TIMESTAMP
)
LANGUAGE sql AS $$
    SELECT id, train_id, travel_date, travel_class, number_of_seats, total_fare, status, booking_date
    FROM bookings_booking
    WHERE user_id = p_user_id
    ORDER BY booking_date DESC;
$$;
7) Train CRUD (example: create train) — you would adapt to your fields
CREATE OR REPLACE FUNCTION fn_create_train(
    p_train_number TEXT,
    p_train_name TEXT,
    p_origin TEXT,
    p_destination TEXT,
    p_departure_time TIME,
    p_arrival_time TIME,
    p_travel_duration INTERVAL,
    p_available_seats JSONB,
    p_fare_per_class JSONB
)
RETURNS INT
LANGUAGE plpgsql AS $$
DECLARE origin_id INT;
    dest_id INT;
    new_id INT;
BEGIN
    INSERT INTO trains_station (name)
    SELECT p_origin
    WHERE NOT EXISTS (SELECT 1 FROM trains_station WHERE name = p_origin);

    INSERT INTO trains_station (name)
    SELECT p_destination
    WHERE NOT EXISTS (SELECT 1 FROM trains_station WHERE name = p_destination);

    SELECT id INTO origin_id FROM trains_station WHERE name = p_origin LIMIT 1;
    SELECT id INTO dest_id   FROM trains_station WHERE name = p_destination LIMIT 1;

    INSERT INTO trains_train (train_number, train_name, origin_id, destination_id, departure_time, arrival_time, travel_duration, available_seats, fare_per_class)
    VALUES (p_train_number, p_train_name, origin_id, dest_id, p_departure_time, p_arrival_time, p_travel_duration, p_available_seats, p_fare_per_class)
    RETURNING id INTO new_id;

    RETURN new_id;
END;
$$;
Likewise create fn_update_train(...) and fn_delete_train(train_id) (DELETE FROM trains_train WHERE id = ...).

8) Bulk create stations
CREATE OR REPLACE FUNCTION fn_bulk_create_stations(p_names TEXT[])
RETURNS TABLE(created_count INT, skipped_count INT)
LANGUAGE plpgsql AS $$
DECLARE s TEXT;
    created INT := 0;
    skipped INT := 0;
BEGIN
    FOREACH s IN ARRAY p_names
    LOOP
        s := btrim(s);
        IF s = '' THEN CONTINUE; END IF;
        IF NOT EXISTS (SELECT 1 FROM trains_station WHERE name = s) THEN
            INSERT INTO trains_station (name) VALUES (s);
            created := created + 1;
        ELSE
            skipped := skipped + 1;
        END IF;
    END LOOP;
    RETURN QUERY SELECT created, skipped;
END;
$$;
Django side — calling functions & example views
General pattern:

from django.db import connection, transaction
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
Example: TrainSearchAPIView using fn_search_trains
class TrainSearchAPIView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request):
        origin = request.query_params.get('origin', '')
        destination = request.query_params.get('destination', '')
        travel_date = request.query_params.get('date')  # YYYY-MM-DD

        with connection.cursor() as cursor:
            cursor.execute("SELECT * FROM fn_search_trains(%s, %s, %s);",
                           [origin, destination, travel_date])
            rows = cursor.fetchall()

        result = []
        for r in rows:
            result.append({
                "id": r[0],
                "train_number": r[1],
                "train_name": r[2],
                "origin": r[3],
                "destination": r[4],
                "departure_time": r[5].strftime("%H:%M") if r[5] else None,
                "arrival_time": r[6].strftime("%H:%M") if r[6] else None,
                "travel_duration": r[7],
                "available_classes": r[8],
                "available_seats": r[9],
                "fare_per_class": r[10],
            })
        return Response(result, status=status.HTTP_200_OK)
Example: AvailableOriginsView
class AvailableOriginsView(APIView):
    permission_classes = [IsAuthenticated]
    def get(self, request):
        destination = request.query_params.get('destination', '').strip()
        with connection.cursor() as cursor:
            cursor.execute("SELECT * FROM fn_get_available_origins(%s);", [destination])
            rows = cursor.fetchall()
        return Response([r[0] for r in rows])
Example: BookingCreateView calling fn_reserve_booking safely in transaction
class BookingCreateView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request):
        user = request.user
        train_id = request.data.get('train_id')
        travel_date = request.data.get('travel_date')  # 'YYYY-MM-DD'
        travel_class = request.data.get('travel_class')
        number_of_seats = int(request.data.get('number_of_seats', 0))

        try:
            with transaction.atomic():
                with connection.cursor() as cursor:
                    cursor.execute(
                        "SELECT fn_reserve_booking(%s, %s, %s, %s, %s);",
                        [user.id, train_id, travel_date, travel_class, number_of_seats]
                    )
                    booking_id = cursor.fetchone()[0]
            return Response({"booking_id": booking_id}, status=status.HTTP_201_CREATED)
        except Exception as e:
            return Response({"error": str(e)}, status=status.HTTP_400_BAD_REQUEST)
Example: Cancel booking
class BookingCancelView(APIView):
    permission_classes = [IsAuthenticated]
    def post(self, request, booking_id):
        user = request.user
        try:
            with transaction.atomic():
                with connection.cursor() as cursor:
                    cursor.execute("SELECT fn_cancel_booking(%s, %s);", [booking_id, user.id])
                    success = cursor.fetchone()[0]
            if success:
                return Response({"message":"Booking cancelled"}, status=200)
            else:
                return Response({"message":"Already cancelled"}, status=400)
        except Exception as e:
            return Response({"error": str(e)}, status=400)
Example: Booking history
class BookingHistoryView(APIView):
    permission_classes = [IsAuthenticated]
    def get(self, request):
        with connection.cursor() as cursor:
            cursor.execute("SELECT * FROM fn_get_booking_history(%s);", [request.user.id])
            rows = cursor.fetchall()
        data = []
        for r in rows:
            data.append({
                "id": r[0],
                "train_id": r[1],
                "travel_date": str(r[2]),
                "travel_class": r[3],
                "number_of_seats": r[4],
                "total_fare": float(r[5]) if r[5] is not None else None,
                "status": r[6],
                "booking_date": r[7].isoformat() if r[7] else None
            })
        return Response(data)
Migration & deployment steps (recommended)
Create a dev DB snapshot and test functions there first.

Create SQL functions in a migration file:

Use Django RunSQL operations in a migration or run psql manually.

Add DB indexes where needed.

Update Django views to call functions (keep old views disabled until tested).

Add unit tests:

Concurrent booking tests to ensure locks work (simulate multiple requests).

Seat-count edge cases.

Logging and error mapping: translate DB exceptions (e.g., RAISE EXCEPTION) to friendly API messages.

Monitoring: track deadlocks/timeouts; tune timeouts.

Rollback: keep ORM code or backup to revert quickly.

Notes & Best Practices
Transactions & Locking: Always use FOR UPDATE and wrap booking code in transaction.atomic() to avoid races.

Two models for seat counts: either:

Keep available_seats as base capacity + compute remaining by subtracting bookings (preferred for audit/history).

Or update available_seats on booking/cancel (faster reads, but requires careful sync).

Validation: Validate travel_date format in Django before calling DB function.

Error handling: DB RAISE EXCEPTION will propagate; catch in Django to return readable JSON.

Permissions: Keep admin-only restrictions in Django view permissions, not in DB (DB functions assume caller is authorized).

Testing concurrency: Use pytest with threads/processes to simulate simultaneous bookings.

Want me to generate these for your exact schema?
I created templates using reasonable assumptions about your models and JSON fields. I can now:

Adapt all SQL functions to match your exact table/column names (I used trains_train, trains_station, bookings_booking, and fields like available_seats, fare_per_class).

Create Django migration RunSQL files for the functions.

Produce full updated view files replacing the ORM queries.

If you want that, paste (or confirm) the exact table/field names and whether you want:

to keep updating available_seats in the train row on booking (mutable capacity), or

to keep available_seats static and compute remaining seats by subtracting bookings_booking (immutable capacity + compute on read).

I’ll then produce the complete SQL + Django migration + updated views ready to drop into your repo.





