
——1 Counting Matching Vehicles Based on Specific Configuration and Availability
SELECT COUNT(DISTINCT v.vin) AS NumberOfMatchingVehicles
FROM vehicle v
INNER JOIN order o ON v.wheels = o.wheels
                    AND v.paint = o.paint
                    AND v.seats = o.seats
                    AND v.stage = o.stage
		    AND SUBSTR(o.location, 4, 2) = v.country ----different countries can have different regulations for cars config
WHERE v.is_available_for_match = 1 
      AND o.is_available_for_match = 1 
      AND v.stage != 'ineligible' 
      AND o.stage != 'ineligible';

——485


——2 Counting Matching Vehicles Within Logistics Route
WITH vehicle_location AS (
    SELECT v.*, l.logistics_route
    FROM vehicle v
    JOIN location l ON v.location = l.location
    WHERE v.is_available_for_match = 1 AND v.stage != 'ineligible'
),
order_location AS (
    SELECT o.*, l.logistics_route
    FROM "order" o
    JOIN location l ON o.location = l.location
    WHERE o.is_available_for_match = 1 AND o.stage != 'ineligible'
),
matched AS (
    SELECT vl.vin, ol.id, vl.logistics_route,
           ROW_NUMBER() OVER (PARTITION BY vl.vin ORDER BY ol.reservation_date) AS rn ---ensures that each vehicle is matched to only one order
    FROM vehicle_location vl
    JOIN order_location ol ON vl.wheels = ol.wheels
                            AND vl.paint = ol.paint
                            AND vl.seats = ol.seats
                            AND vl.stage = ol.stage
                            AND SUBSTR(vl.location, 4, 2) = SUBSTR(ol.location, 4, 2)
                            AND vl.logistics_route = ol.logistics_route
)
SELECT COUNT(DISTINCT vin) AS NumberOfMatchingVehiclesWithinLogisticsRoute
FROM matched
WHERE rn = 1;

——411


——3 Listing Vehicles and Matched Orders Based on Exact Location
WITH vehicle_location AS (
    SELECT v.*, l.logistics_route
    FROM vehicle v
    JOIN location l ON v.location = l.location
    WHERE v.is_available_for_match = 1 AND v.stage != 'ineligible'
),
order_location AS (
    SELECT o.*, l.logistics_route
    FROM "order" o
    JOIN location l ON o.location = l.location
    WHERE o.is_available_for_match = 1 AND o.stage != 'ineligible'
),
matched AS (
    SELECT vl.vin, ol.id AS order_id, vl.logistics_route,
           ROW_NUMBER() OVER (PARTITION BY vl.vin ORDER BY ol.reservation_date) AS rn
    FROM vehicle_location vl
    JOIN order_location ol ON vl.wheels = ol.wheels
                            AND vl.paint = ol.paint
                            AND vl.seats = ol.seats
                            AND vl.stage = ol.stage
                            AND vl.location = ol.location  -- Matching on exact locations
)
SELECT vin, order_id
FROM matched
WHERE rn = 1;

----output in csv


——4 Prioritizing and Matching Vehicles to Orders Based on Location, Metro Area, and Logistics Route
WITH vehicle_location AS (
    SELECT v.*, l.location AS vehicle_location, l.metro AS vehicle_metro, l.logistics_route
    FROM vehicle v
    JOIN location l ON v.location = l.location
    WHERE v.is_available_for_match = 1 AND v.stage != 'ineligible'
),
order_location AS (
    SELECT o.*, l.location AS order_location, l.metro AS order_metro, l.logistics_route
    FROM "order" o
    JOIN location l ON o.location = l.location
    WHERE o.is_available_for_match = 1 AND o.stage != 'ineligible'
),
matched AS (
    SELECT vl.vin, ol.id AS order_id,
           CASE
               WHEN vl.vehicle_location = ol.order_location THEN 1
               WHEN vl.vehicle_metro = ol.order_metro THEN 2
               WHEN vl.logistics_route = ol.logistics_route THEN 3
               ELSE 4
           END AS priority,
           ROW_NUMBER() OVER (PARTITION BY vl.vin ORDER BY 
                              CASE
                                  WHEN vl.vehicle_location = ol.order_location THEN 1
                                  WHEN vl.vehicle_metro = ol.order_metro THEN 2
                                  WHEN vl.logistics_route = ol.logistics_route THEN 3
                                  ELSE 4
                              END, ol.reservation_date) AS rn
    FROM vehicle_location vl
    JOIN order_location ol ON vl.logistics_route = ol.logistics_route
                           AND vl.wheels = ol.wheels
                           AND vl.paint = ol.paint
                           AND vl.seats = ol.seats
                           AND vl.stage = ol.stage
)
SELECT vin, order_id
FROM matched
WHERE rn = 1;

----output in csv

