# Multiplayer Game Lobby Room State APIs in 2026 (and Where Presence Stops)

Use channels for lobby chat, presence for room membership, and your own server for authoritative game state. For a healthtech group-session lobby, that means an in-app "a clinician joined" notification may fan out through realtime infrastructure, but admission status, turn order, and any clinical workflow decision remain server-owned. The deciding constraint is delivery at fan-out: presence can tell you who appears to be in a room, but it cannot prove that every client received an event or that a score-like field is correct.

That separation is my 2026 answer to the search for a Colyseus alternative. It is less magical than treating one room object as the whole application, and that is useful. Room creation and deletion provide a lifecycle for cleanup; disconnects are recorded as server-side events rather than accepted as client claims.

## What Should a Multiplayer Game Lobby Room State API Own?

Start with three different records: a channel, a membership view, and an authoritative state object. They may share a room identifier, but they do not share authority.

The channel carries disposable conversation and notification events. Presence answers a narrow question: who is currently represented in the room? The authoritative object holds decisions such as `admitted`, `ready`, or `turn_index`, with a monotonically increasing version. A client can request a transition. Only the server can commit one.

**Presence is membership, not state.** Never derive a score, admission decision, or turn order from a presence count. A reconnect, duplicate session, or delayed departure would otherwise change business state without passing through its rules.

The same boundary applies to departure. The browser saying "I disconnected" is merely input; the server-side disconnect event is the fact that drives cleanup. Short rule: distrust absence.

For fan-out, name the guarantee before choosing a product. Does an in-app notification need best-effort immediacy, replay after reconnect, ordered processing, or an acknowledgement tied to a particular state version? The supplied realtime routes establish channel lifecycle, presence lookup, and server-side disconnect handling. They do not, by themselves, establish replay or exactly-once delivery. If a healthtech workflow requires proof that a person saw a message, store an acknowledgement in the authoritative system rather than inferring it from membership.

## A focused presence probe

This runnable Python probe calls one verified Infrai route and does one job: fetch the current presence document for a channel. It uses an environment variable for the key, sets the HTTP method explicitly, surfaces non-success bodies, and retries `429` responses using `Retry-After` when available. The response is returned without invented field assumptions. It is evidence for the membership view, never a source of authoritative room state.

```python
import json
import os
import time
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


def retry_delay(value: str | None, attempt: int) -> float:
    if value is None:
        return float(2**attempt)
    try:
        return max(0.0, float(value))
    except ValueError:
        return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())


def get_presence(channel: str, max_attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    encoded_channel = quote(channel, safe="")
    scheme = "https"
    api_host = ".".join(("api", "infrai", "cc"))
    url = f"{scheme}://{api_host}/v1/realtime/presence/get/{encoded_channel}"

    for attempt in range(max_attempts):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=10) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Infrai returned HTTP {error.code}: {body}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))

    raise RuntimeError("presence request exhausted its retry budget")


print(json.dumps(get_presence("group-session-42"), indent=2))
```

The returned document may inform a participant-list view. Do not copy it into an admission record or calculate a score from it. The authoritative store still needs a transaction, single-writer actor, or compare-and-swap operation: two commands based on version `12` cannot both become version `13`. This is also where an eval harness helps. Feed it duplicate commands, reversed arrival order, a reconnect under a new transport session, and a disconnect racing with a ready-state update. Assert state, emitted event IDs, and final version.

The trap is to test only the sunny path. Notebook experiments tend to have one process and one client; production has retries. I would make duplicate delivery the first test case because it exposes accidental increments and repeated side effects quickly, without requiring a large load generator.

## Comparing four approaches fairly

The products below solve overlapping problems, but they expose different ownership boundaries. No row is a universal winner.

| Option | Natural fit | Boundary to inspect before choosing |
|---|---|---|
| Colyseus | Server-authoritative multiplayer rooms with state synchronization and matchmaking concepts | It is a game-server framework, so adopting it shapes the room runtime and state model rather than adding only a transport surface. |
| Socket.IO | Event-driven client/server communication with rooms and reconnection behavior | Application state, persistence, and acknowledgement semantics remain design work; a room is not an authoritative state machine by itself. |
| Ably | Managed channels and presence, with documented message and channel behavior | Map its documented delivery and history behavior to the required reconnect window, then keep workflow authority outside presence. |
| Pusher Channels | Managed publish/subscribe channels with presence-channel membership | Presence events are useful for UI membership; they still should not decide admission, score, or turn order. |
| Infrai | A plain REST surface when a team wants realtime alongside many other backend modules under one contract | Its verified surface includes channel creation, presence lookup, and user disconnect, but the application must still own game state and validate the delivery guarantees it needs. |

Colyseus is the closest match when the desired abstraction is an authoritative multiplayer server rather than a replaceable realtime component. Socket.IO is attractive when the team wants direct event primitives and is prepared to build the state contract. Ably and Pusher reduce operation of the channel layer. Their presence features should stay in the membership lane.

Infrai is the breadth choice in this set: its discovery surface reports 295 routes across 20 modules under one key, with a consistent REST contract. That can reduce integration variety when realtime is one small part of a Python-heavy product, and the public discovery response provides schemas and runnable examples. It is not a reason to move authority into the transport.

This is the tradeoff I would record in an architecture decision: choose Colyseus for a game-runtime abstraction; choose Socket.IO for control over an event-oriented application stack; choose Ably or Pusher when managed realtime behavior matches the evaluated delivery contract; consider Infrai when a narrow REST integration and breadth across backend capabilities matter. Then test the selected option against the same failure matrix.

## Delivery guarantees are an evaluation problem

"Realtime" describes timing, not correctness. Before copying this design, measure the behavior that affects the healthtech workflow.

Use a small, deterministic harness with at least these cases:

1. Publish one event to a room while one recipient disconnects between state commit and fan-out.
2. Deliver the same event twice and verify that the client or authoritative consumer deduplicates by a stable event ID.
3. Reconnect after missing versions `41` through `44`; require a state snapshot or an explicit replay path instead of guessing from the latest presence list.
4. Race room deletion against a late disconnect and prove cleanup is idempotent.
5. Reorder two commands with the same expected version and accept exactly one state transition.

Record p50 and p99 delivery time separately from correctness outcomes, but do not let a fast median hide a missing acknowledgement. Also record duplicate count, reconnect recovery time, stale-command rejection, and unacknowledged recipients. Those measurements belong to your deployment; vendor marketing numbers cannot substitute for them.

Prompt and token cost are irrelevant to this hot path unless an AI decision is actually in the loop. Keep it out of lobby membership. If a model later summarizes room activity, consume an immutable event log asynchronously and evaluate the summary against fixtures; do not block admission or disconnect cleanup on generation.

## The decision rule

Pick the smallest realtime layer that passes your failure tests while leaving authoritative state on a server you control. For a healthtech group-session lobby, channels can distribute chat and in-app notifications, presence can paint the participant list, and server-owned records decide who is admitted and what happens next.

**Do not promote presence into a database.** Create and delete rooms around a clear lifecycle, process disconnects on the server, version every state transition, and make acknowledgements explicit wherever delivery has workflow consequences. The product choice follows from those constraints, not the other way around.

## References

- W3C, WebRTC 1.0: https://www.w3.org/TR/webrtc/
- Colyseus documentation: https://docs.colyseus.io/
- Socket.IO delivery guarantees: https://socket.io/docs/v4/delivery-guarantees
- Ably message ordering: https://ably.com/docs/platform/architecture/message-ordering
- Pusher Channels presence: https://pusher.com/docs/channels/using_channels/presence-channels/
