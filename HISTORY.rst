Release History
===============

dev
---

Bugfixes
~~~~~~~~

- Correctly reject all of the connection-specific headers mentioned in RFC 7540
  § 8.1.2.2, not just the ``Connection:`` header.

2.1.1 (2016-02-05)
------------------

Bugfixes
~~~~~~~~

- Added debug representations for all events.
- Fixed problems with setup.py that caused trouble on older setuptools/pip
  installs.

2.1.0 (2016-02-02)
------------------

API Changes (Backward-Compatible)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Added new field to ``DataReceived``: ``flow_controlled_length``. This is the
  length of the frame including padded data, allowing users to correctly track
  changes to the flow control window.
- Defined new ``UnsupportedFrameError``, thrown when frames that are known to
  hyperframe but not supported by hyper-h2 are received. For
  backward-compatibility reasons, this is a ``ProtocolError`` *and* a
  ``KeyError``.

Bugfixes
~~~~~~~~

- Hyper-h2 now correctly accounts for padding when maintaining flow control
  windows.
- Resolved a bug where hyper-h2 would mistakenly apply
  SETTINGS_INITIAL_WINDOW_SIZE to the connection flow control window in
  addition to the stream-level flow control windows.
- Invalid Content-Length headers now throw ``ProtocolError`` exceptions and
  correctly tear the connection down, instead of leaving the connection in an
  indeterminate state.
- Invalid header blocks now throw ``ProtocolError``, rather than a grab bag of
  possible other exceptions.

2.0.0 (2016-01-25)
------------------

API Changes (Breaking)
~~~~~~~~~~~~~~~~~~~~~~

- Attempts to open streams with invalid stream IDs, either by the remote peer
  or by the user, are now rejected as a ``ProtocolError``. Previously these
  were allowed, and would cause remote peers to error.
- Receiving frames that have invalid padding now causes the connection to be
  terminated with a ``ProtocolError`` being raised. Previously these passed
  undetected.
- Settings values set by both the user and the remote peer are now validated
  when they're set. If they're invalid, a new ``InvalidSettingsValueError`` is
  raised and, if set by the remote peer, a connection error is signaled.
  Previously, it was possible to set invalid values. These would either be
  caught when building frames, or would be allowed to stand.
- Settings changes no longer require user action to be acknowledged: hyper-h2
  acknowledges them automatically. This moves the location where some
  exceptions may be thrown, and also causes the ``acknowledge_settings`` method
  to be removed from the public API.
- Removed a number of methods on the ``H2Connection`` object from the public,
  semantically versioned API, by renaming them to have leading underscores.
  Specifically, removed:

    - ``get_stream_by_id``
    - ``get_or_create_stream``
    - ``begin_new_stream``
    - ``receive_frame``
    - ``acknowledge_settings``

- Added full support for receiving CONTINUATION frames, including policing
  logic about when and how they are received. Previously, receiving
  CONTINUATION frames was not supported and would throw exceptions.
- All public API functions on ``H2Connection`` except for ``receive_data`` no
  longer return lists of events, because these lists were always empty. Events
  are now only raised by ``receive_data``.
- Calls to ``increment_flow_control_window`` with out of range values now raise
  ``ValueError`` exceptions. Previously they would be allowed, or would cause
  errors when serializing frames.

API Changes (Backward-Compatible)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Added ``PriorityUpdated`` event for signaling priority changes.
- Added ``get_next_available_stream_id`` function.
- Receiving DATA frames on streams not in the OPEN or HALF_CLOSED_LOCAL states
  now causes a stream reset, rather than a connection reset. The error is now
  also classified as a ``StreamClosedError``, rather than a more generic
  ``ProtocolError``.
- Receiving HEADERS or PUSH_PROMISE frames in the HALF_CLOSED_REMOTE state now
  causes a stream reset, rather than a connection reset.
- Receiving frames that violate the max frame size now causes connection errors
  with error code FRAME_SIZE_ERROR, not a generic PROTOCOL_ERROR. This
  condition now also raises a ``FrameTooLargeError``, a new subclass of
  ``ProtocolError``.
- Made ``NoSuchStreamError`` a subclass of ``ProtocolError``.
- The ``StreamReset`` event is now also fired whenever a protocol error from
  the remote peer forces a stream to close early. This is only fired once.
- The ``StreamReset`` event now carries a flag, ``remote_reset``, that is set
  to ``True`` in all cases where ``StreamReset`` would previously have fired
  (e.g. when the remote peer sent a RST_STREAM), and is set to ``False`` when
  it fires because the remote peer made a protocol error.
- Hyper-h2 now rejects attempts by peers to increment a flow control window by
  zero bytes.
- Hyper-h2 now rejects peers sending header blocks that are ill-formed for a
  number of reasons as set out in RFC 7540 Section 8.1.2.
- Attempting to send non-PRIORITY frames on closed streams now raises
  ``StreamClosedError``.
- Remote peers attempting to increase the flow control window beyond
  ``2**31 - 1``, either by window increment or by settings frame, are now
  rejected as ``ProtocolError``.
- Local attempts to increase the flow control window beyond ``2**31 - 1`` by
  window increment are now rejected as ``ProtocolError``.
- The bytes that represent individual settings are now available in
  ``h2.settings``, instead of needing users to import them from hyperframe.

Bugfixes
~~~~~~~~

- RFC 7540 requires that a separate minimum stream ID be used for inbound and
  outbound streams. Hyper-h2 now obeys this requirement.
- Hyper-h2 now does a better job of reporting the last stream ID it has
  partially handled when terminating connections.
- Fixed an error in the arguments of ``StreamIDTooLowError``.
- Prevent ``ValueError`` leaking from Hyperframe.
- Prevent ``struct.error`` and ``InvalidFrameError`` leaking from Hyperframe.

1.1.1 (2015-11-17)
------------------

Bugfixes
~~~~~~~~

- Forcibly lowercase all header names to improve compatibility with
  implementations that demand lower-case header names.

1.1.0 (2015-10-28)
------------------

API Changes (Backward-Compatible)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Added a new ``ConnectionTerminated`` event, which fires when GOAWAY frames
  are received.
- Added a subclass of ``NoSuchStreamError``, called ``StreamClosedError``, that
  fires when actions are taken on a stream that is closed and has had its state
  flushed from the system.
- Added ``StreamIDTooLowError``, raised when the user or the remote peer
  attempts to create a stream with an ID lower than one previously used in the
  dialog. Inherits from ``ValueError`` for backward-compatibility reasons.

Bugfixes
~~~~~~~~

- Do not throw ``ProtocolError`` when attempting to send multiple GOAWAY
  frames on one connection.
- We no longer forcefully change the decoder table size when settings changes
  are ACKed, instead waiting for remote acknowledgement of the change.
- Improve the performance of checking whether a stream is open.
- We now attempt to lazily garbage collect closed streams, to avoid having the
  state hang around indefinitely, leaking memory.
- Avoid further per-stream allocations, leading to substantial performance
  improvements when many short-lived streams are used.

1.0.0 (2015-10-15)
------------------

- First production release!
