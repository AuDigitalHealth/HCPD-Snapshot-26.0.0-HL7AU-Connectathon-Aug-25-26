### Overview

The Health Connect Provider Directory API (HCPD API) provides FHIR-based access to provider and organisation directory data, enabling secure search, retrieval, and bulk export capabilities.

The HCPD API is the national implementation of the HCPD FHIR Implementation Guide.

This specification defines how clients interact with the service in practice, focusing on supported operations, validation rules, and operational requirements for integration. While the FHIR IG remains the normative source of truth for resource structure and semantics, these API pages define the behaviour and capabilities available through the Health Connect APIs.

Detailed behaviour for each capability is described on its own page:

- [Search API technical specification](search-api.html) — real-time search and read behaviour. Use the Search API for targeted, real-time queries and interactive lookups.
- [Export API technical specification](export-api.html) — asynchronous bulk data export behaviour. Use the Export API for large-scale extracts and local data synchronisation.

### API features and capabilities

The HCPD API supports two primary public capabilities:

- Real-time search and read access.
- Asynchronous bulk data export.

Search capabilities allow clients to locate healthcare providers, organisations, services, and related entities using FHIR-compliant query parameters, paging, and filtering.

Export capabilities enable clients to request large datasets asynchronously using the FHIR Bulk Data Access `$export` pattern for synchronisation to local directories.

### Supported resources

The public API provides access to the following FHIR resources:

- `Organization`
- `Practitioner`
- `PractitionerRole`
- `HealthcareService`
- `Location`
- `Endpoint`
- `Provenance`

All resources conform to HCPD FHIR profiles, including constraints on structure, cardinality, search parameters, and terminology.

#### Elements not supported by the service

Some elements permitted by the underlying FHIR profiles are not accepted or stored by the Health Connect Provider Directory service. Only `Practitioner` is affected.

For each element below:

- **On write**, the element is stripped from the resource before it is persisted. A create or update carrying it still succeeds — only the restricted element is removed, and other identifiers and extensions are retained. A subsequent read never returns it.
- **On read and search**, the element is never present in a response. Clients must not rely on it being present.
- **On search**, where the element also has a corresponding search parameter, that parameter is **rejected** — including when it is reached through a forward or reverse chain. See [Restricted search parameters](search-api.html#restricted-search-parameters).

<table class="grid">
	<thead>
		<tr>
			<th>Profile</th>
			<th>Element</th>
			<th>Cardinality in profile</th>
			<th>Search behaviour</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td rowspan="6">Practitioner</td>
			<td>Gender Identity</td>
			<td>0..*</td>
			<td>Search parameter rejected</td>
		</tr>
		<tr>
			<td>Gender</td>
			<td>0..1</td>
			<td>Search parameter rejected</td>
		</tr>
		<tr>
			<td>Birth Date</td>
			<td>0..1</td>
			<td>No search parameter</td>
		</tr>
		<tr>
			<td>Photo</td>
			<td>0..*</td>
			<td>No search parameter</td>
		</tr>
		<tr>
			<td>Address</td>
			<td>0..*</td>
			<td>Search parameter rejected</td>
		</tr>
		<tr>
			<td>PBS Prescriber Number</td>
			<td>1..*</td>
			<td>Search parameter rejected</td>
		</tr>
	</tbody>
</table>

**Note:** In lieu of gender and gender identity, only *Recorded Sex or Gender (RSG)* sourced from the HI Service is supported in the Health Connect Provider Directory, and can be stored and disclosed through the Directory. See the [Practitioner's Recorded Sex or Gender](SearchParameter-practitioner-rsg.html) search parameter.

The PBS prescriber number is identified by the system `http://ns.electronichealth.net.au/id/medicare-prescriber-number`.

### Environments and base URLs

| Environment | Base URL | Usage |
|---|---|---|
| SIT | `https://sit.healthconnect.digitalhealth.gov.au/api/v1/fhir` | Stable integration testing with test data. |
| SVT | `https://svt.healthconnect.digitalhealth.gov.au/api/v1/fhir` | Production-like data and performance testing. |
| PROD | `https://healthconnect.digitalhealth.gov.au/api/v1/fhir` | Live environment (post-conformance). |

#### Endpoint path convention

All examples in these API pages are path-only and are relative to the environment base URL above.

Example:

- Full URL: `https://sit.healthconnect.digitalhealth.gov.au/api/v1/fhir/Organization?name=clinic`
- Path-only form used in examples: `/Organization?name=clinic`

### Authentication and authorisation

All API requests must include a valid OAuth2 bearer token (JWT) issued by the Health Connect Authorisation Service (HCAS).

#### Required scopes by operation

Clients are not required to present all scopes on every request. The required scope is operation-specific:

| Operation | Required scope |
|---|---|
| Search operations | `search` |
| Read operations | `read` |
| Export kickoff, polling, and file download operations | `export` |

High-level authorisation flow:

1. Client registration (onboarding).
2. Token issued by the identity provider (HCAS).
3. Token sent as `Authorization: Bearer <jwt>`.

Authentication example:

```http
GET /Organization?identifier=8003626566707032
Authorization: Bearer <jwt>
X-Request-ID: 6f0d0f2e-3e42-4a43-986a-a4ecba0ab7e4
Accept: application/fhir+json
```

### Error handling

All errors are returned as a FHIR `OperationOutcome`. Every `OperationOutcome` also carries the request correlation identifier as an additional informational issue, so that a client-side failure can be correlated with server-side logs for traceability.

```json
{
  "resourceType": "OperationOutcome",
  "issue": [
    {
      "severity": "error",
      "code": "invalid",
      "diagnostics": "some error message here"
    },
    {
      "severity": "information",
      "code": "informational",
      "diagnostics": "X-Request-ID: 9c63ec77-8d78-4bf1-a170-9909b33a786b"
    }
  ]
}
```

#### HTTP status mapping

| HTTP | Meaning | Typical cause |
|---|---|---|
| 400 | Bad Request | Invalid FHIR query or parameters; restricted search parameter; paging beyond the result threshold |
| 401 | Unauthorized | Missing or invalid token |
| 403 | Forbidden | Scope or policy restriction; unsupported resource type in an export request |
| 404 | Not Found | Resource not found or not visible |
| 409 | Conflict | Duplicate or inconsistent data |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Server Error | Internal system failure |
| 503 | Service Unavailable | Temporary system outage |
