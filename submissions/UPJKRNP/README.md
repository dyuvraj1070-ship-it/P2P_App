# Group Study Room: Pankaj Singh's README

**Group:** UPJKRNP
**Live app:** https://dyuvraj1070-ship-it.github.io/P2P_App/submissions/UPJKRNP/

A serverless study room with **chat**, a **shared notes pad** and **file drops**. All user data goes browser-to-browser over WebRTC. There is no backend and no database.

**This README is written from my side (Pankaj, Shared Notes).** My part is explained in detail. My teammates' parts are summarised so you can see the whole project.

---

## Team

| Member | SAP ID | GitHub | Area |
|---|---|---|---|
| **Pankaj Singh (me)** | 590028972 | Notbing00 | **Shared notes** |
| Vipransh Sharma | 590022121 | vipransh-upes | P2P mesh and connections |
| Arnav Verma | 590027671 | lackofbraincellsverma | File transfer |
| Yuvraj Dhiman | 590025374 | dyuvraj1070-ship-it | UI and styling |

---

## What the whole team built (at a glance)

### Vipransh: P2P mesh and connections
- Each peer claims the lowest free slot ID (`studyroom-<room>-<0..9>`) on the public PeerJS broker, then dials every other slot to form a full mesh. No host, so the room survives anyone leaving (max 10 people).
- STUN plus TURN fallback (including TURN over TCP/443) so peers can connect across different networks.
- Fix for two peers dialling each other at the same time: both sides keep the link dialled by the lower slot.
- Join and leave notices, and a hint after 20 seconds if nobody connects.

### Arnav: File transfer
- Only file metadata is broadcast. The owner streams the file in 16 KB chunks when someone clicks **Get**.
- Back-pressure on the data channel, a progress bar, and reassembly into a downloadable Blob.
- Files over 25 MB stream straight to disk on Chrome/Edge. Hard cap is 100 MB.
- Chunks are accepted only from the file's owner, with size and index checked.

### Yuvraj: UI and styling
- Responsive layout (tabs on phones), light/dark theme toggle, status indicator.
- Avatar stack of who is online, chat bubbles, unread badge, one-click invite link.
- File cards with type icons and progress bars.

### Shared work (check commit history for who did exactly what)
- Chat with unique message IDs (de-duplication) and Lamport-clock ordering, plus history sync for late joiners.
- Optional room password (PBKDF2, hashed into the peer IDs).
- Input validation on incoming messages (type and size checks).

---

## My part in detail: Shared Notes (Pankaj)

One document that everyone in the room edits live, with no server in between.

| # | What I built | How it works |
|---|---|---|
| 1 | **Notes panel** | A `<textarea id="notes">` in its own tab. What one person types shows on every device. |
| 2 | **Debounced broadcast (120 ms)** | The app waits 120 ms after the last keystroke, then sends one `notes` message. No message on every key press. |
| 3 | **Lamport clock** | Each edit carries `{ text, clock, by }`. The clock goes up on every local edit and jumps to `max(local, remote)` on every remote one, so all devices agree on "newer" without trusting device time. |
| 4 | **Conflict resolution (`mergeNotes`)** | A remote update wins only if its clock is higher. On a tie, the higher slot ID wins. Every device picks the same winner, so the notes converge. |
| 5 | **Caret stays in place** | When a remote edit arrives, I find the part of the text that changed. If your cursor is after that part, I shift it by the length difference. If it is before, it stays put. |
| 6 | **Late joiners get the notes** | The current notes are sent in the `hello` message on connect, and merged with the same `mergeNotes`. |
| 7 | **Survives refresh** | Notes are saved to `localStorage` per room (`saveState` and `loadState`) and restored when you rejoin. |
| 8 | **Input validation** | `cleanNotes()` accepts only text up to 200,000 characters, a finite clock and an integer slot. Anything else is dropped. |
| 9 | **"Last edit by" label** | The notes header shows who edited last, or "you edited just now". |

### Problem I solved: the cursor jumped while typing
- **Problem:** While I typed in the notes, a remote update threw my cursor to the end of the text.
- **Why it happened:** Setting a textarea's `value` from code resets the caret. Every remote update did that.
- **What I tried:** First I saved the caret position and restored the same number after the update. That is not enough, because the number drifts when the remote change adds or removes text before the caret.
- **Final fix:** Compare old and new text, find where they differ, and shift the caret by `new length - old length` when the change is before it.

### Limitation of my part
Notes use last-writer-wins on the **whole document**. If two people type within the same ~120 ms window, one change overwrites the other. Fixing this properly needs character-level merging (CRDT or OT), which we did not build.

---

## Problems the team faced

| # | Problem | Fix | Who |
|---|---|---|---|
| 1 | Join/leave notices vanished when a new message arrived | Notices kept in a separate `events` list, merged by time on every redraw | Team |
| 2 | Two users with the same name saw each other's messages as "You" | Random per-tab client ID on every message | Team |
| 3 | A peer could send huge or malformed data | Type and size checks on every incoming message | Team |
| 4 | Two peers connected to each other twice | Keep the link dialled by the lower slot | Vipransh |
| 5 | Cursor jumped in shared notes | Diff old and new text, shift the caret | **Pankaj** |
| 6 | Wrong password silently gave an empty room | 20-second hint to check code and password | Team |

---

## How to use
1. Open the live link on two or more devices.
2. Enter a name and the same room code (and the same password, if the room has one), or open a link with `#room=CODE`.
3. Chat, edit the notes together, or drop files. Click **Get** to pull a file from its owner.

## How to test my part
1. Open the same room on two devices and go to the **Notes** tab.
2. Type on one. The text shows on the other, with "last edit by <name>".
3. Keep your cursor mid-sentence on device A, then type at the start on device B. A's cursor should not jump.
4. Refresh one device. The notes come back.
5. Join from a third device late. It gets the current notes straight away.

## Libraries and services
- [PeerJS](https://peerjs.com/) (MIT) for WebRTC
- PeerJS Cloud broker for signalling only
- Google and Twilio public STUN, Open Relay (Metered) free TURN
- Browser APIs: WebRTC data channels, Web Crypto (PBKDF2), `localStorage`, File System Access

## Limitations
- Files live only in the owner's browser, so the owner must stay online.
- Max 10 people per room and 100 MB per file.
- Display names are not unique or verified.
- No rate limiting.
- Notes are last-writer-wins on the whole document.
- Depends on free public services that can be slow or down.

## Screenshots
Add images to `screenshots/` and keep these names, or edit the links.

![Join screen](screenshots/01-join.png)
![Connected room with shared notes](screenshots/03-notes.png)
![Notes edited by two people](screenshots/03b-notes-two-people.png)
