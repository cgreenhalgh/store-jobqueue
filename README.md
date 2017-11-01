# store-jobqueue

Design for a [databox](https://github.com/me-box) store (API) for a queue of jobs (or async RPC)

Goals:
- minimal changes to other components of databox, including permissions
- clients (apps) can submit jobs and monitor their own jobs
- server (driver) can monitor submitted jobs and return outcomes
- all asynchronous

Approach:
- one datasource = one server (driver) task entry; may be shared by many clients
- marked as an actuator => clients have permission to post
- server created store so has full permissions including other paths

## Client API:

### Submit job

- URL: `/<DSID>/job/`
- METHOD: POST
- Body: new JSON-encoded job record
- Note: returns immediately

Job record is JSON object with
- `job` (string) - client-allocated GUID to uniquely identify job
- `client` (string) - client-specific ID or nonce (may be checked/enforced with macaroon extension)
- `request` (JSON) - request-specific data

### Cancel job

- URL: `/<DSID>/job/cancel`
- METHOD: POST
- Request Body: JSON object with `job` and `client`
- Returns: last known status (see below)
- Note: returns immediately; job will be cancelled asynchronously
- Error: 404 if job/client unknown; 403 if job belongs to another client.

### Get job status (poll)

- URL: `/<DSID>/job/`
- METHOD: GET
- Request body: JSON object with `job` and `client`
- Returns: JSON job status
- Note: status is status last know to store

Job status record is JSON object with
- `job` (string) as per job record (above)
- `client` (string) as per job record (above)
- `status` (string) job status: "submitted", "accepted" (by server), "complete", "cancelled", "forbidden", "failed", "retrying"
- `response` (JSON) - request-specific response

### Subscribe to job status changes

- URL: `/sub/<DSID>/job`
- Method: POST
- Request body: JSON object with `client` (may be replaced or checked/enforced by macaroon extension)

## Server API

Note that clients have permission to GET and POST `/<DSID>/` so a different path prefix is needed for server-only operations.

TODO
