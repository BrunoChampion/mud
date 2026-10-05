# Siri–iPhone–Linux Audio Relay: Minimal Validation MVP

## Objective

Determine whether an iPhone can start a continuous voice connection to a Linux server through Siri, keep it working with the phone locked and pocketed, and disconnect through Siri. Validate on the user's iPhone 16e, installed iOS version, and actual earphones before developing the full product.

The eventual product will let a server-side AI agent operate a browser and GUI on Linux. This MVP tests only the iPhone restrictions and audio connection. It requires no AI model, Linux desktop, WhatsApp integration, glasses, or additional phone.

## Required user experience

After initial installation, permissions, and configuration:

1. With the phone locked and pocketed, say “Hey Siri, start relay in [app name].”
2. The app connects to Linux, activates the earphone microphone, and plays “Connected.”
3. Speak and hear the server return the speech as a delayed echo.
4. Leave the connection active during periods of silence and resume speaking without touching the phone.
5. Say “Hey Siri, stop relay in [app name].” Recording and the server connection stop.

Register explicit Start and Stop App Shortcuts. Merely opening the app is not evidence that Siri can start recording or establish the session. Exact Siri phrasing must be verified on the device.

## Minimum implementation

### Native iPhone app

- Swift/SwiftUI app with a server address, Start button, Stop button, and connection status.
- App Intents/App Shortcuts exposing the same Start and Stop operations as the buttons.
- Microphone permission requested during initial foreground setup.
- Audio session configured for simultaneous recording and playback, background audio enabled, and correct Bluetooth/wired earphone routing.
- One continuous authenticated WebRTC audio session with Linux; the native app owns the session after the Siri action finishes.
- Audio interruption and route-change handling; bounded reconnection attempts after a temporary network loss.
- Explicit states: disconnected, connecting, connected, interrupted, reconnecting, and failed. Start is idempotent; Stop cancels retries and releases the microphone.
- If using AudioRecordingIntent, implement its required Live Activity and recording indication. Do not assume this permits background microphone startup.

### Linux echo service

- Single-user signaling endpoint and WebRTC audio receiver on the existing VPS.
- Return received speech after a short delay, using headphones to avoid feedback.
- Log session identifiers, connection events, and timestamps/counts of audio packets received and returned. During silence, monitor transport health separately: silence suppression can mean no audio packets even when the connection is healthy.
- Keep no permanent audio recordings. Use authenticated, encrypted communication and avoid exposing an unauthenticated relay.

No desktop or browser is required on Linux for this test. A functioning server echo proves both outbound microphone transmission and inbound playback.

## Test matrix

Run on a physical device, with Siri allowed while locked. Record the iOS version, app build, earphone model, network, and exact Siri phrase. Separate first-time permission setup from ordinary use.

| Test | Procedure | Evidence required |
| --- | --- | --- |
| Foreground baseline | Start using the app button; speak and stop | Server receives speech; echo plays through earphones; Stop releases microphone |
| Siri while unlocked | Start and stop using Siri | Shortcut executes the intended action and the audio session starts/stops |
| Locked startup | Lock the phone with app inactive; invoke Siri Start | Audio starts without unlock, screen interaction, or manual launch; otherwise record the exact prompt/error |
| App lifecycle | Repeat locked startup with app suspended, after reboot and first unlock, and after force-quitting | Report each state separately; do not assume success in one proves all states |
| Background endurance | Start, lock, and keep session active for 30 minutes; speak periodically | Both audio directions continue with phone pocketed |
| Silence | Within endurance test, remain silent for at least 10 minutes, then speak | Connection remains usable without restarting the app |
| Locked stop | Invoke Siri Stop during the active locked session | Microphone releases, session closes, and reconnection does not restart it |
| Interruptions | Invoke Siri for another request; receive a call; disconnect/reconnect earphones | Recovery behavior and any required user intervention are documented |
| Network recovery | Switch Wi-Fi/cellular; briefly disable connectivity and restore it | Session recovers or gives an audible failure notice; retry duration is logged |

Repeat locked Start/Stop five times. Cross-check client events with server logs; a local “Connected” label alone is insufficient evidence.

## Decision gates

- **Full intended experience validated:** locked Siri startup and stop succeed repeatedly, audio survives the endurance/silence tests, and interruptions/network changes recover without screen interaction.
- **Conditional feasibility:** audio survives after foreground startup, but locked startup or recovery requires unlocking. This supports a different workflow; it does not meet the user's original hands-free requirement. Obtain agreement before developing around that limitation.
- **Blocked:** no legitimate way to sustain the requested session, or startup consistently requires interaction unacceptable to the user. Stop product development and report the exact restriction.

Do not use silent playback, fake calls, or unrelated background modes to keep the process alive. A bare persistent network socket is not a general entitlement to run indefinitely in the background.

## Known restrictions and unresolved question

Apple supports ongoing background audio, but an Apple DTS engineer states that ordinary audio APIs cannot activate recording from the background. Communication frameworks such as CallKit provide exceptions for genuine communication apps. Whether a particular AI relay qualifies must be established; do not treat those frameworks as a guaranteed workaround or add them until the basic test identifies a blocker.

Install the prototype directly with Xcode/development signing. This validates runtime behavior, not future App Store approval. Real background audio and recording consent/indication remain necessary.

## Deliverables

Provide the installable development build, runnable echo service, short setup instructions, and a test report containing Pass/Fail/Needs Unlock for every test, relevant logs, and the recommended go/no-go decision. Do not implement the full agent until the decisive locked-startup test has been completed.

## Apple references

- [App Shortcuts and Siri actions](https://developer.apple.com/documentation/appintents/acceleratingappinteractionswithappintents)
- [Background recording and playback](https://developer.apple.com/documentation/avfaudio/avaudiosession/category-swift.struct/playandrecord)
- [AudioRecordingIntent and required Live Activity](https://developer.apple.com/documentation/appintents/audiorecordingintent)
- [Apple DTS explanation of background microphone restrictions](https://developer.apple.com/forums/thread/816408)
- [App Review Guidelines: background services and recording consent](https://developer.apple.com/app-store/review/guidelines/)
