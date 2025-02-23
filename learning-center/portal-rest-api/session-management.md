# Session Management


The REST API endpoints for session management allow you to request that a workshop session be allocated.

## Disabling portal user registration

When using the REST API to trigger creation of workshop sessions, it is recommended that user registration through the training portal web interface be disabled. This will mean that only the admin user will be able to access the web interface for the training portal directly.

```
apiVersion: training.eduk8s.io/v1alpha1
kind: TrainingPortal
metadata:
  name: educates-tutorials
spec:
  portal:
    registration:
      type: one-step
      enabled: false
  workshops:
  ...
```

## Requesting a workshop session

The form of the URL sub path for requesting the allocation of a workshop environment via the REST API is ``/workshops/environment/<name>/request/``. The name segment needs to be replaced with the name of the workshop environment. When making the request, the access token must be supplied in the HTTP ``Authorization`` header with type set as ``Bearer``:

```
curl -v -H "Authorization: Bearer <access-token>" \
<training-portal-url>/workshops/environment/<name>/request/?index_url=https://hub.test/
```

A query string parameter ``index_url`` can be supplied. When the workshop session is restarted from the workshop environment web interface, the session will be deleted, and the user will then be redirected to the supplied URL. This URL would be that for your front end web application which has requested the workshop session, allowing them to select a different workshop.

Note that the value of the ``index_url`` will not be available if session cookies are cleared, or a session URL is shared with another user. In this case a user would be redirected back to the training portal URL instead. You can override the global default for this case by specifying the index URL as part of the ``TrainingPortal`` configuration.

When successful, the JSON response from the request will be of the form:

```
{
    "name": "educaes-tutorials-w01-s001",
    "user": "8d2d0c8b-6ff5-4244-b136-110fd8d8431a",
    "url": "/workshops/session/educates-tutorials-w01-s001/activate/?token=6UIW4D8Bhf0egVmsEKYlaOcTywrpQJGi&index_url=https%3A%2F%2Fhub.test%2F",
    "workshop": "educates-tutorials",
    "environment": "educates-tutorials-w01",
    "namespace": "educates-tutorials-w01-s001"
}
```

This will include the name of the workshop session, an ID for identifying the user, and a URL path with an activation token and index URL included as query string parameters.

The users browser should be redirected to this URL path on the training portal host. Accessing the URL will activate the workshop session and then redirect the user to the workshop dashboard.

If the workshop session is not activated, which confirms allocation of the session, the session will be deleted after 60 seconds.

When a user is redirected back to the URL for the index page, a query string parameter will be supplied to notify of the reason the user is being returned. This can be used to display a banner or other indication as to why they were returned.

The name of the query string parameter is ``notification`` and the possible values are:

* ``session-deleted`` - Used when the workshop session was completed or restarted.
* ``workshop-invalid`` - Used when the name of the workshop environment supplied when attempting to create the workshop was invalid.
* ``session-unavailable`` - Used when capacity has been reached and a workshop session cannot be created.
* ``session-invalid`` - Used when an attempt is made to access a session which doesn't exist. This can occur when the workshop dashboard is refreshed sometime after the workshop session had expired and been deleted.

Note that in prior versions the name of the session was returned via the "session" property, where as the "name" property is now used. To support older code using the REST API, the "session" property is still returned, but it is deprecated and will be removed in a future version.

## Associating sessions with a user

When the workshop session is requested, a unique user account will be created in the training portal each time. This can if necessary be identified through the use of the ``user`` identifier returned in the response.

If the front end using the REST API to create workshop sessions is tracking user activity, to have the training portal associate all workshops sessions created by the same user, supply the ``user`` identifier with subsequent requests by the same user in the request parameter:

```
curl -v -H "Authorization: Bearer <access-token>" \
https://lab-markdown-sample-ui.test/workshops/environment/<name>/request/?index_url=https://hub.test/&user=<user>
```

If the supplied ID matches a user in the training portal, it will be used internally by the training portal, and the same value will be returned for ``user`` in the response.

When the user does match, if there is already a workshop session allocated to the user for the workshop being requested, a link to the existing workshop session will be returned rather than creating a new workshop session.

If there is no matching user, possibly because the training portal had been completely redeployed since the last time it was accessed, a new user identifier will be returned.

The first time that a request is made to create a workshop session for a user, where ``user`` is not supplied, you can optionally supply request parameters  for the following to have these set as the user details in the training portal.

* ``email`` - The email address of the user.
* ``first_name`` - The first name of the user.
* ``last_name`` - The last name of the user.

These details will be accessible through the admin pages of the training portal.

When sessions are being associated with a user, it is possible to query all active sessions for that user across the different workshops hosted by the instance of the training portal:

```
curl -v -H "Authorization: Bearer <access-token>" \
<training-portal-url>/workshops/user/<user>/sessions/
```

The response will be of the form:

```
{
  "user": "8d2d0c8b-6ff5-4244-b136-110fd8d8431a",
  "sessions": [
    {
      "name": "educates-tutorials-w01-s001",
      "workshop": "educates-tutorials",
      "environment": "educates-tutorials-w01",
      "namespace": "educates-tutorials-w01-s001",
      "started": "2020-07-31T03:57:33.942Z",
      "expires": "2020-07-31T04:57:33.942Z",
      "countdown": 3353,
      "extendable": false
    }
  ]
}
```

Once a workshop has expired, or has otherwise been shutdown, an entry for the workshop will no longer be returned.

## Listing all workshop sessions

To get a list of all running workshops sessions allocated to users, you can provide the ``sessions=true`` flag to the query string parameters of the REST API call for listing the workshop environments available through the training portal.

```
curl -v -H "Authorization: Bearer <access-token>" |
<training-portal-url>/workshops/catalog/environments/?sessions=true
```

The JSON response will be of the form:

```
{
  "portal": {
    "name": "educates-tutorials",
    "uid": "9b82a7b1-97db-4333-962c-97be6b5d7ee0",
    "generation": 476,
    "url": "<training-portal-url>",
    "sessions": {
      "maximum": 10,
      "registered": 0,
      "anonymous": 0,
      "allocated": 1
    }
  },
  "environments": [
    {
      "name": "educates-tutorials-w01",
      "state": "RUNNING",
      "workshop": {
        "name": "lab-et-self-guided-tour",
        "id": "15e5f1a569496f335049bb00c370ee20",
        "title": "Workshop Building Tutorial",
        "description": "A guided tour of how to build a workshop for your team's learning center.",
        "vendor": "",
        "authors": [],
        "difficulty": "",
        "duration": "",
        "tags": [],
        "logo": "",
        "url": "<workshop-repository-url>"
      },
      "duration": 1800,
      "capacity": 10,
      "reserved": 0,
      "allocated": 1,
      "available": 0,
      "sessions": [
        {
          "name": "educates-tutorials-w01-s002",
          "state": "RUNNING",
          "namespace": "educates-tutorials-w01-s002",
          "user": "672338f3-4085-4782-8d9b-ae1637e1c28c",
          "started": "2021-11-05T15:50:04.824Z",
          "expires": "2021-11-05T16:20:04.824Z",
          "countdown": 1737,
          "extendable": false
        }
      ]
    }
  ]
}
```

No workshop sessions will be returned if anonymous access to this REST API endpoint is enabled and you are not authenticated against the training portal.

Only workshop environments with a ``state`` of ``RUNNING`` will be returned by default. If you want to see workshop environments which are being shutdown, and any workshop sessions against those which still haven't been completed, supply the ``state`` query string parameter with value ``STOPPING``.

```
curl -v -H "Authorization: Bearer <access-token>" \
<training-portal-url>/workshops/catalog/environments/?sessions=true&state=RUNNING&state=STOPPING
```

The ``state`` query string parameter can be included more than once to be able to see workshop environments in both ``RUNNING`` and ``STOPPING`` states.
