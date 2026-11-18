---
name: android-device
description: Interact with local Android device capabilities from inside OpenCode, including notifications, reminder alarms, toast messages, clipboard access, URL opening, device information, and workspace file bridge actions (list/share/save/pick/capture/events/backup).
---

# Device Actions

Use this skill when the user asks OpenCode to interact with local device capabilities, including notifications, reminder alarms, toast messages, clipboard access, URL opening, and device information. In the in-app chat, treat the current environment as the device by default; natural requests such as "send a notification" or "remind me in one minute" should use this skill.

Decision: use this skill instead of MCP for the local device capability bridge. The app already exposes a loopback HTTP server inside the foreground service, and OpenCode can use this skill from the bundled Ubuntu environment with only Python stdlib. This avoids adding another MCP server registration/lifecycle layer while still giving OpenCode callable local device actions.

The app provides an authenticated loopback server whose port and access token are injected into the OpenCode process. Always use the helper script instead of calling the endpoint directly:

```bash
python3 "$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py" health
python3 "$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py" tools
python3 "$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py" device-info
python3 "$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py" notify --title "OpenCode" --message "Task completed"
python3 "$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py" alarm --delay-seconds 60 --title "Reminder" --message "Check the task"
python3 "$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py" toast --message "OpenCode is running"
python3 "$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py" clipboard-set --text "Copied from OpenCode"
python3 "$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py" clipboard-get
python3 "$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py" open-url --url "https://example.com"
python3 "$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py" file-list --path .
python3 "$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py" file-share --path report.md
python3 "$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py" file-save --path report.md --destination downloads
python3 "$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py" file-pick --target-dir inbox
python3 "$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py" file-pick --target-dir inbox --mime "image/*"
python3 "$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py" file-capture --target-dir inbox
python3 "$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py" file-list --path . --recursive
python3 "$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py" file-diagnostics
python3 "$OPENCODE_CONFIG_HOME/skills/android-device/android_device.py" file-backup
```

Available actions:

- `health`: verify the local capability server is reachable.
- `tools`: list server-side device actions.
- `device-info`: read basic device version, model, and battery information.
- `notify`: show a system notification. Requires `--title` and `--message`.
- `alarm`: schedule a notification after `--delay-seconds`. Optional `--title` and `--message` customize the alarm notification.
- `toast`: show a short toast. Requires `--message`.
- `clipboard-set`: set clipboard plain text. Requires `--text`.
- `clipboard-get`: read clipboard plain text.
- `open-url`: open `http`, `https`, `mailto`, or `tel` URL with the device default handler. Requires `--url`.
- `file-list`: list files and folders inside the current OpenCode project. Optional `--path` is relative to the current project; optional `--recursive` includes every nested file with its relative path.
- `file-share`: share a file from the current OpenCode project through the Android share sheet. Requires `--path` (project-relative).
- `file-save`: save a file from the current OpenCode project to the device. `--destination downloads` saves directly to the Downloads folder; `--destination pick` opens the system save dialog. Requires `--path`; optional `--name` renames the exported file.
- `file-pick`: open the system file picker and import the selected files into the current OpenCode project. Optional `--target-dir` selects the destination folder (default: project root); optional `--mime` filters the picker (for example `image/*`, `application/pdf`). The command waits until the user finishes picking or cancels (up to 3 minutes).
- `file-capture`: open the camera app and save the photo directly into the current OpenCode project. Optional `--target-dir` selects the destination folder (default: project root); optional `--name` sets the file name. The command waits until the user captures or cancels (up to 3 minutes).
- `file-diagnostics`: check current-project writability, live event-delivery support, configured Android folder links, and free device storage.
- `file-backup`: create a ZIP backup of the current OpenCode project and save it to the device Downloads folder.

Paths in this skill are relative to the current OpenCode project (for example `/root/chat` or `/root/workspace`). Files shared into OpenCode from other Android apps land in `inbox/` automatically.

If `notify` or `alarm` fails with a permission error, ask the user to grant notification permission to the app in system settings.
