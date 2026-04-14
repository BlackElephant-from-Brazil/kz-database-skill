# Triggers & Functions — KZ Serviços Database

## Functions

### `public.get_user_role()`
Returns the `user_role` of the authenticated user. Used by all RLS policies.
```sql
CREATE OR REPLACE FUNCTION public.get_user_role()
RETURNS user_role AS $$
  SELECT role FROM public.users WHERE id = auth.uid();
$$ LANGUAGE sql SECURITY DEFINER STABLE;
```

### `set_address_location()`
Auto-populates `addresses.location` (GEOGRAPHY) from `latitude`/`longitude`.
```sql
NEW.location := ST_SetSRID(ST_MakePoint(NEW.longitude, NEW.latitude), 4326)::geography;
```
**Trigger**: `trg_set_address_location` — BEFORE INSERT OR UPDATE OF latitude, longitude ON addresses

### `set_driver_location()`
Auto-populates `driver_locations.location` (GEOGRAPHY) from `latitude`/`longitude`.
**Trigger**: `trg_set_driver_location` — BEFORE INSERT OR UPDATE OF latitude, longitude ON driver_locations

### `log_trip_status_change()`
Records trip status changes to `trip_status_history`. Uses `auth.uid()` or `client_id` as `changed_by`.
```sql
IF OLD.status IS DISTINCT FROM NEW.status THEN
  INSERT INTO trip_status_history (trip_id, from_status, to_status, changed_by)
  VALUES (NEW.id, OLD.status::VARCHAR, NEW.status::VARCHAR, COALESCE(auth.uid(), NEW.client_id));
END IF;
```
**Trigger**: `trg_log_trip_status_change` — AFTER UPDATE OF status ON trips

### `log_service_request_status_change()`
Records service request status changes to `service_request_status_history`.
**Trigger**: `trg_log_service_request_status_change` — AFTER UPDATE OF status ON service_requests

### `recalculate_provider_rating()`
Recalculates `provider_profiles.average_rating` and `total_ratings` when a new rating is inserted.
**Trigger**: `trg_recalculate_provider_rating` — AFTER INSERT ON ratings

### `update_updated_at_column()`
Generic function that sets `NEW.updated_at = now()`.
Applied to: `users`, `provider_profiles`, `driver_profiles`, `vehicles`, `trips`, `service_requests`, `driver_locations`

## Trigger Summary

| Trigger | Table | Event | Function |
|---------|-------|-------|----------|
| `trg_set_address_location` | addresses | BEFORE INSERT/UPDATE(lat,lng) | `set_address_location()` |
| `trg_set_driver_location` | driver_locations | BEFORE INSERT/UPDATE(lat,lng) | `set_driver_location()` |
| `trg_log_trip_status_change` | trips | AFTER UPDATE(status) | `log_trip_status_change()` |
| `trg_log_service_request_status_change` | service_requests | AFTER UPDATE(status) | `log_service_request_status_change()` |
| `trg_recalculate_provider_rating` | ratings | AFTER INSERT | `recalculate_provider_rating()` |
| `trg_users_updated_at` | users | BEFORE UPDATE | `update_updated_at_column()` |
| `trg_provider_profiles_updated_at` | provider_profiles | BEFORE UPDATE | `update_updated_at_column()` |
| `trg_driver_profiles_updated_at` | driver_profiles | BEFORE UPDATE | `update_updated_at_column()` |
| `trg_vehicles_updated_at` | vehicles | BEFORE UPDATE | `update_updated_at_column()` |
| `trg_trips_updated_at` | trips | BEFORE UPDATE | `update_updated_at_column()` |
| `trg_service_requests_updated_at` | service_requests | BEFORE UPDATE | `update_updated_at_column()` |
| `trg_driver_locations_updated_at` | driver_locations | BEFORE UPDATE | `update_updated_at_column()` |

## Realtime Publications

```sql
ALTER PUBLICATION supabase_realtime ADD TABLE driver_locations;
ALTER PUBLICATION supabase_realtime ADD TABLE chat_messages;
ALTER PUBLICATION supabase_realtime ADD TABLE notifications;
```

## PostgreSQL Extensions

- **PostGIS**: `CREATE EXTENSION IF NOT EXISTS "postgis";`
- **pg_trgm**: `CREATE EXTENSION IF NOT EXISTS "pg_trgm";`
- **unaccent**: `CREATE EXTENSION IF NOT EXISTS "unaccent";`
