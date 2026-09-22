# Guarded wallet mutation writes

This fork preserves nv-websocket-client 2.14 at upstream commit
`3168d4e1f034450b41278f5f73f2c9faeaa5beef`. Its complete 58-file Java source
matched the Maven Central 2.14 sources before this change. The Gradle source
module has version `2.14-fearless-guard1`; no binary publication is implied.

`sendTextGuarded` queues one complete uncompressed text message. Queue acceptance
is not authorization or a successful network send. On the writer thread, the
library checks the connection, rejects any negotiated compression extension,
serializes and masks immutable frame bytes, invokes preparation listeners, and
flushes preceding ordinary output. Only then does it invoke `WebSocketWriteGuard`.
The action atomically commits the message and directly writes and flushes the
underlying socket output. Guarded bytes never enter the WebSocket buffered output
and are not replayed after any writer failure, stop, cancellation, or reconnect.

An authorizer must synchronously invoke its action at most once on the calling
writer thread, never retain it, propagate write failures, and never reenter or wait
on writer-dependent work. It must acquire its authority lock and resample the
clock after any blocking persistence before invoking the action. No application
completion callback runs under this authority or the writer monitor. Cancellation
uses atomic request state; failure callbacks run after those critical sections.

A denied or cancelled queued request reports one failure and writes no bytes.
Cancellation cannot recall a committed write. I/O failure after commitment reports
`mayHaveBeenWritten=true`; callers must reconcile status and must not replay.
The irreversible boundary is underlying `OutputStream.write`, including the
platform socket/TLS implementation. Once accepted there, expiry or revocation
cannot retract bytes. A blocking socket output holds the synchronous authority
until it returns; the API does not claim an application-level completion deadline.

Guarded writes reject compression before invoking any compressor, even though
2.14's built-in outgoing compressor is stateless. This protects compatibility with
stateful/custom extensions. Ordinary `sendText`/frame APIs retain their existing
buffering, fragmentation, compression and delivery behavior.

Run `./gradlew --dependency-verification=strict test`. Tests exercise the real
WritingThread and output stream with queue expiry/revocation, stop during an
authority wait, cancellation, pre-flush delay, partial write/flush failure,
unknown outcomes, immutable frames, compression rejection, cross-thread misuse,
and callback reentry. Two additional tests perform a real loopback WebSocket handshake and verify
authorized output and revocation during preparation followed by legacy traffic.
This is not a claim of TLS, real-device or production-funded validation.
