# Findings

Places where the public Petstore (`https://petstore.swagger.io/v2`) doesn't behave the way its own
[Swagger definition](https://petstore.swagger.io/) says it should. Checked on 7 November 2026.

Each one has a skipped test in the collection named `Known issue PS-NN`, so it shows up in every
report without making the run fail. If the API is ever fixed, un-skip the test and it should pass.

The Petstore is a demo, so none of this is surprising. I've written it up the way I would for a
real API, because the same problems would matter there.

| ID | Finding | Severity | Request in collection |
|---|---|---|---|
| PS-01 | Login succeeds with a wrong password, and for users that don't exist | Critical | 04 / Login with a wrong password |
| PS-02 | `GET /user/{username}` returns the user's password in plain text | Critical | 03 / Get the user |
| PS-04 | Wrong field types cause a 500 server error instead of 405 | Major | 04 / Wrong field types are rejected |
| PS-08 | Orders with an unknown status or a negative quantity are accepted | Major | 04 / Order with an unknown status… |
| PS-03 | A pet with no `name` (a required field) is accepted | Minor | 04 / Pet without the required name |
| PS-05 | `PUT /pet` creates a pet that doesn't exist instead of returning 404 | Minor | 04 / PUT on a pet that does not exist |
| PS-06 | An invalid status in `findByStatus` returns 200 and an empty list instead of 400 | Minor | 04 / Invalid status filter |
| PS-07 | A non-numeric pet ID returns 404 with a Java exception message instead of 400 | Minor | 04 / Non-numeric pet ID |
| PS-09 | 404 from `DELETE /pet/{id}` has an empty body, unlike every other error | Trivial | 04 / Deleting a pet that does not exist |

---

### PS-01: Login succeeds with a wrong password, and for users that don't exist

```bash
curl -s "https://petstore.swagger.io/v2/user/login?username=nobody_at_all&password=wrong"
```

Expected: 400 "Invalid username/password supplied" (as documented).
Actual: `200 {"code":200,"type":"unknown","message":"logged in user session:1794050274228"}`

Any username and password combination gets a session.

### PS-02: `GET /user/{username}` returns the password in plain text

Create a user with password `Secret#123`, then:

```bash
curl -s "https://petstore.swagger.io/v2/user/<username>"
```

Actual: the response includes `"password":"Secret#123"`. Passwords should never be returned by
an API, and anyone who knows a username can read it. Combined with PS-01 this would be a full
account takeover on a real system.

### PS-04: Wrong field types cause a 500

```bash
curl -s -X POST https://petstore.swagger.io/v2/pet -H "Content-Type: application/json" \
  -d '{"id": "not-a-number", "name": 123}'
```

Expected: 405 "Invalid input". Actual: `500 {"code":500,"type":"unknown","message":"something bad happened"}`.
A client mistake shouldn't be reported as a server failure; it also makes real 500s harder to spot
in monitoring.

### PS-08: Invalid orders are accepted

Placing an order with `"status": "shipped-to-mars"` and `"quantity": -5` returns 200 and stores
it exactly as sent. The spec limits status to `placed`, `approved` and `delivered`.

### PS-03: Pet without a name is accepted

`POST /pet` with `{"id": <id>, "photoUrls": []}` returns 200. `name` and `photoUrls` are marked
required in the spec.

### PS-05: `PUT /pet` creates missing pets

`PUT /pet` with an ID that doesn't exist returns 200 and creates the pet. The spec lists
`404 Pet not found` for this case.

### PS-06: Invalid status filter returns 200

`GET /pet/findByStatus?status=active` returns `200 []`. The spec says 400 "Invalid status value".
The empty list is at least safe; it doesn't return unrelated pets.

### PS-07: Non-numeric ID leaks a Java exception

```bash
curl -s https://petstore.swagger.io/v2/pet/abc
```

Actual: `404 {"code":404,"type":"unknown","message":"java.lang.NumberFormatException: For input string: \"abc\""}`.
The spec says 400 "Invalid ID supplied". The message also exposes the server's implementation language.

### PS-09: Empty body on one 404

Deleting a pet that doesn't exist returns 404 with no body at all. Other errors return a JSON
object with `code`, `type` and `message`, so clients have to handle this one specially.
