# Offline focus repaint control-stream evidence

These asciinema v2 recordings capture real output bytes from an offline
Codewhale TUI in an 80×24 PTY, with synchronized output off. They belong on a
separate evidence branch, outside the code pull request.

## Replay

Download this directory and replay each recording in a terminal with asciinema:

```sh
asciinema play baseline.cast
asciinema play fixed.cast
```

The recording header declares the actual initial PTY dimensions. Every output
event uses its observed time and original output byte slice, decoded as UTF-8
with replacement on invalid sequences. No intermediate frames are simulated.
The recordings include startup, the measured focus transitions, an input
liveness check, and shutdown. The three ordinary FocusIn reports occur at
approximately 6.50, 8.31, and 10.13 seconds in `baseline.cast`, and 6.15, 7.96,
and 9.78 seconds in `fixed.cast`.

## Inputs and isolation

The driver starts with a clear environment and a newly created HOME, XDG
directories, workspace, and temporary directory. It disables model-catalog
fetches, update checks, local Ollama discovery, and animations. No provider is
configured or connected, and no host conversation or host configuration is
loaded.

The interaction is scripted from the repository's existing offline launch-card
PTY helper:

1. Clear the initial composer, paste `seed`, and press Enter to open the provider
   picker.
2. Press Ctrl+O to explore offline, then Enter at the ready screen.
3. Clear the composer, paste `start the session`, and press Enter. This stays
   offline and produces a queue receipt.
4. Clear the composer, run `/queue drop 1`, and clear it again.
5. After an idle control interval, send three pairs of raw terminal focus
   reports: FocusOut (`ESC [ O`) followed by FocusIn (`ESC [ I`), waiting for the
   UI to settle after each report.
6. Paste `focus probe alive` to verify input still works, then clear it without
   submitting. Exit with Ctrl+C.

The only terminal replies are responses to actual CPR, DA, keyboard-flag, or
color queries. There are no model responses in the recordings. All visible
message text listed above is synthetic test input.

## Results and scope

`baseline.cast` comes from clean main `c4195348f8b50bcee5a99b0b5e513ef144342957`.
Each of its three ordinary FocusIn events emits one CSI 2J whole-screen erase;
FocusOut and idle intervals emit none. The input liveness check succeeds and the
child exits with code 0. `fixed.cast` comes from the focus-recapture change in
`50d8d18da216e6f29d5c9bb897a426f126674b98`, based on that main revision. Each
of its three ordinary FocusIn events emits zero CSI 2J and zero CSI 3J. Input
remains live and the child exits with code 0.

The broader local matrix captures three ordinary FocusIn events in each case:

| PTY dimensions | Synchronized output | Baseline CSI 2J per FocusIn | Fixed CSI 2J per FocusIn |
|---|---|---|---|
| 80×24 | off | 1, 1, 1 | 0, 0, 0 |
| 80×24 | on | 1, 1, 1 | 0, 0, 0 |
| 120×40 | off | 1, 1, 1 | 0, 0, 0 |
| 120×40 | on | 1, 1, 1 | 0, 0, 0 |

All cases complete without capture errors, preserve live input, and exit with
code 0. Each measured FocusIn contains zero synchronized-output begin/end
pairs with the setting off, or one balanced pair with it on. Startup has its
own terminal capability exchange, so these counts describe the measured live
phases rather than the entire recording.

The comparison concerns redundant whole-screen erasure on ordinary focus
transitions. The change retains a full repaint: focus return resets scroll
margins and origin, restores terminal modes, and repaints the drawable viewport,
including blank cells. It removes the preceding erase from that recovery path;
it does not replace recovery with an incremental diff. Resize, suspend/resume,
and external-terminal handoffs retain clear-and-redraw recovery. A pending clear
also takes precedence over ordinary focus recapture.

Full repaint output remains approximately 4.1 KB at 80 columns and 7.5 KB at
120 columns. Temporary per-cell repaint directives can also cause the next
ordinary diff to write another frame; this occurs at later FocusOut reports
and the input check in the default fixed runs, without any whole-screen erase.
The evidence does not establish a reduction in repaint byte volume. The
measured idle intervals emit no output.

A separate VTE-path control uses `VTE_VERSION=6800` without TMUX: a resize while
unfocused emits no frame, and returning focus applies the pending resize and
clear. Both baseline and fixed runs emit zero bytes while the resize is
deferred, then 7469 bytes including two CSI 2J sequences when focus returns at
120×40. The required resize clear is preserved; input remains live and both
children exit with code 0. This separate resize control is not included in
the two 80×24 recordings.

These are PTY control-stream recordings. They do not validate GNOME Terminal's
native rendering, compositor occlusion behavior, GPU presentation, or a visible
flash on a particular terminal. Replaying the same bytes in another terminal
does not establish that native visual result.

## Artifact checksums

The copies below are unchanged from the captured cast files. Their complete
visible output and scripted input provenance were reviewed for host paths,
configuration, credentials, and personal messages. The workspace basename and
resume identifier in each recording belong to its newly generated synthetic
session.

```text
c868ada4115fa38a6f997c55092e9145be309c79423372ea468c5e72793aa912  baseline.cast
8e6dfe458ef92bcb900ec7a54e579e8b1324d8af055390a547dbd6372d96d66f  fixed.cast
```
