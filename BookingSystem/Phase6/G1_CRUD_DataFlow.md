# G1 – CRUD Data Flow: Booking System (Phase 6)

> Each diagram models the actual data flow observed in the Phase 6 Booking System.
> Verified via: Browser DevTools (Network + Console) and VS Code source code search.
> App running at: `http://localhost:5002`

---

## C – CREATE a Resource

> Endpoint: `POST /api/resources`
> Observed response: `{"ok": true, "data": {"id": 3, "name": "qweert", ...}}`

```mermaid
sequenceDiagram
    participant U as User (Browser)
    participant F as Frontend (resources.js / form.js)
    participant B as Backend Route (POST /api/resources)
    participant S as Service Layer
    participant DB as PostgreSQL

    U->>F: Fills in form (name, description, price, price_unit)
    F->>F: Client-side validation (required fields check)

    alt Validation fails (client-side)
        F-->>U: 400 - Show inline error messages (no request sent)
    else Validation passes
        F->>B: POST /api/resources {name, description, available, price, price_unit}
        B->>B: Server-side validation

        alt Invalid or missing fields
            B-->>F: 400 Bad Request {ok: false, errors: [...]}
            F-->>U: Display validation errors
        else Fields valid
            B->>S: createResource(data)
            S->>DB: INSERT INTO resources (...) VALUES (...)

            alt Duplicate resource (unique constraint)
                DB-->>S: Unique violation error
                S-->>B: Conflict
                B-->>F: 409 Conflict {ok: false, message: "Resource already exists"}
                F-->>U: Show conflict error
            else Insert successful
                DB-->>S: New row
                S-->>B: {id: 3, name: "qweert", description: "...", available: true, price: "12.00", price_unit: "hour", created_at: "..."}
                B-->>F: 201 Created {ok: true, data: {...}}
                F-->>U: Show success message + refresh resource list
            end
        end
    end
```

---

## R – READ Resources

> Endpoint: `GET /api/resources`
> Observed response: `{"ok": true, "data": [{id, name, description, available, price, price_unit, created_at}]}`

```mermaid
sequenceDiagram
    participant U as User (Browser)
    participant F as Frontend (resources.js)
    participant B as Backend Route (GET /api/resources)
    participant S as Service Layer
    participant DB as PostgreSQL

    U->>F: Navigates to Resources page (page load)
    F->>B: GET /api/resources

    alt Database connection error
        B-->>F: 500 Internal Server Error {ok: false, message: "READ ALL failed"}
        F-->>U: Show error message
    else Connected successfully
        B->>S: getAllResources()
        S->>DB: SELECT * FROM resources ORDER BY created_at

        alt No resources found
            DB-->>S: Empty array []
            S-->>B: []
            B-->>F: 200 OK {ok: true, data: []}
            F-->>U: Show empty resource list
        else Resources found
            DB-->>S: Rows []
            S-->>B: Array of resource objects
            B-->>F: 200 OK {ok: true, data: [{id, name, description, available, price, price_unit, created_at}, ...]}
            F-->>U: Render resource list in UI
        end
    end
```

---

## U – UPDATE a Resource

> Endpoint: `PUT /api/resources/:id`
> Confirmed via source code: `method = "PUT"` in `form.js`

```mermaid
sequenceDiagram
    participant U as User (Browser)
    participant F as Frontend (resources.js / form.js)
    participant B as Backend Route (PUT /api/resources/:id)
    participant S as Service Layer
    participant DB as PostgreSQL

    U->>F: Clicks resource from list to select it
    F->>F: Populates form with resource data
    F->>F: Sets hidden resourceId input field
    U->>F: Edits fields and clicks "Update" button

    F->>F: Client-side validation (required fields)

    alt Validation fails (client-side)
        F-->>U: Show inline error (no request sent)
    else Validation passes
        F->>B: PUT /api/resources/:id {name, description, available, price, price_unit}
        B->>B: Server-side validation

        alt Missing resource ID
            B-->>F: 400 Bad Request {ok: false, message: "Update failed: missing resource ID"}
            F-->>U: Show error message
        else Fields valid
            B->>S: updateResource(id, data)
            S->>DB: UPDATE resources SET name=$1, description=$2, ... WHERE id=$3

            alt Resource not found
                DB-->>S: 0 rows affected
                S-->>B: Not found
                B-->>F: 404 Not Found {ok: false, message: "Resource not found"}
                F-->>U: Show not found error
            else Update successful
                DB-->>S: Updated row
                S-->>B: Updated resource object
                B-->>F: 200 OK {ok: true, data: {id, name, description, available, price, price_unit, created_at}}
                F-->>U: Show "successfully updated" message + refresh list
            end
        end
    end
```

---

## D – DELETE a Resource

> Endpoint: `DELETE /api/resources/:id`
> Observed: `DELETE http://localhost:5002/api/resources/2` → `204 No Content`

```mermaid
sequenceDiagram
    participant U as User (Browser)
    participant F as Frontend (resources.js / form.js)
    participant B as Backend Route (DELETE /api/resources/:id)
    participant S as Service Layer
    participant DB as PostgreSQL

    U->>F: Clicks resource from list to select it
    F->>F: Populates form + sets hidden resourceId input
    U->>F: Clicks "Delete" button

    alt No resource selected (missing ID)
        F-->>U: Show error "Select a resource first" (no request sent)
    else Resource selected
        F->>B: DELETE /api/resources/:id

        B->>S: deleteResource(id)
        S->>DB: DELETE FROM resources WHERE id = $1

        alt Resource not found
            DB-->>S: 0 rows affected
            S-->>B: Not found
            B-->>F: 404 Not Found {ok: false, message: "Resource not found"}
            F-->>U: Show not found error
        alt Resource has active bookings (FK constraint)
            DB-->>S: Foreign key violation
            S-->>B: Conflict
            B-->>F: 409 Conflict {ok: false, message: "Cannot delete: resource has active bookings"}
            F-->>U: Show conflict warning
        else Delete successful
            DB-->>S: Row deleted
            S-->>B: Success
            B-->>F: 204 No Content (empty body)
            F-->>U: Remove resource from list + clear form
        end
    end
```

---

## Summary Table

| Operation | Method   | Endpoint                  | Success Code   | Observed / Confirmed       |
|-----------|----------|---------------------------|----------------|----------------------------|
| Create    | POST     | `/api/resources`          | 201 Created    | ✅ DevTools Network         |
| Read All  | GET      | `/api/resources`          | 200 OK         | ✅ DevTools Network         |
| Update    | PUT      | `/api/resources/:id`      | 200 OK         | ✅ Source code (form.js)    |
| Delete    | DELETE   | `/api/resources/:id`      | 204 No Content | ✅ DevTools Network         |

---

## Key Observations

- The app runs on `http://localhost:5002` (mapped from internal port 5000)
- Frontend files involved: `resources.js` and `form.js`
- Response format is consistent: `{"ok": true, "data": {...}}` for success
- The Update button is only enabled after selecting a resource from the list
- Delete returns **204 No Content** with an empty response body
- Failed validations return requests named **"invalid"** in DevTools (400 status)
- PGHOST must be set to `database` (Docker service name) not `localhost`
