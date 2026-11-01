# Windows computer-use for AI agents: four problems when an agent and a human share one desktop

> **Field notes · partly day one.** Section 1 (minimized windows), 2 (typing leaks), 3 (keyboard layouts) and 4 (Thai OCR) were built and tested on **2026-10-30**. The rest of the tool has been in daily use since 2026-10-07. Tested on one machine: Windows 11 build 26300, Python 3 + `ctypes`, 125 % DPI, apps from classic Win32 to UWP and Chromium. Expect rough edges; this README will be updated after two more weeks of real use.

I run several AI coding agents (Claude Code) on one Windows laptop, and they need to click and type in desktop apps **while I keep working on the same screen**. I built a small private MCP server for that. Before adding more to it I compared it with the two open-source tools most people use:

| | [trycua/cua](https://github.com/trycua/cua) (cua-driver, Windows backend) | [CursorTouch/Windows-MCP](https://github.com/CursorTouch/Windows-MCP) |
|---|---|---|
| 1. Minimized target window | Refuses: `cannot capture minimized window` (`platform-windows/src/capture.rs`, `wgc.rs`) | Restores it with `SW_RESTORE` and switches to it (`desktop/service.py`) — the user's screen changes |
| 2. Typing leaks | **Handled.** Foreground `type_text` waits for each character to be read: 211 characters into a window slowed to 40 ms per key all landed, none leaked | Not tested |
| 3. Non-US keyboard layout | **Fixed after 0.38.0.** In 0.38.0, `press_key "3"` typed `-` on a Thai-layout window, both background and foreground ([trycua/cua#6181](https://github.com/trycua/cua/issues/6181)). Fixed in [trycua/cua#6201](https://github.com/trycua/cua/pull/6201): the nightly build of 2026-11-01 types `3`, `1` and `a` correctly in both modes. Not in a stable release yet | Not tested |
| 4. Thai OCR | OCR extension is English-only ([trycua/cua#4332](https://github.com/trycua/cua/issues/4332)); not tested by me | Not tested |

cua rows 1–3 were re-tested against cua-driver 0.38.0 on 2026-10-30 with throwaway WinForms windows, and row 3 again against `0.38.1-nightly.20261101` on 2026-11-02; row 4 is from the linked issue. So sections 2 and 3 are mainly traps for hand-rolled `SendInput` code; cua avoids both once the fix ships.

Both are good projects with far more scope than mine (Cua runs whole sandboxed VMs on three OSes). These notes are only about four details that matter when **an agent and a human share one desktop**. The code here is short illustrative `ctypes`; the tool itself stays private.

---

## 1. Act on a minimized window without the user seeing it

Agents are usually asked to work in an app the user has minimized. Restoring it steals the screen; refusing it means the user has to babysit. UWP apps unload their UI while minimized (their UIA tree is just the frame; File Explorer also needed the trick below in my tests), and a minimized window has no rendered content to capture.

What works: make the window fully transparent, restore it **without activation**, do the work, minimize it again.

```python
GWL_EXSTYLE, WS_EX_LAYERED, LWA_ALPHA = -20, 0x80000, 0x2
SW_SHOWNOACTIVATE, SW_SHOWMINNOACTIVE = 4, 7

ex = GetWindowLongPtrW(hwnd, GWL_EXSTYLE)
if ex & WS_EX_LAYERED:
    raise RuntimeError("window manages its own transparency; restore it normally")
saved = WINDOWPLACEMENT(); saved.length = sizeof(saved)
GetWindowPlacement(hwnd, byref(saved))            # keeps WPF_RESTORETOMAXIMIZED

SetWindowLongPtrW(hwnd, GWL_EXSTYLE, ex | WS_EX_LAYERED)
SetLayeredWindowAttributes(hwnd, 0, 0, LWA_ALPHA)  # alpha 0: invisible, still rendered

wp = WINDOWPLACEMENT.from_buffer_copy(saved); wp.flags = 0; wp.showCmd = SW_SHOWNOACTIVATE
SetWindowPlacement(hwnd, byref(wp))                # restore, no activation
# UWP apps rebuild their UI after restore: wait ~0.5 s before reading UIA
try:
    ...                                            # UIA reads, Invoke, PrintWindow, OCR
finally:
    saved.showCmd = SW_SHOWMINNOACTIVE
    SetWindowPlacement(hwnd, byref(saved))         # back to minimized, maximized-on-restore kept
    SetWindowLongPtrW(hwnd, GWL_EXSTYLE, ex)
```

Things I had to learn the hard way:

- **Cloaking doesn't work cross-process.** `DwmSetWindowAttribute(DWMWA_CLOAK)` on another process's window returns `E_ACCESSDENIED`. Layered alpha 0 is the available substitute.
- **Restore can still activate.** Some apps grab the foreground on restore despite `SW_SHOWNOACTIVATE`. Remember the foreground window before, and hand focus straight back if it moved.
- **Save the placement, not just "was minimized".** Re-minimizing with `ShowWindow` alone forgets that the window was maximized before it was minimized; the next time the user clicks it on the taskbar it comes back small. Writing the saved `WINDOWPLACEMENT` back keeps the flag.
- **The user may click the window mid-peek.** A watcher thread (50 ms) checks whether the window became foreground without us causing it. If so, drop the transparency at once and **don't** re-minimize: the user wants it open.
- **Never send real keystrokes into an invisible window.** If a step needs real input (anything UIA patterns can't do), make the window visible first. Background UIA patterns (`Invoke`, `Value`, `Toggle`) are fine while it is transparent.
- **Skip windows that are already layered.** Their own alpha would be overwritten.
- **Only one peek per window at a time,** and don't hold the peek lock while the caller works (a code review found this could deadlock a background job against a real-input step on another thread).

Checked by comparing screen pixels before and during a peek (identical), while `PrintWindow` still returned the full window image. Elevated (admin) windows are out of reach of a non-elevated process (UIPI), so this is only tested on normal windows.

## 2. Typed text leaking into the user's window

(cua-driver 0.38.0 does not have this problem: its foreground typing waits for each character. This section is for anyone writing their own.)

The obvious approach: borrow the foreground, `SendInput` the text as `KEYEVENTF_UNICODE` events, give the foreground back. On a slow app the tail of the text arrived in **my terminal** instead.

`SendInput` only queues input. It returns before the target thread has read the keys, and the keys go to whichever window is foreground *when they are read*. If you restore the user's window first, the rest of the queue follows it there.

What fixed it:

1. **Type in chunks of 8 and read back.** After each chunk, read the field (UIA `ValuePattern` or `WM_GETTEXT`) and wait until the chunk is there before sending more, and before handing focus back.
2. **Compare against the value before typing**, not "does the field contain the text". A field that already held the same text made the first version pass too early.
3. **Drain before restoring focus.** A few `SendMessageTimeout(hwnd, WM_NULL, …)` round trips until they come back instantly: the target thread is idle, so its queue is empty.
4. **Check the foreground before every character** and stop if it changed.
5. **Stop when the human types.** `GetLastInputInfo` also counts your own injected keys, so it can't tell you apart from the user by itself. Tag every `SendInput` event with your own `dwExtraInfo` value and run a low-level keyboard hook: an event that is not `LLKHF_INJECTED` *with your tag* is someone else. Input from someone else in the last ~0.3 s stops typing, and a key from them during an action puts a warning in the result. Two traps: prime the idle tracker before your first injected key (otherwise that key looks like the user), and arm the hook *before* you take focus.
6. **Don't press Enter if the readback never matched.** Report "unverified" instead.

Often you don't need real keys at all: for a field that already has text, `ValuePattern.SetValue(old + new)` plus a readback appends in the background. Real keys are the fallback, not the default.

## 3. Non-Latin keyboard layouts turn digits into other characters

(cua-driver 0.38.0 has this bug; [trycua/cua#6201](https://github.com/trycua/cua/pull/6201) fixes it with the approach below, and the 2026-11-01 nightly passed the same test. This section is for anyone writing their own.)

With the Thai (Kedmanee) layout active, an agent pressing the key "3" typed "-" in the app. Virtual-key codes describe a physical key; what character it produces depends on the layout of the **target's thread**, and the user can switch layouts at any time.

Sending everything as Unicode fixes text but breaks single-key shortcuts some apps listen for as virtual keys. What I settled on, per printable character with no modifiers:

```python
hkl = GetKeyboardLayout(GetWindowThreadProcessId(fg, None))
r = VkKeyScanExW(ch, hkl)
vk, shift_state = r & 0xFF, (r >> 8) & 0xFF
if r == -1 or shift_state or 0x60 <= vk <= 0x6F:
    send_unicode(ch)        # not on this layout, needs shift, or a numpad key
else:
    send_vk(vk)             # the layout's own unshifted key for this char
```

The numpad exclusion matters. `VkKeyScanExW` on the Thai layout:

| char | US layout | Thai layout |
|---|---|---|
| `3` | `0x33` (top-row 3) | `0x63` (`VK_NUMPAD3`) |
| `-` | `0xBD` | `0x33` (top-row 3) |

On Thai the top-row digit keys type Thai characters, so the only unshifted key that produces "3" is on the numpad, and with NumLock off it arrives as Page Down. Also note row two: sending VK `0x33` "for 3" types "-" there, which is exactly the bug I started with. Chords like `ctrl+s` stay on virtual keys, which are layout-independent for shortcuts.

## 4. Reading Thai text on screen

For custom-drawn apps with no useful UIA tree (a Qt chat app, canvas UIs), the fallback is OCR on a window capture and clicking the matched text. The built-in `Windows.Media.Ocr` offers no Thai language, so I compared two local engines on the same 8 real UI lines (Thai + English mixed, from screenshots of apps I use):

| Engine | Exact lines | Character error rate | RAM | Time |
|---|---|---|---|---|
| tesseract.js 7.0.0, `tessdata_fast` tha+eng | 6 / 8 | 14 % | ~110 MB | 0.6 s |
| RapidOCR, PP-OCRv5 Thai | 3 / 8 | 41 % | 266 MB | 0.75 s |

Small sample, one machine; treat it as a hint, not a benchmark. I kept tesseract.js in a persistent Node worker (spawned once, exits after 10 min idle) and removed RapidOCR. `tessdata_best` does not load in tesseract.js (it needs the `DotProductSSE` path), so use `tessdata_fast` and pin its SHA-256.

Details that made matching reliable:

- **Match whole lines, not words.** Tesseract splits Thai (no spaces between words) into syllable-sized "words", so a word-level search for a Thai phrase never matches.
- **Normalize ำ.** OCR often returns nikhahit + sara aa (`ํ` + `า`) instead of the single sara am (`ำ`). Normalize both sides before comparing.
- **Fuzzy match, but keep every digit.** A similarity threshold (~0.85) absorbs vowel confusions such as บัญชิ for บัญชี, but on numbered lists it happily matched "20. …" for "27. …". Require all digits of the query to match exactly.
- **Big captures: crop bands yourself.** For tall windows I split into up to 3 overlapping bands (48 px overlap) and OCR them in parallel. Passing `rectangle` to tesseract.js returned garbage for some bands; cropping the image first worked. Assign each line to the band whose non-overlapping core contains the line's centre, so cut lines are never duplicated or lost. A 28-line window went from 2.2 s to 1.0 s at the same accuracy.
- **Grow workers on demand.** Starting with 3 workers cost ~185 MB; starting with 1 and adding workers only when bands queue keeps the idle cost near 100 MB.

---

## Not covered here

The tool also drives elevated (admin) windows through a separate helper the user approves once with UAC. I'm deliberately not publishing how that helper works.

## License

Text and snippets: MIT. See [LICENSE](LICENSE).
