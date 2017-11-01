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
- the server sees requests as approximately a time-series (interleaving requests from different clients, and cancel being just a kind of request)
- the clients see each job as apprimately a single key-value (status)

Security:
- ideally the macaroons issued to clients for the job API would specify their client ID

Notes:
- type used for time should be consistent with recent/ongoing discussion

## Client API:

### Submit job

- URL: `/<DSID>/job/`
- METHOD: POST
- Body: new JSON-encoded job record
- Returns: JSON object with `job` and `client` (future cases may be server allocated or implicit)
- Note: returns immediately

Job record is JSON object with
- `job` (string) - client-allocated GUID to uniquely identify job
- `client` (string) - client-specific ID or nonce (may be checked/enforced with macaroon extension)
- `request` (JSON) - request-specific data
- (`timestamp` is added by store on receipt)

Note: will generate a notification to server of new request. Initial status is "submitted".

### Cancel job

- URL: `/<DSID>/job/<JOBID>/cancel`
- METHOD: POST
- Request Body: JSON object with `client` (may be replaced by macaroon extension)
- Returns: last known status (see below)
- Note: returns immediately; job will be cancelled asynchronously
- Error: 404 if job/client unknown; 403 if job belongs to another client.

### Get job status (poll)

- URL: `/<DSID>/job/<JOBID>/`
- METHOD: GET
- Request body: JSON object with `job` and `client`
- Returns: JSON job status
- Note: status is status last know to store

Job status record is JSON object with
- `job` (string) as per job record (above)
- `client` (string) as per job record (above)
- `status` (string) job status: "submitted", "accepted" (by server), "complete", "cancelled", "forbidden", "failed", "retrying"
- `response` (JSON) - request-specific response (if complete)
- `error` (string) error message 

### Subscribe to job status changes

- URL: `/sub/<DSID>/job`
- Method: POST
- Request body: JSON object with `client` (may be replaced or checked/enforced by macaroon extension)

Notification is JSON object with
- `datasource_id` - DSID
- `job` - job ID
- `type` `status` (may not be needed?)
- `client` - client ID
- `data` - new status record

Note that notifications should be filtered by client ID, i.e. each client should only see status changes for its own jobs.

## Server API

Note that clients have permission to GET and POST `/<DSID>/` so a different path prefix is needed for server-only operations.

### Update job status

- URL: `/jobs/<DSID>/job/<JOBID>/`
- Method: POST
- Request body: JSON job status (see Get job status, above)

Note: will generate notification of status to client.

### Subscribe to job requests

- URL: `/sub/<DSID>/jobs`
- Method: POST
- Request body: none

Notification is JSON object with
- `datasource_id` - DSID
- `type` `request` (may not be needed?)
- `client` - client ID
- `data` - new job record or job cancel record
- `timestamp` - server-assigned request timestamp

Note that notifications should be filtered by client ID, i.e. each client should only see status changes for its own jobs.

### Get latest request

- URL: `/jobs/<DSID>/latest`
- Method: GET
- Returns: JSON encoded latest request object (as per server notification, above)

### Get requests since time

- URL: `/jobs/<DSID>/since/<TIME>`
- Method: GET
- Returns: JSON encoded array or request since given time, inclusive
