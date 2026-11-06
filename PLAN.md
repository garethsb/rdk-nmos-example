<!--
SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# Rivermax Dev Kit + NvNmos C API Example Plan

This document is the execution specification for a standalone example that
combines the [Rivermax Dev Kit](https://github.com/NVIDIA/rivermax-dev-kit)
(RDK) data plane with the [NvNmos](https://github.com/NVIDIA/nvnmos) C API
control plane. Complete stages in order. A stage is complete only after all of
its review criteria pass.

## Outcome

The repository will provide one application, `rdk-nmos-node`, which:

- hosts an NMOS Node in-process with `libnvnmos`;
- offers an SMPTE ST 2110-20 video Sender, an ST 2110-30 audio Sender and an
  ST 2110-40 ancillary data Sender, each using RDK's `MediaSenderApp`;
- offers matching video, audio and ancillary data Receivers, each using RDK's
  `MediaReceiverApp`;
- supports SMPTE ST 2022-7 for every Sender and Receiver when started with two
  interface addresses;
- generates a moving video test pattern, a 1 kHz audio tone, and timecode, AFD
  and closed caption ancillary data;
- on each Receiver, either reports statistics only or also writes received
  media to files;
- applies IS-05 activations, including disable and re-enable, to each Sender
  and Receiver independently, without restarting the process.

Two instances of `rdk-nmos-node` on one host or on two hosts, connected through
their IS-05 APIs, form the end-to-end test.

Licensing follows the REUSE Specification and is checked with
`fsfe/reuse-action`. Vendored text such as the Apache-2.0 licence is copied
byte-for-byte, never retyped.

## Constraints

- C++17 application code using the public C API of NvNmos and the public C++
  API of RDK.
- No GStreamer, `nvnmosd`, MXL, IS-08 or display.
- The application does not depacketize RTP itself. Frame reconstruction on the
  receive side belongs in RDK (see [Required RDK Changes](#required-rdk-changes)).
- RDK changes are made on a branch of a fork of `NVIDIA/rivermax-dev-kit`,
  based on its `dev` branch, and are offered upstream as pull requests. This
  repository pins a commit on that branch until the changes are available
  upstream, then pins an upstream commit.
- Pin compatible dependency revisions. Do not build moving default branches.
- The Rivermax SDK and its licence are prerequisites installed by the user.
  This repository does not download or redistribute them.
- Code that does not depend on Rivermax, such as SDP mapping and test pattern
  generation, builds and is unit tested without the Rivermax SDK, so that CI
  can run it on hosted runners.

## Current RDK Behaviour

This section records the RDK behaviour the design depends on, as of the public
`dev` branch at `51ea8f1`.

- `MediaSenderApp` is configured with a `MediaSenderSettings` struct through
  `MediaSenderUserProvidedSettingsBuilder`, then started with `initialize()`
  and `run()`. One instance can send video, audio and ancillary data.
- RDK's video, audio and ancillary settings calculators generate the SDP for
  each stream from those settings. With two local IPs and
  `enable_redundancy`, the SDP has one media description per path and an
  `a=group:DUP` attribute.
- `MediaSenderApp::set_media_essence_sources()` accepts an application
  `IMediaEssenceSource` per stream. `AncillaryEssenceSource` already generates
  AFD, timecode and closed caption data.
- `MediaSenderSettings` has one `source_port` for all paths.
- `MediaReceiverApp` accepts an `IMediaEssenceSink` per stream, but does not
  yet reconstruct frames. It is excluded from the default build on `dev`. It
  supports one path per stream.
- `IPOReceiverApp` merges two or more paths per stream at the RTP level. It
  takes one sender IP per path in `source_ips`.
- Every send and receive loop exits only when the process-wide flag returned
  by `SignalHandler::get_received_signal()` is set. No app has its own stop.
- Every `BaseApp` constructor installs RDK's default SIGINT handler, which sets
  that flag.
- `RivermaxDevKitFacade::initialize_rivermax()` returns a shared handle.
  Rivermax is released when the last handle is destroyed.
- `MediaSenderApp`, `MediaReceiverApp` and `IPOReceiverApp` derive
  destination addresses for their streams by incrementing the last octet of
  the configured address per stream. With one stream per app, the configured
  address is used unchanged.
- The generated SDP for each media type includes `a=ts-refclk` and
  `a=mediaclk`. The video and audio calculators set `a=ts-refclk` from the
  configured reference clock. The ancillary data calculator does not, so its
  SDP always advertises a traceable PTP reference clock.

## Required RDK Changes

### Changes Made by This Project

Each change is a separate pull request against `NVIDIA/rivermax-dev-kit`
`dev`, with RDK unit tests where RDK has a test target for the area changed.
They are listed in the order the stages need them.

1. **Reference clock in ancillary data SDP.** Set `a=ts-refclk` in the
   ST 2110-40 SDP from the configured reference clock, as the video and audio
   calculators do. NvNmos derives the Node clock from `a=ts-refclk`, so all
   Senders must advertise the same clock.
2. **Per-path source port.** Allow a source port per path in
   `MediaSenderSettings`. Until this change lands, the application rejects an
   activation whose legs have different source ports.
3. **Frame reconstruction in `MediaReceiverApp`.** Complete the RTP media
   packet buffer reader and `MediaReconstructionDataConsumer`, so that
   `MediaReceiverApp` delivers complete ST 2110-20 frames or fields,
   ST 2110-30 sample blocks and ST 2110-40 ancillary data packets to the
   `IMediaEssenceSink`. Re-enable `media_receiver` and `rdk_media_receiver` in
   the default build.
4. **ST 2022-7 in `MediaReceiverApp`.** Accept two paths per stream, using the
   same hardware path merging as `IPOReceiverApp`.
5. **Paths without a source filter.** Allow a multipath receiver to join a
   group without a sender IP on a path, for any-source multicast.

Changes 1 and 2 are independent. Changes 4 and 5 build on change 3. Work on
all of them can start alongside Stage 1, and the RDK pin moves forward as each
lands on the fork branch.

### Changes That May Be on the RDK Roadmap

A per-app stop may be on the RDK roadmap: a way to ask one app, or one of its
I/O nodes, to stop, and to wait for it to finish, without setting the
process-wide signal flag. The Sender and Receiver stages need such a stop. If
it reaches the public `dev` branch, this project uses it as RDK provides it.
If it does not reach `dev` before the Sender stage, this project adds it as a
further change, with the same shape: a per-app stop flag that every send and
receive loop and essence source checks together with the process-wide signal
flag, and a wait that returns when the app's threads have finished.

## Architecture

### Dependencies

- `libnvnmos` is consumed as the installed `nvnmos::nvnmos` CMake target.
- RDK is built from source with CMake `FetchContent` at the pinned commit, as
  RDK's README describes for external applications.
- libsdptransform, which RDK already uses, parses SDP in this application.
- `scripts/bootstrap.sh` builds and installs the pinned `libnvnmos` into
  `.prefix`. `NVNMOS_SRC` and `RDK_SRC` select local source trees instead.

### Components

- **Configuration.** Hand-written command-line parsing, as in the NvNmos
  example. The options are the HTTP port, Node seed, one or two interface
  addresses, which Senders and Receivers to offer, whether Senders activate at
  start-up, CPU cores, the Receiver output mode and the output directory.
- **SDP mapping.** Pure functions with no Rivermax dependency. They convert
  between SDP and a plain description of one stream: media type and format,
  and for each leg the interface address, destination address and port, source
  address and port, and whether the leg is enabled. They also add and read the
  NvNmos SDP attributes.
- **RDK adapter.** Converts a stream description into `MediaSenderSettings` or
  `MediaReceiverSettings`, and extracts the SDP that RDK generates.
- **Stream.** One per NMOS Sender or Receiver. It owns at most one RDK app and
  the thread that runs it, and has one function to apply a requested state.
- **Media sources.** An `IMediaEssenceSource` for the video test pattern and
  one for the audio tone. Ancillary data uses RDK's `AncillaryEssenceSource`.
- **Media sinks.** An `IMediaEssenceSink` that counts media units and checks
  timestamps and payload, and optionally writes payload to one file per
  Receiver.
- **Node.** Creates the NvNmos Node server, adds Senders and Receivers,
  dispatches activation callbacks to streams, and shuts down in order.

### Media Formats

The first version uses fixed formats:

| Media | Format |
| --- | --- |
| Video | 1920x1080 progressive, 50 fps, YCbCr 4:2:2, 10-bit, ST 2110-21 type N |
| Audio | 48 kHz, L24, 2 channels, 1 ms packet time |
| Ancillary data | Timecode, AFD and CEA-608 closed captions, one packet group per frame |

The video pattern is colour bars with a moving element, so that a viewer can
see frames advance. Pattern and tone generation have deterministic unit tests.

### Identity

Each instance uses one Node seed. Senders and Receivers use the caller-chosen
names `video`, `audio` and `data`, so NvNmos derives stable NMOS UUIDs from the
seed and those names.

### Configuring Transport Files

For each Sender, the application builds `MediaSenderSettings`, takes the SDP
that RDK generates, and adds:

- session level: `a=x-nvnmos-name:<name>`;
- each media description: `a=x-nvnmos-iface-ip:<interface address>` and
  `a=x-nvnmos-src-port:<source port>`.

For each Receiver, the application builds an SDP for the fixed format with
`a=x-nvnmos-name`, and one media description per interface address with
`a=x-nvnmos-iface-ip`. With two interface addresses it includes
`a=group:DUP`. NvNmos supports ST 2022-7 with one media description per leg
(separate destination addresses), not one media description with two source
filters.

Receivers are constrained to the fixed formats. NvNmos publishes Receiver Caps
derived from the configuring SDP, so a controller can see which Senders are
compatible before connecting. An activation whose SDP has a different format
fails. An unconstrained Receiver, marked with `a=x-nvnmos-caps`, would publish
no Receiver Caps, and an unsupported format would then only be reported when an
activation fails. Receivers can become unconstrained once `MediaReceiverApp`
reconstructs a known set of formats, because each activation already builds a
new app from the active SDP.

### Activation Lifecycle

Senders and Receivers start inactive, as IS-05 requires.

The NvNmos `connection_activated` callback looks up the stream by side and
name, and calls its apply-state function with the active SDP, or with null
for deactivation. The apply-state function:

1. parses the SDP into a stream description, and returns failure if the format
   is not supported or the legs are inconsistent;
2. returns success without change if the description equals the running one;
3. asks the running app to stop, waits for its thread to finish, and destroys
   the app;
4. if every leg is disabled, or the SDP is null, returns success;
5. otherwise builds new settings for the enabled legs, constructs and
   initializes a new app, sets its media source or sink, runs it on a new
   thread, and returns the result.

Because the callback returns after step 5, an immediate IS-05 activation
reports the result to the controller.

With start-up activation selected, the Node calls the apply-state function for
each Sender with its configuring SDP, then calls `nmos_connection_activate`.
That call updates the Node and Connection APIs without invoking the activation
callback, so the stream is not started twice.

If an app fails after activation, the stream logs the failure and keeps its
NMOS state.

### Process Lifecycle

At start-up the Node calls `RivermaxDevKitFacade::initialize_rivermax()` once
and keeps the handle until exit, so that destroying the last app between
activations does not release Rivermax. SIGINT uses RDK's default handler.
The main thread waits for the process-wide signal flag, which also stops every
app. It then joins every stream thread, destroys the NvNmos Node server, and
releases the Rivermax handle.

## Stage 0: Repository Skeleton

Create the Apache-2.0 licensed repository, ignore generated output, document
the project purpose, and install this plan as the source of truth.

Review:

- the repository is on `main`;
- all committed text and source files carry valid SPDX headers where
  appropriate, and `reuse lint` passes;
- the initial signed-off commit leaves a clean working tree.

## Stage 1: Bootstrap, CMake and CI Foundation

- CMake consumes the installed `nvnmos::nvnmos` target, and builds RDK with
  `FetchContent` at the pinned commit when the Rivermax SDK is found.
- Without the Rivermax SDK, CMake builds only the targets that do not need it.
- `scripts/bootstrap.sh` installs the pinned `libnvnmos` into `.prefix`.
- An `rdk-nmos-node` stub links both libraries and reports versions and help.
- GitHub Actions runs the REUSE check, `shellcheck`, the bootstrap, and the
  build and unit tests of the targets that do not need Rivermax.

Review:

- configure and build work against installed prefixes, with and without the
  Rivermax SDK;
- a missing dependency fails with an actionable message;
- CI passes.

## Stage 2: SDP Mapping

Implement the SDP mapping for all three media types, with one and two legs.

Review:

- unit tests cover Sender and Receiver configuring SDP for each media type,
  with one and two legs;
- unit tests parse representative active SDP from NvNmos: unicast, multicast
  with and without `a=source-filter`, one leg disabled with `a=inactive`, and
  unsupported formats;
- round trips through the stream description preserve every field used;
- the tests run in CI without the Rivermax SDK.

## Stage 3: Media Sources and Sinks

Implement the video test pattern and audio tone generators, and the payload
checks and file writing used by the media sinks, as code with no Rivermax
dependency. Wrap them in `IMediaEssenceSource` and `IMediaEssenceSink`
implementations in the targets that build with RDK.

Review:

- unit tests check the generated pattern and tone deterministically;
- unit tests check that the payload checks accept generated media and reject
  corrupted media;
- the tests run in CI without the Rivermax SDK.

## Stage 4: Node and Transport Files

Add the NvNmos Node, and add the three Senders and three Receivers with their
configuring SDP. In this stage the activation callback parses the active SDP
into a stream description, logs it, and returns the result of parsing, without
starting a data plane. Sender SDP comes from RDK's settings calculators.

Review:

- Node and Connection APIs respond on the configured HTTP port;
- the Senders' `/transportfile` matches RDK's generated SDP plus the NvNmos
  attributes, with one and two interface addresses;
- each Receiver publishes Receiver Caps for its fixed format;
- IS-05 activations of every Sender and Receiver, including disable and
  re-enable, are parsed and logged, and an activation whose SDP has a
  different format fails;
- SIGINT exits cleanly.

## Stage 5: Senders

Requires the per-app stop and RDK change 1.

Add the Sender data plane: the RDK adapter for `MediaSenderSettings`, the
streams for Senders, and the activation lifecycle for Senders.

Review:

- each Sender's stream is received by an RDK receiver app or a packet capture
  at the active destination;
- disabling one Sender stops only that stream, and re-enabling resumes it;
- changing a Sender's destination through IS-05 moves the stream;
- with two interface addresses, both legs carry the stream, and disabling one
  leg stops only that leg;
- repeated disable and re-enable cycles do not leak memory or crash;
- SIGINT exits cleanly.

## Stage 6: Receivers

Requires the per-app stop and RDK changes 3 and 4.

Add the Receiver data plane: the RDK adapter for `MediaReceiverSettings`, the
streams for Receivers, and the activation lifecycle for Receivers.

Review:

- connecting each Receiver to the matching Sender of another instance starts
  reception, and the counters increase at the expected rate;
- the payload checks recognise the video pattern, the audio tone and the
  ancillary data;
- with file output selected, the files contain the received media;
- with two legs, reception continues without loss when either leg is disabled
  at the Sender;
- disabling one Receiver stops only that stream, and re-enabling resumes it;
- an activation whose SDP has a different format fails, and the Receiver keeps
  its previous state;
- repeated disable and re-enable cycles do not leak memory or crash;
- SIGINT exits cleanly.

## Stage 7: Connection Helper and Documentation

Add `scripts/connect.sh` to read one instance's Sender state and PATCH the
matching Receivers of another instance. Add `scripts/smoke.sh` to start two
instances, connect them, exercise disable and re-enable on every Sender and
Receiver, check progress, and stop both instances by PID.

Replace the provisional README with prerequisites, bootstrap, build, two
instance use, connection, ST 2022-7 use and troubleshooting.

Review:

- a user following only the README receives media on all three Receivers;
- `shellcheck` passes;
- cleanup succeeds after passing and failing smoke runs.

## Stage 8: Release-Quality Build and CI

Review:

- bootstrap is idempotent, and pinned revisions and required system packages
  are documented in one place;
- the release build treats this repository's warnings as errors;
- README commands match CI commands;
- no build artifacts, downloaded dependencies or credentials are tracked;
- all CI-equivalent local checks pass before the final signed-off commit.

## Stop Conditions

Stop and report, rather than work around, when:

- an RDK change cannot be made without changing RDK behaviour that other RDK
  apps rely on;
- the Rivermax SDK version required by the pinned RDK commit is not available;
- a review criterion cannot be checked on the available hardware.

## Open Questions

- Whether a stream whose app fails after activation should report itself
  disabled with `nmos_connection_activate`, rather than only logging.
