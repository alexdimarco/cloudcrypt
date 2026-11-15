# CloudCrypt

CloudCrypt encrypts a folder before your sync service uploads it. You work with ordinary files on your own computers, and the design aims for the service (Nextcloud, Dropbox, OneDrive, iCloud Drive, Google Drive, Syncthing) to receive only encrypted files. That aim is untested, and it relies on the setup recognising which folders your sync client uploads (design section 13, claim C5).

**Status: design stage. There is no code yet. Do not rely on it to protect real data.**

## How it will work

- A small background program, the *agent*, runs for each user on each computer.
- It keeps an ordinary folder, by default `~/CloudCrypt/<name>`, in two-way step with an encrypted *vault folder* that your sync service already carries. You use the ordinary folder with Finder, Explorer or any app.
- Files in the vault folder are written once and never edited, and each is named by a hash of its contents, so by design two devices never overwrite each other (design section 13, claim C3). When two devices edit the same file, both versions are kept.
- No driver, no administrator rights and no mounted drive. Linux, macOS and Windows.
- A command-line tool and an optional setup page talk to the agent over a private local connection. The agent itself runs no web server; the setup page is a separate helper.

## What it does not protect

- The ordinary folder is unencrypted on each of your computers, and the whole setup needs roughly twice the disk space of your files at the start. This trade-off still awaits the owner's decision (design section 0, item 7).
- The sync service can still see that a vault exists, roughly how much changed and when, and the sizes of the encrypted files. It cannot read your files' contents, names or folder structure (design sections 3 and 6.7).
- Out of scope: a device that is compromised while unlocked, sharing between people, selective sync, and taking back data a lost device already holds (design section 3).

## Read the design

- [The design](docs/design-alpha-architecture.md): start with section 0 and the "In pictures" section.
- [Its pre-code review](docs/review-alpha-architecture-predesign.md): verdict "go, with conditions".

Both documents were drafted by AI agents (Claude) at the owner's direction, and each records its own sources and what it did not check.

## Plan

First three short experiments and some owner decisions, then six milestones to a usable alpha (design section 12).

## License

Copyright 2026 DiMarco Technology Solutions Inc

[Mozilla Public License 2.0](LICENSE). In short, and the license text governs: CloudCrypt can be built into software under any license, but anyone who distributes it must make the source of CloudCrypt's own files, with their changes, available under this license.
