# Petstore API tests (Postman + Newman)

[![API tests](https://github.com/afzal-ahmed-qa/petstore-api-tests-postman/actions/workflows/api-tests.yml/badge.svg)](https://github.com/afzal-ahmed-qa/petstore-api-tests-postman/actions/workflows/api-tests.yml)

A Postman collection that tests the public [Swagger Petstore](https://petstore.swagger.io/) API,
run with Newman in GitHub Actions on every push and once a day. The latest HTML report is here:
https://afzal-ahmed-qa.github.io/petstore-api-tests-postman/ (and the data-driven run is
[here](https://afzal-ahmed-qa.github.io/petstore-api-tests-postman/data-driven.html)).

This started life as a collection I generated from the Swagger file and added tests to by hand.
I've since rebuilt it around real flows, made it clean up after itself, and written down the
places where the API doesn't match its own spec.

## What it covers

The collection is split into folders that run in order:

1. **Warm-up.** The public Petstore can take several seconds to answer after a quiet period, so
   the first call just wakes it up.
2. **Pet lifecycle.** Add a pet, read it back and check it against a JSON schema, update it with
   PUT and with form data, upload a photo, find it by status and by tag, delete it, and confirm
   it's gone.
3. **Store orders.** Inventory, then place, read, cancel and re-check an order.
4. **User lifecycle.** Create a user, log in, update the profile, log out, create users in bulk
   with both `createWithArray` and `createWithList`, and delete everything again.
5. **Negative cases and spec checks.** Unknown IDs, malformed JSON, wrong types, missing required
   fields, invalid filters, invalid orders, wrong passwords.
6. **Data-driven.** One request that checks `findByStatus` for every status listed in
   `data/pet-statuses.csv`.

All 20 endpoints in the Petstore spec are called at least once. A full run is 47 requests and
about 150 assertions, and takes around 15 seconds.

Every request also gets two checks from the collection level: the response time stays under
3 seconds (configurable with `maxResponseTimeMs`), and anything with a body is returned as JSON.

## Things I tried to do properly

The Petstore is shared by everyone who wants to practise API testing, so fixed IDs like `1` or
`12345` get overwritten by other people all the time. Every run generates its own IDs, names and
usernames in pre-request scripts and deletes what it created. Even the negative tests clean up
when the API wrongly accepts something.

Requests pass data along through collection variables (the pet ID from "Add a new pet" is used
by everything after it), so the folders read like a real user's sequence of calls rather than a
list of unrelated endpoints.

Where the API disagrees with its spec, I didn't just change the assertion to whatever the API
returned. The spec check stays in the collection as a skipped test called
`Known issue PS-NN: ...`, so it shows in the report (9 of them at the moment), and the run stays
green for everything else. The details are in [docs/findings.md](docs/findings.md). The two worst:

- Login works with any username and password (PS-01).
- `GET /user/{username}` sends back the user's password in plain text (PS-02).

## Running it

You can import `collections/petstore.postman_collection.json` and
`environments/petstore.postman_environment.json` into Postman and use the Collection Runner. To
run it from the command line instead you need Node.js 20 or newer:

```bash
npm ci
npm test            # whole collection, HTML report in reports/index.html
npm run test:data   # the data-driven folder with data/pet-statuses.csv
```

To run a single folder:

```bash
npx newman run collections/petstore.postman_collection.json \
  -e environments/petstore.postman_environment.json \
  --folder "03 User lifecycle"
```

The environment only holds the base URL and the public demo API key (`special-key`). There are
no real credentials anywhere in the current files.

## Layout

```
collections/    the Postman collection
environments/   base URL and demo API key
data/           CSV for the data-driven run, image for the upload test
docs/           findings: where the API doesn't match its spec
.github/        Newman run in GitHub Actions, report published to Pages
```

## Related

- [playwright-typescript-e2e-framework](https://github.com/afzal-ahmed-qa/playwright-typescript-e2e-framework):
  UI and API automation with Playwright and TypeScript.
- [manual-test-plans-ecommerce](https://github.com/afzal-ahmed-qa/manual-test-plans-ecommerce):
  test strategy, plans and defect reports for an e-commerce site and its API.

---

Afzal Ahmed, [github.com/afzal-ahmed-qa](https://github.com/afzal-ahmed-qa)
