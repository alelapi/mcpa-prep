# Exam-Day Strategy

## Before the day

- [ ] Read the [Candidate Handbook](https://docs.linuxfoundation.org/tc-docs/certification/lf-handbook2) and [Important Instructions](https://docs.linuxfoundation.org/tc-docs/certification/important-instructions-mc)
- [ ] Run the system compatibility check on the exact machine and network you will use
- [ ] Have valid government-issued photo ID ready
- [ ] Confirm your appointment time and time zone

## On the day

- [ ] Clear desk, no notes, no phone within reach, no second monitor
- [ ] Private, quiet, well-lit room with nobody entering
- [ ] Close every other application and disable notifications
- [ ] Laptop on power, wired network if possible
- [ ] Log in early, since check-in takes time

!!! warning "Proctoring is strict"
    Looking away from the screen repeatedly, reading questions aloud, or another person
    entering the room can end the session. Keep your face in frame.

## Time budget

**90 minutes.** The question count is not published. If it is around 60, that gives you
**90 seconds per question**, which is comfortable for recognition questions and tight for
message-flow scenarios.

| Pass | Time | What to do |
| --- | --- | --- |
| 1 | ~55 min | Answer everything you can in under a minute. Flag anything slower. |
| 2 | ~25 min | Work the flagged questions properly. Sketch message flows on the whiteboard if one is provided. |
| 3 | ~10 min | Check nothing is blank. There is no penalty for guessing. |

## Question types and how to attack them

**Recognition.** *"Which method must every server implement?"* Either you know it or you
don't. Answer fast and bank the time.

**Direction and sequence.** *"A server needs user confirmation during a tool call. What
happens next?"* Picture the messages. Who sends what, with which `id`, and what does the
result contain? Eliminate any option in which the **server sends a request** to the client:
that is legacy behavior.

**MUST vs. SHOULD.** *"Which is REQUIRED of a client that receives an InputRequiredResult
containing requestState?"* The exact keyword matters. Echoing `requestState` is a MUST;
logging it is not mentioned at all.

**Security judgement.** *"A remote server receives a token issued for a different API.
What should it do?"* Find the violated boundary first (audience, consent, trust), then pick
the option that enforces it **at the server**, not the one that hopes the client behaves.

**Primitive selection.** *"A server wants users to trigger a code review workflow via a
slash command."* Map to the control model: user-controlled means **prompt**,
application-controlled means **resource**, model-controlled means **tool**.

## Traps specific to this exam

!!! danger "Legacy answers"
    Distractors will describe the pre-2026 protocol as if it were current:
    - "The client sends `initialize` and the server returns its capabilities"
    - "The server sends `sampling/createMessage` to the client"
    - "The client uses `Mcp-Session-Id` to resume its session"
    - "The client sends `ping` to keep the connection alive"
    - "Resource not found returns `-32002`"

    All of these were true once. None are current behavior in `2026-07-28`.

!!! danger "Deprecated is not removed"
    Sampling, Roots and Logging are **deprecated**, not **removed**. They still work and are
    still in the spec. An option saying "sampling is no longer supported" is wrong; one saying
    "new implementations should not adopt sampling" is right.

!!! danger "Protocol can't enforce consent"
    The spec says plainly that MCP **cannot enforce** its security principles at the protocol
    level. Consent, confirmation and access control are **implementor** responsibilities,
    mostly the **host's**. Options claiming "the protocol prevents the server from…" are
    usually wrong.

## The reflexes that earn the most points

1. **Server needs input mid-request** → `InputRequiredResult`, client **retries** with
   `inputResponses` and `requestState`, **new id**.
2. **Secrets or credentials from the user** → **URL mode** elicitation, never form mode.
3. **Token not issued for this server** → **reject it**. Never pass it through.
4. **Local single-user integration** → **stdio**; credentials from the **environment**, not OAuth.
5. **Remote multi-user server** → **Streamable HTTP** + **OAuth 2.1** with PKCE and resource indicators.
6. **Server needs state across calls** → **explicit handle** as a tool argument, bound
   server-side to the authenticated user.
7. **Tool failed because of bad input the model can fix** → result with **`isError: true`**,
   not a JSON-RPC error.
