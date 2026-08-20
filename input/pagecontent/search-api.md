### Overview

The Search API provides real-time access to directory data using standard FHIR search semantics.

Clients may:

- Search across supported resource types.
- Read individual resources by ID.
- Navigate paginated results using server-issued tokens.

All examples on this page are path-only and relative to the environment base URL. See [Endpoint path convention](api-overview.html#endpoint-path-convention).

### Required headers for search and read

Search and read interactions require:

- `Authorization: Bearer <jwt>`
- `Accept: application/fhir+json`

The API only ever returns FHIR JSON. `application/fhir+json` is the only accepted representation — shorthand variants such as `application/json`, and `application/fhir+xml`, are not supported.

`X-Request-ID: <uuid>` is optional but recommended. Where a client supplies it, the value must be a valid UUID and must be unique per request. Where it is not supplied, the service generates one and returns it. Supplying your own allows a request to be traced from the client, through the system, and into the service logs — the same value is echoed back in responses and included as an informational issue in every `OperationOutcome`. See [Error handling](api-overview.html#error-handling).

### Query behaviour and validation

#### Substantive search parameter required

A search must include at least one **substantive** search parameter carrying a non-empty value. A substantive parameter is one that is neither a FHIR system / result-modifier parameter nor a state filter.

- `active`, `status`, and `suppressed` are **state filters**. They are accepted, and they drive visibility filtering, but they do not count as substantive — a search filtered only by state is unbounded across every resource of a type.
- `_id`, `_lastUpdated`, and reverse-chain (`_has:`) parameters **do** count as substantive.

A search with no substantive parameter is rejected with `400 Bad Request` and an `OperationOutcome` of type `invalid`.

```http
GET /Practitioner?_count=10
```

```json
{
  "resourceType": "OperationOutcome",
  "issue": [
    {
      "code": "invalid",
      "diagnostics": "At least one non-state search parameter with a non-empty value must be provided"
    }
  ]
}
```

#### Underscore-prefixed parameters

Any underscore-prefixed parameter that is not a recognised FHIR system or result-modifier parameter, not a reverse-chain (`_has:`) parameter, and not one of the always-allowed standalone parameters (`_id`, `_lastUpdated`) is rejected with `400 Bad Request` and an `OperationOutcome` of type `invalid`. Unrecognised parameters are never silently ignored.

### Paging behaviour

Paging is token-based. Clients must use the `_getpages` token returned by the server in `Bundle.link` URLs, and should follow `Bundle.link.next` and `Bundle.link.previous` rather than reconstructing paging URLs.

```http
GET /?_getpages=<paging-token>
Authorization: Bearer <jwt>
Accept: application/fhir+json
```

#### Count and page limits

- Maximum effective `_count`: `30`
- Where `_count` is omitted, or supplied as `0`, the maximum is applied
- Values above the maximum are reduced to the maximum

Note that omitting `_count` does **not** produce a smaller page — the service applies the configured maximum rather than a smaller default.

```http
GET /Organization?_count=50
```

The result behaves as if `_count=30`.

#### Result set threshold

A search result set is limited to the first **150** matching records. This bounds paging as well as the individual page size.

A paging request that starts at or beyond the threshold (`_getpagesoffset` of `150` or more) is rejected with `400 Bad Request` and an `OperationOutcome` of type `too-costly`.

Clients that need more than the first 150 matching records should narrow the query, or use the [Export API](export-api.html) for bulk retrieval.

### Read behaviour

Read requests retrieve a specific resource by ID. IDs must conform to FHIR formatting rules.

If a resource does not exist, or is not visible due to suppression or status rules, the API returns `404 Not Found`.

```http
GET /Organization/example-healthconnect-organization-1
Authorization: Bearer <jwt>
X-Request-ID: 39f2a244-9bd8-4517-a9f7-c651baf17b71
Accept: application/fhir+json
```

### Search behaviour

- String searches are case-insensitive.
- Partial matching is supported.
- Identifier-based searches (for HPI-I and HPI-O) require exact matching.
- Multiple parameters are combined using logical `AND`.

All responses, including error responses, carry `Cache-Control: no-store`. Directory data must not be cached by intermediaries or clients.

#### Search parameters that are not permitted

The following search parameters may not be supplied by a client, and a search containing one is rejected with `400 Bad Request` and an `OperationOutcome` of type `invalid`:

- `_filter` — the service owns all `_filter` usage and injects it itself during visibility filtering. See [Data visibility rules for search](#data-visibility-rules-for-search).
- Any underscore-prefixed parameter the service does not recognise. See [Underscore-prefixed parameters](#underscore-prefixed-parameters).

#### Restricted search parameters

Some search parameters are restricted for privacy reasons and may not be used to query the directory. On `Practitioner` these are:

- `gender`
- `gender-identity`
- `address` and any `address-*` parameter
- `identifier`, where the value targets the PBS prescriber number system `http://ns.electronichealth.net.au/id/medicare-prescriber-number`

A request using a restricted parameter is rejected with `400 Bad Request` and an `OperationOutcome` of type `not-supported`. A rejected `gender` search directs the client to the [Practitioner's Recorded Sex or Gender](SearchParameter-practitioner-rsg.html) (`rsg`) parameter instead.

The same restrictions are enforced on forward-chain and reverse-chain parameters, so a restricted parameter cannot be reached indirectly — `PractitionerRole?practitioner.gender=female` is rejected in the same way as `Practitioner?gender=female`. They are also applied to every `_typeFilter` query in a bulk export request.

### Data visibility rules for search

By default, the Health Connect Provider Directory returns only resources that are **active** and **not suppressed**. The same filtering is applied to resources pulled in by `_include` and `_revinclude`, so an include cannot be used to reach resources a direct search would hide.

To retrieve inactive resources, specify the state explicitly in the query. The state element differs by resource type, and the two `status`-based types do **not** share the same value set:

| Resource type | State element | Active value | Non-active value |
|---|---|---|---|
| `Practitioner`, `PractitionerRole`, `Organization`, `HealthcareService` | `active` (boolean) | `true` | `false` |
| `Location` | `status` | `active` | `inactive` |
| `Endpoint` | `status` | `active` | `off` |

`Endpoint` uses `off` rather than `inactive`. A query of `Endpoint?status=inactive` will not return non-active endpoints.

Visibility for bulk export differs from search visibility where `_since` is used. See [Visibility rules for export](export-api.html#visibility-rules-for-export).

### Search examples

Read one organisation by FHIR ID:

```http
GET /Organization/example-healthconnect-organization-1
```

Search for an organisation by its HPI-O:

```http
GET /Organization?identifier=8003626566707032
```

Search for organisations by name and state:

```http
GET /Organization?name=Aged%20Care&address-state=QLD
```

Find healthcare services of a given type that are open after 5:00pm and where Portuguese is spoken, returning the organisation that provides each service:

```http
GET /HealthcareService?service-type=http://snomed.info/sct|789718008&languages=pt&endtime=gt1700&_include=HealthcareService:organization
```

Find practitioner roles at a given location, returning the practitioner and the organisation with each role:

```http
GET /PractitionerRole?location.address-city=Balmain&_include=PractitionerRole:practitioner&_include=PractitionerRole:organization
```

Find practitioners who hold a role at a given organisation, using a reverse chain:

```http
GET /Practitioner?_has:PractitionerRole:practitioner:organization=Organization/example-healthconnect-organization-1
```

Retrieve everything that has changed since a point in time, for local synchronisation:

```http
GET /PractitionerRole?_lastUpdated=gt2026-01-01T00:00:00Z&_include=PractitionerRole:practitioner
```

For the full list of search parameters available on each resource type — including the custom search parameters defined by this Implementation Guide, their modifiers, and the prefix operators supported on time-based parameters — see the [FHIR Artefacts](artifacts.html) index and the [CapabilityStatement](capability-statements.html).

### Unsupported operations

Public Search API interactions are read-only. The following are not supported:

- `_search`
- `_history`
- `$meta`
- `$diff`
- `$validate`
- Create, update, and delete interactions (`POST`, `PATCH`, `PUT`, `DELETE`)

Requests using an unsupported operation are rejected.
