# MSC3030: Jump to date API endpoint

Add an API that makes it easy to find the closest messages for a given
timestamp.

The goal of this change is to have clients be able to implement a jump to date
feature in order to see messages back at a given point in time. Pick a date from
a calender, heatmap, or paginate next/previous between days and view all of the
messages that were sent on that date.

For our [roadmap of feature parity with
Gitter](https://github.com/vector-im/roadmap/issues/26), we're also interested
in using this for a new better static Matrix archive. Our idea is to server-side
render [Hydrogen](https://github.com/vector-im/hydrogen-web) and this new
endpoint would allow us to jump back on the fly without having to paginate and
keep track of everything in order to display the selected date.

Also useful for archiving and backup use cases. This new endpoint can be used to
slice the messages by day and persist to file.

Related issue: [*URL for an arbitrary day of history and navigation for next and
previous days*
(vector-im/element-web#7677)](https://github.com/vector-im/element-web/issues/7677)


## Problem

These types of use cases are not supported by the current Matrix API because it
has no way to fetch or filter older messages besides a manual brute force
pagination from the latest. Paginating is time-consuming and expensive to
process every event as you go (not practical for clients). Imagine wanting to
get a message from 3 years ago 😫


## Proposal

Add new client API endpoint `GET
/_matrix/client/v1/rooms/{roomId}/timestamp_to_event?ts=<timestamp>?dir=[f|b]`
which fetches the closest `event_id` to the given timestamp `ts` query parameter
in the direction specified by the `dir` query parameter. The direction `dir`
query parameter accepts `f` for forward-in-time from the timestamp and `b` for
backward-in-time from the timestamp. This endpoint also returns
`origin_server_ts` to make it easy to do a quick comparison to see if the
`event_id` fetched is too far out of range to be useful for your use case.

When an event can't be found in the given direction, the endpoint throws a 404
`"errcode":"M_NOT_FOUND",` (example error message `"error":"Unable to find event
from 1672531200000 in direction f"`).

In order to solve the problem where a remote federated homeserver does not have
all of the history in a room and no suitably close event, we also add a server
API endpoint `GET
/_matrix/federation/v1/timestamp_to_event/{roomId}?ts=<timestamp>?dir=[f|b]`
which other homeservers can use to ask about their closest `event_id` to the
timestamp. This endpoint also returns `origin_server_ts` to make it easy to do a
quick comparison to see if the remote `event_id` fetched is closer than the
local one.

The heuristics for deciding when to ask another homeserver for a closer event if
your homeserver doesn't have something close, is left up to the homeserver
implementation. Although the heuristics will probably be based on whether the
closest event is a forward/backward extremity indicating it's next to a gap of
events which are potentially closer.

These endpoints are authenticated and should be rate-limited like similar client
and federation endpoints to prevent resource exhaustion abuse.

```
GET /_matrix/client/v1/rooms/<roomID>/timestamp_to_event?ts=<timestamp>&dir=<direction>
{
    "event_id": ...
    "origin_server_ts": ...
}
```

Federation API endpoint:
```
GET /_matrix/federation/v1/timestamp_to_event/<roomID>?ts=<timestamp>&dir=<direction>
{
    "event_id": ...
    "origin_server_ts": ...
}
```

---

In order to paginate `/messages`, we need a pagination token which we can get
using `GET /_matrix/client/r0/rooms/{roomId}/context/{eventId}?limit=0` for the
`event_id` returned by `/timestamp_to_event`.

We can always iterate on `/timestamp_to_event` later and return a pagination
token directly in another MSC ⏩


## Potential issues

If you ask for "the message with `origin_server_ts` closest to Jan 1st 2018" you
might actually get a rogue random delayed one that was backfilled from a
federated server, but the human can figure that out by trying again with a
slight variation on the date or something.

Clients could abuse this new API for getting the `m.room.create` event, so
servers might want to put extra care into optimizing whatever lookups they do.
The create event contains quite a lot of information that a client needs in
order to operate, so it is frequently requested by said clients. For example,
the room type and room version (for displaying warnings about stability).


## Alternatives

We chose the current `/timestamp_to_event` route because it sounded like the
easist path forward to bring it to fruition and get some real-world experience.
And was on our mind during the [initial discussion](https://docs.google.com/document/d/1KCEmpnGr4J-I8EeaVQ8QJZKBDu53ViI7V62y5BzfXr0/edit#bookmark=id.qu9k9wje9pxm) because there was some prior art with a [WIP
implementation](https://github.com/matrix-org/synapse/pull/9445/commits/91b1b3606c9fb9eede0a6963bc42dfb70635449f)
from @erikjohnston. The alternatives haven't been thrown out for a particular
reason and we could still go down those routes depending on how people like the
current design.


### Paginate `/messages` from timestamp 

Add the `?around=<timestamp>` query parameter to the `GET
/_matrix/client/r0/rooms/{roomId}/messages` endpoint. This will start the
response at the message with `origin_server_ts` closest to the provided `around`
timestamp. The direction is determined by the existing `?dir` query parameter.

Use topological ordering, just as Element would use if you follow a permalink.

### Filter by date in `RoomEventFilter`

Extend `RoomEventFilter` to be able to specify a timestamp or a date range. The
`RoomEventFilter` can be passed via the `?filter` query param on the `/messages`
endpoint.


### New `destination_server_ts` field

Add a new field and index on messages called `destination_server_ts` which
indicates when the message was received from federation. This gives a more
"real" time for how someone would actually consume those messages.

The contract of the API is "show me messages my server received at time T"
rather than the messy confusion of showing a delayed message which happened to
originally be sent at time T.

We've decided against this approach because the backfill from federated servers
could be horribly late.

---

Related issue around `/sync` vs `/messages`,
https://github.com/matrix-org/synapse/issues/7164

> Sync returns things in the order they arrive at the server; backfill returns
> them in the order determined by the event graph.
>
> *-- @richvdh, https://github.com/matrix-org/synapse/issues/7164#issuecomment-605877176*

> The general idea is that, if you're following a room in real-time (ie,
> `/sync`), you probably want to see the messages as they arrive at your server,
> rather than skipping any that arrived late; whereas if you're looking at a
> historical section of timeline (ie, `/messages`), you want to see the best
> representation of the state of the room as others were seeing it at the time.
>
> *-- @richvdh , https://github.com/matrix-org/synapse/issues/7164#issuecomment-605953296*


## Security considerations

We're only going to expose messages according to the existing message history
setting in the room (`m.room.history_visibility`). No extra data is exposed,
just a new way to sort through it all.



## Unstable prefix

While this MSC is not considered stable, the endpoints are available at `/unstable/org.matrix.msc3030` instead of their `/v1` description from above.

```
GET /_matrix/client/unstable/org.matrix.msc3030/rooms/<roomID>/timestamp_to_event?ts=<timestamp>&dir=<direction>
{
    "event_id": ...
    "origin_server_ts": ...
}
```

```
GET /_matrix/federation/unstable/org.matrix.msc3030/timestamp_to_event/<roomID>?ts=<timestamp>&dir=<direction>
{
    "event_id": ...
    "origin_server_ts": ...
}
```

Servers will indicate support for the new endpoint via a non-empty value for feature flag
`org.matrix.msc3030` in `unstable_features` in the response to `GET
/_matrix/client/versions`.