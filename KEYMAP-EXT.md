# Extending LinearMouse's Sequence Capabilities

Here is a breakdown of why your current implementation is not working, along with the missing files and the concrete fix.

---

### Why "Command + Left-Click" is failing to open links

In [`ButtonActionExecutor.swift`](file:///Users/andrewgorbet/mybin/proj/exec/linearmouse/LinearMouse/EventTransformer/ButtonActionExecutor.swift), you updated `postClickEvent` to assign `mouseDownEvent.flags = .maskCommand` and post both mouse-down and mouse-up events immediately.

There are three reasons macOS applications and web browsers (Chrome, Safari, Firefox, Arc) ignore or drop this sequence:

1. **Browsers rely on keyboard state events (`flagsChanged`), not just mouse event flags**:
   Web browsers track whether modifier keys (Command, Shift, Option, Control) are active via keyboard event streams and system modifier state tables (`NSEvent.modifierFlags` and `CGEventSourceFlagsState`). Simply stamping `.maskCommand` onto a raw `CGEvent` without simulating the physical key-down event for `Command` causes the browser's link click handler to see that no Command key is currently held.
2. **Missing Device-Specific Modifier Flags**:
   macOS distinguishes between generic modifier flags (`.maskCommand` / `NX_COMMANDMASK`) and device-specific bits (`NX_DEVICELCMDKEYMASK`). When Cocoa / WebKit converts `CGEvent` to `NSEvent`, it often ignores generic flags if the device-specific mask is absent.
3. **Zero delay between Mouse Down and Mouse Up**:
   Posting `mouseDownEvent.post(...)` immediately followed by `mouseUpEvent.post(...)` in the exact same microsecond often causes WebKit and Chromium event loops to drop or misidentify the click (hit testing and DOM dispatch require a tick between down and up).

---

### Files You Modified vs Files You May Have Missed

You already edited:

- [`LinearMouse/Model/Configuration/Scheme/Buttons/Mapping/Action.swift`](file:///Users/andrewgorbet/mybin/proj/exec/linearmouse/LinearMouse/Model/Configuration/Scheme/Buttons/Mapping/Action.swift)
- [`LinearMouse/Model/Configuration/Scheme/Buttons/Mapping/Action+CustomStringConvertible.swift`](file:///Users/andrewgorbet/mybin/proj/exec/linearmouse/LinearMouse/Model/Configuration/Scheme/Buttons/Mapping/Action+CustomStringConvertible.swift)
- [`LinearMouse/EventTransformer/ButtonActionExecutor.swift`](file:///Users/andrewgorbet/mybin/proj/exec/linearmouse/LinearMouse/EventTransformer/ButtonActionExecutor.swift)
- [`LinearMouse/Localizable.xcstrings`](file:///Users/andrewgorbet/mybin/proj/exec/linearmouse/LinearMouse/Localizable.xcstrings)

**File you also need to update for the UI Picker:**

- [`LinearMouse/UI/ButtonsSettings/ButtonMappingsSection/ButtonMappingAction/ButtonMappingActionPicker.swift`](file:///Users/andrewgorbet/mybin/proj/exec/linearmouse/LinearMouse/UI/ButtonsSettings/ButtonMappingsSection/ButtonMappingAction/ButtonMappingActionPicker.swift#L101-L109):
  Add your new action types so they appear in the UI dropdown:
  ```swift
  .section("Mouse Button") { [
      .actionType(.arg0(.mouseButtonLeft)),
      .actionType(.arg0(.mouseButtonLeftDouble)),
      .actionType(.arg0(.mouseButtonLeftCommand)),
      .actionType(.arg0(.mouseButtonLeftShift)),
      .actionType(.arg0(.mouseButtonMiddle)),
      .actionType(.arg0(.mouseButtonRight)),
      .actionType(.arg0(.mouseButtonBack)),
      .actionType(.arg0(.mouseButtonForward))
  ]
  ```

---

### How to Fix the Action Execution in [`ButtonActionExecutor.swift`](file:///Users/andrewgorbet/mybin/proj/exec/linearmouse/LinearMouse/EventTransformer/ButtonActionExecutor.swift)

Instead of only tagging the CGEvent flags on an instantaneous click, use the built-in `keySimulator` to:

1. Press the modifier key down (`keySimulator.down(...)`).
2. Post the mouse down event with both generic and device-specific flags.
3. Wait ~15ms (a single frame) using `DispatchQueue.main.asyncAfter`.
4. Post the mouse up event.
5. Release the modifier key (`keySimulator.up(...)`).

Here is the implementation:

```swift
// In ButtonActionExecutor.swift

        switch action {
        case .arg0(.mouseButtonLeftCommand):
            postModifierClickEvent(mouseButton: .left, modifierKeys: [.command])

        case .arg0(.mouseButtonLeftShift):
            postModifierClickEvent(mouseButton: .left, modifierKeys: [.shift])

        // ...
```

And add the helper method:

```swift
    private func postModifierClickEvent(
        mouseButton: CGMouseButton,
        modifierKeys: [Key]
    ) {
        guard let location = CGEvent(source: nil)?.location else {
            return
        }

        let source = CGEventSource(stateID: .hidSystemState)

        // 1. Press modifier keys so system and browser register modifier state
        try? keySimulator.down(keys: modifierKeys, tap: .cgSessionEventTap)

        // 2. Build complete modifier flags (generic + device specific)
        var flags = CGEventFlags()
        for key in modifierKeys {
            switch key {
            case .command, .commandRight:
                flags.insert([.maskCommand, CGEventFlags(rawValue: UInt64(NX_DEVICELCMDKEYMASK))])
            case .shift, .shiftRight:
                flags.insert([.maskShift, CGEventFlags(rawValue: UInt64(NX_DEVICELSHIFTKEYMASK))])
            case .option, .optionRight:
                flags.insert([.maskAlternate, CGEventFlags(rawValue: UInt64(NX_DEVICELALTKEYMASK))])
            case .control, .controlRight:
                flags.insert([.maskControl, CGEventFlags(rawValue: UInt64(NX_DEVICELCTLKEYMASK))])
            default:
                break
            }
        }

        // 3. Post Mouse Down
        guard let mouseDownEvent = CGEvent(
            mouseEventSource: source,
            mouseType: mouseButton.fixedCGEventType(of: .leftMouseDown),
            mouseCursorPosition: location,
            mouseButton: mouseButton
        ) else {
            try? keySimulator.up(keys: modifierKeys.reversed(), tap: .cgSessionEventTap)
            return
        }
        mouseDownEvent.flags = flags
        mouseDownEvent.isLinearMouseSyntheticEvent = true
        mouseDownEvent.post(tap: .cgSessionEventTap)

        // 4. Post Mouse Up after a short delay (15ms), then release modifier keys
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.015) { [self] in
            guard let mouseUpEvent = CGEvent(
                mouseEventSource: source,
                mouseType: mouseButton.fixedCGEventType(of: .leftMouseUp),
                mouseCursorPosition: location,
                mouseButton: mouseButton
            ) else {
                try? keySimulator.up(keys: modifierKeys.reversed(), tap: .cgSessionEventTap)
                return
            }
            mouseUpEvent.flags = flags
            mouseUpEvent.isLinearMouseSyntheticEvent = true
            mouseUpEvent.post(tap: .cgSessionEventTap)

            DispatchQueue.main.asyncAfter(deadline: .now() + 0.015) { [self] in
                try? keySimulator.up(keys: modifierKeys.reversed(), tap: .cgSessionEventTap)
                resetKeySimulatorIfNothingIsHeld()
            }
        }
    }
```

---

### Important Logitech G502X Note

If you are mapping one of the G502X's extra buttons (e.g. G4, G5, G7, G8, DPI shift):

- If **Logitech G HUB** is running, G HUB may intercept and swallow or remap these buttons before macOS or LinearMouse's event tap receives them.
- To ensure LinearMouse receives the physical clicks, open **Logitech G HUB**, switch the mouse to **On-Board Memory Mode**, or assign the target buttons in G HUB to generic mouse buttons (e.g., Forward/Back / Generic Mouse Button 4/5).

Would you like me to apply these code updates to [`ButtonActionExecutor.swift`](file:///Users/andrewgorbet/mybin/proj/exec/linearmouse/LinearMouse/EventTransformer/ButtonActionExecutor.swift) and [`ButtonMappingActionPicker.swift`](file:///Users/andrewgorbet/mybin/proj/exec/linearmouse/LinearMouse/UI/ButtonsSettings/ButtonMappingsSection/ButtonMappingAction/ButtonMappingActionPicker.swift) for you?
