# Design: CloudCrypt alpha architecture

BUILD STATUS: proposed (revision 3); pre-code review docs/review-alpha-architecture-predesign.md verdict GO_WITH_CONDITIONS; 15 conditions, of which revision 2 applied 12 fully and revision 3 completes conditions 7, 9 and 14, re-verified in that review's addenda; owner rulings A to D and the decisions in section 14 are open; build not started. This design supersedes docs/design-phase-r-architecture.md, whose premise that existing vaults and users must be carried forward the owner corrected on 2026-11-14.

*(Note beside the original, 2026-11-14, by the main Claude session: at the owner's request, a section "In pictures" was added between sections 0 and 1. It restates the design and changes no design text, so it is not a revision. The file's sha256 therefore no longer matches the revision-3 hash, 165e5ab9a8d1c7cc3ae67a5a3ae996ac46c853ef0d588f535c9f08e731e08751, that the pre-code review's addendum 2 records.)*

*(Note beside the original, 2026-11-14, by the main Claude session: revision 4, the public edition, made at the owner's request when the project moved to its own public repository, https://github.com/alexdimarco/cloudcrypt. It renames the product from its earlier working name to CloudCrypt throughout: its commands, folders and settings, its file types, magic values and key-derivation labels (for example `cloudcrypt`, `~/CloudCrypt/<name>`, `.cck`, `CCKY` and `"cc0/"`). The six pictures were redrawn to fit the longer name. Nothing else changed: every decision, condition and open ruling stands as in revision 3, and BUILD STATUS above still describes the review state. The sha256 values recorded in this file and in the pre-code review identify the private originals under the old name. This repository has no code yet: "today's code", "the old tree", its line counts, version 0.17.0 and the tag `legacy/0.17.0` describe the private predecessor, as do citations marked [repo] or [review] (so README.md:3, DESIGN.md, PROMPT.md and CLAUDE.md in those citations are the predecessor's files, not this repository's) and the other documents named here that are not in this repository; none of them is published. Names inside those citations were renamed too, so `.cloudcrypt` in section 10 (the predecessor's hidden metadata folder) and `cmd/cloudcrypt` in section 9's deleted list are the predecessor's, not this design's. Files this design plans for this repository, such as `docs/format-spec.md` and `docs/dependencies.md`, do not exist yet. Section 9, M1 in section 12, and decisions 8 and 13 and Ruling B in section 14 still plan the rewrite inside the predecessor's repository; after the move they are open for the owner to restate.)*

*(Note beside the original, 2026-11-14, by the main Claude session: revision 5, made at the owner's request. Section 9 and M1 in section 12 are restated for this repository, which starts empty: nothing is tagged or deleted here, the predecessor's code comes here only under section 9's porting rule, CI runs on GitHub's hosted runners, and the process rules move into this repository's own files. The earlier text of both, which planned the rewrite inside the predecessor's repository, is kept in the private original; for section 9 and M1 this supersedes the revision 4 note's last sentence and its mention of section 9's deleted list. Short notes beside section 0 item 6, Gate 0, decisions 8 and 13 and Ruling B say how the restatement bears on them; those decisions stay the owner's. At the owner's request the predecessor's commit identifier is withheld (marked [withheld]). BUILD STATUS still describes the review, which saw revision 3, not revisions 4 and 5. Under section 12's gates the restatement owes a 10-lens delta check of sections 9 and 12 before M1 code [judgement]; this session has not run one and does not certify the restatement. Sources: revision 4 of this design; the predecessor's license file and repository rules, opened read-only and not published; GitHub's billing documentation, seen through a web search on 2026-11-14 without opening the page.)*

Owner's statement (2026-11-14, verbatim): "re-assess with the understanding that this tool is not in use but still pre alpha. I want to make sure the design is clean and purpose driven and futer supportable. Managing encrypted files needs to just work well and with low user friction"

Scope source: docs/project-review-2026-11-14.md (the review: 146 findings, 144 confirmed, 13 P0, at commit [withheld], version 0.17.0). Date of this work: 2026-11-14. Authored by one agent after a design panel of four proposals and two judges (section 15).

Labels used throughout. **[repo]** means I opened the file and line. **[review]** means a finding id or section of the review, carried from its text. **[recall]** means a platform or toolchain fact stated from memory that must be checked before anyone relies on it. **[judgement]** means my estimate or opinion. A "term of art" is defined where it first appears. A mark such as (Condition 3) shows text that revision 2 added or changed to apply that numbered condition of the pre-code review; where a condition limits a claim, the claim carries its C-number from section 13 beside it (Condition 14).

## 0. Decision summary

1. **What we build.** A small background program (the "agent") per user that keeps an ordinary folder on the user's disk, the "mirror" (default `~/CloudCrypt/<name>`), in two-way step with an encrypted vault folder that the user's existing sync client (Nextcloud, Dropbox, OneDrive, iCloud Drive, Google Drive, Syncthing) already carries. The user works in the mirror with Finder, Explorer, a Linux file manager and every normal app. This is a full rewrite of format, engine, commands and interface under the same purpose.
2. **Storage model.** The vault folder holds only write-once files with content-hash names: key files, packs of encrypted chunks, and encrypted change-log segments. Nothing is ever edited in place, no two devices write the same name, and the current state is derived by merging records, so the sync client cannot make us overwrite blind and conflicts keep both versions.
3. **Access model.** Mirror folder on Linux, macOS and Windows. No driver, no administrator rights, no browser file manager, no mounted drive in the alpha. A mount can be added later as a second "materializer" over the same engine and format.
4. **Process model.** One agent per user owns every key, every vault folder write and every mirror write. The command line and an optional small setup-and-status page (a separate helper program) are clients of the agent over an owner-only local socket. The core binary does not import `net/http`, so it has no web server to attack.
5. **What users experience.** Install, pick a sync folder from a detected list, choose a password, save a recovery key (typed back once), and a folder appears. Second device: pick the vault, type the password. Daily use is zero steps. Conflicts appear as extra files named for the device and date; nothing is lost.
6. **What survives from today.** About 1,200 of today's 22,683 non-test Go lines (the chunker, portable-name rules, atomic file writer, keychain, hidden-password prompt, recovery-key reading, Host allowlist) and the repository's test discipline. About 21,480 lines are deleted, including the browser GUI, WebDAV, rclone and rsync management, remote profiles and the old format code (section 9). *(Note beside the original, 2026-11-14, by the main Claude session, revision 5: in this repository nothing is deleted. Those 21,480 lines stay in the private predecessor, and the 1,200 come here only under section 9's porting rule, or are rewritten.)*
7. **The trade accepted.** The mirror holds plaintext on every device. Local disk cost is the mirror plus the vault folder plus version history plus the trash (8.6): about twice the live data at the start, growing with history (Condition 6). It is the price of zero drivers; it needs the owner's explicit acceptance (decision 2).
8. **Why.** The review's P0 findings are not seven bugs; they come from one shape: one mutable file per path, written from an in-memory copy, by several front ends with no shared owner (review 5.1 to 5.3). With no users to carry forward, the cheapest fix is to stop having that shape. That removes the vault-folder overwrite P0s by construction; the mirror moves the data-loss risk into the plan function (7.2), where it is bounded by named guards, each conditional (C9) (Condition 14).

## In pictures: what CloudCrypt does and how it works

*(Added 2026-11-14 by the main Claude session at the owner's request. The six pictures restate sections 0 to 8 and decide nothing; where a picture and the numbered text differ, the text wins. Paths and times are examples, Nextcloud stands for any supported sync client, and the numbers are the design's proposals, most marked [judgement] where they appear.)*

**Picture 1. What it does.** Your files stay readable in an ordinary folder on each device; only encrypted files go into the folder your sync client uploads. The picture follows one save in one direction; every device does both.

```text
            LAPTOP                              DESKTOP

  +-------------------------+         +-------------------------+
  | ~/CloudCrypt/Documents  |         | ~/CloudCrypt/Documents  |
  | your files, readable    |         | the same files appear   |
  +-------------------------+         +-------------------------+
               |  you save a file                  ^  it appears
               v                                   |
  +-------------------------+         +-------------------------+
  | CloudCrypt agent        |         | CloudCrypt agent        |
  | encrypts the change     |         | decrypts the change     |
  +-------------------------+         +-------------------------+
               |  new encrypted files              ^  read and merged
               v                                   |
  +-------------------------+         +-------------------------+
  | ~/Nextcloud/CloudCrypt  |         | ~/Nextcloud/CloudCrypt  |
  | encrypted files only    |         | encrypted files only    |
  +-------------------------+         +-------------------------+
               |                                   ^
               |    +-----------------------+      |
               +--->| Nextcloud: your sync  |------+
                    | client and its server |
                    | see only encrypted    |
                    | files                 |
                    +-----------------------+
```

The cost is that every device keeps a readable copy of every file, and CloudCrypt needs about twice the disk space of your files at the start, growing with history. That trade is decision 2, still open (section 14). Nextcloud can still see how many encrypted files there are, their sizes and when they change (6.7). Source: section 0, items 1, 2, 5 and 7.

**Picture 2. The parts on one device.** The numbers in brackets match the notes below.

```text
  +---------------------------+       +-------------------------------+
  | Finder, Explorer, any app |       | cloudcrypt     (command line) |
  +---------------------------+       | cloudcrypt-ui  (setup page)   |
                |                     +-------------------------------+
                |  (1)                               |  (2)
                v                                    v
  +---------------------------+  (3)  +-------------------------------+
  | ~/CloudCrypt/Documents    |<----->| CloudCrypt agent              |
  | the mirror: plain files   |       | one per user, from login;     |
  +---------------------------+       | the only CloudCrypt program   |
                                      | that holds keys or writes     |
                                      | to either folder              |
                                      +-------------------------------+
                                          |  (4)               |  (5)
                                          v                    v
                             +------------------------+ +-------------+
                             | ~/Nextcloud/CloudCrypt | | OS keychain |
                             | encrypted, write-once  | | vault key   |
                             +------------------------+ +-------------+
```

1. You open, edit, save and delete files with any app, as in any folder. Daily use has no CloudCrypt step (J4).
2. The command line and the setup page only send requests to the agent, over a local connection that only your user account can open. The agent opens no network port; the setup page's separate helper listens on 127.0.0.1 only (5.1, 5.4). The page is planned for milestone M5, and page versus tray is Owner Ruling C.
3. The agent scans the mirror (every 30 seconds to 10 minutes when idle, every 2 seconds for files changed in the last minute), decides what to do for each file by the three-way rule of 7.2, and changes the mirror only through one guarded writer (7.2.2).
4. It adds new encrypted files here and never edits one, and it reads the files your other devices add (6.1). Your sync client carries this folder; CloudCrypt never talks to the cloud itself.
5. "Remember on this device" (on by default) keeps the vault key in the operating system's keychain, so you are not asked for the password at each login. The password itself is never stored (5.3).

Source: 5.1 to 5.5.

**Picture 3. Saving a file, and how it reaches your other device.**

```text
  LAPTOP
   1  You save report.docx in ~/CloudCrypt/Documents.
   2  The agent's next scan sees the change. It waits until the
      file has been unchanged for 2 seconds.
   3  It splits the file into chunks and encrypts them. A small
      save goes inside the change record; a large one goes into
      a pack file first.
   4  It writes one change record: "report.docx, new version,
      replacing the version this device last saw".
   5  Nextcloud uploads the new files.
        |
        |   seconds or days later; arrival order does not matter
        v
  DESKTOP
   6  Nextcloud downloads them.
   7  The agent merges the new record with all the others and
      finds that the vault has a newer report.docx.
   8  It writes the new version to a temporary file and checks
      that your copy has not changed (if it has, both versions
      are kept as a conflict). Then it moves the old copy to its
      trash and renames the new one into place.
```

Source: J4, 5.6, 6.4, row 2 of the table in 7.2, and 7.2.2.

**Picture 4. What the vault folder holds.**

```text
  ~/Nextcloud/CloudCrypt/      the vault folder; your sync client
  |                            carries it to your other devices
  |
  +-- keys/<n>-<hash>.cck      the vault key, locked once by your
  |                            password and once by your recovery
  |                            key; a password change adds a new
  |                            key file
  |
  +-- packs/<xx>/<hash>.ccp    encrypted pieces ("chunks") of file
  |                            contents; many files share one pack
  |
  +-- log/<xx>/<hash>.ccj      encrypted change records (new
                               version, move, delete, discard); one
                               file per batch of saves
```

Every file here is encrypted, written once and never edited. Its name is a hash of its own bytes (a short fingerprint computed from the contents), so two devices never write the same name. Each device works out the current state by merging all the change records; the order they arrive in does not matter, and duplicated or renamed copies change nothing. That is why the sync client cannot make one device's save overwrite another's; claim C3 in section 13 says what this depends on. Source: 6.1 to 6.5, with the file names simplified.

**Picture 5. When two devices edit the same file.**

```text
  LAPTOP                            DESKTOP
  edits report.docx at 15:02        edits report.docx at 15:04
  (neither device has seen the other's edit yet)
          |                                 |
          +----------------+----------------+
                           |
                           v
            both edits are kept as versions
                           |
                           v
  ~/CloudCrypt/Documents on every device then shows:

    report.docx                                       <- Desktop's edit
    report (conflict from Laptop, 14 Nov 15:02).docx  <- Laptop's edit
```

The later edit keeps the plain name, and the other is kept beside it, named for its device and time. You open both, keep the one you want and delete the other. CloudCrypt records that choice, so the deleted copy does not come back on your other devices; `cloudcrypt conflicts --keep <file>` does the same in one command. Nothing is merged automatically, and a device with a wrong clock can only change which copy keeps the plain name, never what is kept. Source: J5, 7.1, and rows 19 to 22 of the table in 7.2.

**Picture 6. The checks before your folder changes.**

```text
  Before the agent changes or removes a file in your folder:

  Folder's marker file missing?        -> pauses and asks you
  Files it saw in the vault now gone?  -> changes nothing; reports it
  Would delete over half the files,    -> stops until you confirm
  or over 200 of them?
  Still waiting for records it needs?  -> waits; changes nothing
  A file it has never synced?          -> never deletes or replaces it
  You edited the file since last sync? -> keeps your edit; never
                                          overwrites it
  None of these?                       -> moves the old copy to the
                                          trash (kept up to 30 days),
                                          then writes the new one
```

Source: 7.2 (rows 9 to 16 and the guards below the table), 7.2.1, 6.6 and 8.6; the half-the-files rule applies once the vault holds at least 10 files. These checks are the riskiest new code (milestone M3), and none of it is built or tested yet (claim C9 in section 13).

## 1. Purpose and scope

**What the product does.** It keeps a person's files encrypted on their own devices before the files reach a folder that a sync client carries, so the storage provider sees ciphertext only, and it lets the person work with the files with as little friction as possible. [repo] README.md:3 and DESIGN.md:5 describe the same baseline: encrypted data in a normal directory, carried by a separate sync client, no cloud-provider API. [repo] PROMPT.md:19-20 requires "Plaintext never enters the configured vault directory" and "Virtual paths and file metadata are encrypted"; both survive as hard rules. The September review set the goal as an always-on drive opened in Finder or Explorer [repo] docs/drive-review-2026-09-02.md:11 (the "no mount backend anywhere" paragraph); this design meets the "opened in the normal file manager" half with a folder, and defers the "drive" half (section 5.5).

**In scope for the alpha.** Create a vault inside a sync folder; join it from more devices; a mirror folder that stays in step; conflicts that lose nothing; password change; mandatory recovery key; forgotten-password recovery; lost-device key replacement (design in the alpha, build right after it, section 12); restore of deleted or older versions; status and error messages a non-expert can act on; verify and doctor; start at login; Linux, macOS, Windows.

**Out of scope, and why.**
- A mounted drive (FUSE, WinFsp, macFUSE, NFS loopback, OS file-provider APIs). It needs a driver or permission prompt on two of three systems [recall], and the September design's mount wrote plaintext into a cache anyway ([repo] drive-review:290: "rclone VFS cache (plaintext)"), so it does not buy "no plaintext at rest". It attaches later without a format change (5.5).
- Sharing between people (per-person keys). One person, many devices. The format reserves a slot kind for it.
- Remote profiles, rclone, rsync, SSH keys, remote pull and push, managed program downloads. The sync client is the transport.
- WebDAV and a browser file manager. The operating system's file manager is the file manager.
- Garbage collection (space reclaim) in the first alpha cut; the design is in 7.4 and the format reserves what it needs.

**Cut from today's feature set** (non-test lines from my count script over the working copy, 2026-11-14): rclone and rsync managers, ingest, transports, remotes, SSH keys, dependency probe, importer (about 3,800); `internal/localdav` WebDAV (1,171); `internal/webui` (5,397); `internal/processguard` (193, it stops other programs by name); `internal/vaultmove` and `internal/profile` (438); the 2,411-line CLI becomes fifteen commands (5.1; the one command count used throughout) (Condition 14); scrypt and PBKDF2 password functions (Argon2id only); the seal and unseal ceremony; the content-layout migration; both old metadata-folder names. [repo] PROMPT.md:15 lists a "local WebDAV-compatible endpoint" among the original access modes and PROMPT.md:12 asks for scrypt and PBKDF2; this design drops both. That changes a founding document, so it is an owner decision (decision 6).

## 2. User journeys and the friction budget

**Counting rule.** One step is one thing the person must read, click or type. A decision is a choice that is not just pressing Enter. **Friction budget** (a limit I set; [judgement]): install at most 3 steps; first vault at most 6 steps and 3 decisions; joining a second device at most 4 steps and 2 decisions; daily use 0 steps; a conflict at most 3 steps and 1 decision; forgotten password at most 4 steps; lost device at most 5 steps on one device and 2 on each other; uninstall at most 3 steps. A journey over budget fails the friction review. Counted by this rule, J1 is 4 steps while builds are unsigned, and J2 (typing the password twice, saving the key, ticking the box, typing two groups) is about 9; both are marked conditional below rather than claimed inside the budget, and the M4 friction review decides what to cut (for example, accepting the suggested passphrase removes a decision) (Condition 14).

**J1. Install: 3 steps, 0 decisions once builds are signed; 4 steps in the alpha (Condition 14).** Download, run, open CloudCrypt. No driver, no administrator prompt, no restart. The agent registers itself to start at login for this user only and says so. It is also registered to restart if it stops: launchd `KeepAlive`, systemd `Restart=on-failure`, and a Windows logon task set to restart on failure [recall for all three; to verify]. A second small scheduled job runs `cloudcrypt status --notify` every 15 minutes, so a dead agent still produces an alert (7.5) (Condition 10). Updates in the alpha are a manual reinstall (5.4) (Condition 12). Differences by system: macOS registers a launchd user agent; Windows registers a per-user Run entry or Task Scheduler logon task; Linux writes an XDG autostart entry (a desktop-file standard for login programs), plus an optional systemd user unit [recall for all three]. In the alpha the builds are unsigned, so macOS Gatekeeper and Windows SmartScreen add one warning-dismissal step [recall]; signing is a beta gate (decision 9), and until then the alpha counts the warning dismissal as the fourth step.

**J2. First vault: 6 screens and 3 decisions; about 9 steps by the counting rule, which is over budget and flagged for the M4 friction review (Condition 14).**
1. Choose "Create a vault".
2. Pick the sync folder from a list CloudCrypt builds by looking for known folders and marker files: Nextcloud, Dropbox, OneDrive, iCloud Drive, Google Drive, Syncthing (or Browse) [recall for the markers; M1's detector includes a table of them for the owner to check]. Decision 1: which sync folder. If CloudCrypt does not recognise the sync client, the first-run text says so: the check that keeps the mirror and the recovery-key file out of synced places covers only the clients on the list (C5) (Condition 15).
3. Accept the suggested name ("Documents"); the mirror becomes `~/CloudCrypt/Documents`, outside every sync folder, defaulted not asked.
4. Type a password twice, or accept a suggested passphrase. Decision 2: password. A typed password must be at least 12 characters and not among the 1,000 most common passwords [judgement]; the suggested passphrase meets this by construction (Condition 15).
5. Recovery key screen: the default offer is to print it or write it down. Saving to a file is offered second and goes only through the same placement check as every plaintext write (8.6): the save dialog starts in a local folder outside every synced folder, and the save is refused, with the reason, if the place is the vault folder, the mirror, a detected sync root or an ancestor of one. If the key is copied, the screen warns that clipboard history may keep a copy. Then tick "I stored it" and type two requested groups back. Not skippable, because no server can reset a password. Decision 3: where the recovery key lives. Detection of synced places is best effort (OneDrive can redirect Documents and Desktop), so the screen says where the file went (Condition 6).
6. Done: the mirror opens and the first sync starts. "Remember on this device" and "start at login" are on by default and visible in settings.

On a terminal the same six steps run as `cloudcrypt setup`. The CLI wizard exists first; the page (M5) is the same steps on a screen.

**J3. Second device: 4 steps after install, 1 to 2 decisions.** (1) Choose "Open an existing vault"; CloudCrypt looks in detected sync folders for key files and lists "Documents (in Nextcloud)". If the sync client has not delivered the files, the page says "waiting for Nextcloud to deliver the vault" and refreshes itself; it never shows a format error for a half-delivered folder. (2) Pick it, only if there are several. (3) Type the password or paste the recovery key. (4) The mirror fills in with a progress bar. Nothing asks for paths, formats or keys.

**J4. Daily use: 0 steps.** Open `~/CloudCrypt/Documents` from any file manager or any Open and Save dialog. The agent notices a saved file once it has been unchanged for about 2 seconds [judgement], packs it, writes ciphertext, and the sync client does the rest. If another app holds a file (common on Windows [recall]) the agent retries and the status says "waiting for Word to finish with budget.xlsx". There is no per-file badge in the file manager in the alpha (that needs OS shell extensions [recall]); status shows in the page and the CLI, and a minimal notifier fires for states that need action from M4 (7.5); a tray helper comes later (M9). Zero steps holds while the agent runs: a user who disables autostart gets no alert from CloudCrypt, and files are then not being protected (Condition 10).

**J5. A sync conflict: 2 to 3 steps, 1 decision.** Two devices edit `report.docx` before syncing. On every device the mirror shows `report.docx` (the later edit) and `report (conflict from Laptop, 14 Nov 15:02).docx`. The person opens both and deletes the one they do not want; that delete is recorded as "discarded" so the copy does not return. Nothing is merged automatically and nothing is lost. The four things a person can do with a conflict copy (delete it, delete the plain-name file, rename it, edit it) are plan rows 19 to 22 in 7.2, and `cloudcrypt conflicts --keep <file>` does the first two in one command (Condition 3).

**J6. Forgotten password with the recovery key: 4 steps, 1 decision.** "Forgot password", paste or pick the recovery key, type a new password twice, accept the offer of a new recovery key (default yes). Other devices keep working because they hold the vault key in their keychain; a password is only a lock on a copy of the key. Old key files remain in the vault folder and in the provider's history, so the old password and the old recovery key still open them; a new recovery key does not retire the old one. Only replacing the key (J7) protects future data from someone who saw the old ones (Condition 6). Without both password and recovery key the data cannot be read by anyone, and J2 step 5 says so in plain words.

**J7. Lost or stolen device: 5 steps on one good device, 2 on each other.** (1) Menu, "A device was lost". (2) Read two sentences: this protects files written from now on; it cannot take back what the lost device already holds; also remove the device at your sync provider and change that password. (3) Type the current password. (4) Choose a **new** password and save a **new** recovery key (required, see 8.5). (5) CloudCrypt makes a new vault-key generation and uses it for new writes. Each other device asks for the new password once. Two honest tiers are shown: a lost device with disk encryption and a login lock is probably fine; an unprotected one can read everything it had. **Write limit, open (Owner Ruling A, section 14):** a lost device that still holds the old vault key can write valid segments under the old key. Provisional wording until the owner rules: the protection above covers reading files written from now on; a thief who holds the old key and can reach the sync account can still add or delete records under the old key, and cutting the device off at the sync provider is the stop. The alternative, a must-understand `rekey` cutoff record that makes readers quarantine old-key segments beyond a recorded cutoff, is described in 8.5 (Condition 9). Build is M7.

**J8. Uninstall: 3 steps, 1 decision.** Settings, "Remove CloudCrypt from this device"; choose to keep the mirror as ordinary files (default; the text says they are no longer encrypted by CloudCrypt) or delete it; uninstall the app with the OS. Nothing in the cloud folder is deleted. The removal also purges this device's trash, local state, logs, status file, keychain items, login entry and 15-minute check job (8.6), so no plaintext or key stays behind (Condition 6).

**Where systems differ.**

| Concern | Linux | macOS | Windows |
|---|---|---|---|
| Login start | XDG autostart (+ optional systemd user unit) [recall] | launchd user agent [recall] | Run entry or logon task [recall] |
| Mirror protection | mode 0700 folder | mode 0700 folder | user-profile access list [recall]; the page warns if the mirror is outside the profile |
| Change detection (alpha) | stat-walk polling | stat-walk polling | stat-walk polling |
| Busy-file handling | rare | rare | common; retry with back-off ([repo] internal/vault/rename_windows.go, 31 lines, has the rename retry) |
| Name rules | case-sensitive | case-insensitive, decomposed Unicode possible [recall] | case-insensitive, reserved names, forbidden characters [recall] ([repo] portable.go, 100 lines) |
| Key store | Secret Service; none means "unlock at each login" | login keychain via `security -i` ([repo] internal/keychain) | Credential Manager |
| Platform trap | headless machines have no key store | iCloud and OneDrive "online only" placeholders | OneDrive redirects Documents and Desktop; long paths |

## 3. Goals and non-goals

**Goals.** G1 Managing encrypted files just works: a folder, no commands in daily use. G2 No silent data loss: the vault-folder overwrite P0s are removed by construction; mirror data loss is bounded by named guards, each conditional (C9) (Condition 14). G3 Plaintext never enters the synced vault folder. G4 The provider learns only what 6.7 states. G5 One owner per open vault, one commit routine, one parser for provider bytes, one place that starts child programs, one writer of plaintext. G6 A written format specification with test vectors, so another person can write a compatible reader. G7 A design one maintainer can support: about 10,000 to 13,000 lines of new product Go [judgement] (the one line-count range used everywhere; revision 2 raised it for the added guards) (Condition 14), few moving parts.

**Non-goals.** Protection against a compromised unlocked device or a user with the password and hostile intent. Hiding that a vault exists, its approximate size or its change timing. Protection against a provider that withholds the newest files (detectable only in part, 6.6). Multi-person sharing. Selective sync (all of a vault is mirrored in the alpha). A mount. Revoking a lost device's access to data it already holds (impossible with symmetric keys; section 8.5).

## 4. Threat model and trust boundaries

Each boundary has one place that enforces it and a test that pins it. Scenario ids (SC-nn) are rows of the catalogue in 11.3.

| # | Boundary | Single enforcement point | Test that pins it |
|---|---|---|---|
| T1 | Hostile or careless storage provider or sync client: reads, deletes, truncates, duplicates, reorders, serves old files, swaps in empty placeholders, renames to "conflicted copy" | `internal/wire`, the only package that parses bytes from the vault folder: fixed size caps, no panics, every object authenticated by AEAD (authenticated encryption: encryption that also detects any change) before use; file names carry no meaning | A fuzz target per decoder; a byte-flip test over every region of a sample vault; the hostile-folder simulator (SC-05 to SC-09); the vault-folder walk after every end-to-end scenario that searches names and bodies for planted plaintext (SC-21). [repo] internal/vault/leak_test.go:23 is the ancestor; the review says it checks only file names (P1-21 re-check, review strength 82) |
| T2 | Other accounts on the same machine | The per-user run directory (mode 0700; socket mode 0600) plus the peer-user check on the socket (`SO_PEERCRED` Linux, `LOCAL_PEERCRED` macOS [recall]); no secret in any argument or environment | A test that every command through `platform/spawn` is captured and fails if a secret appears; a second-user connection is refused (Linux and macOS CI); a mode check. Windows relies on the profile folder access list [recall] and a runner that reads it |
| T3 | Web pages and the browser | Only the separate `cloudcrypt-ui` helper listens, on loopback; one middleware checks a bearer header (no cookie), a Host allowlist ([repo] internal/loopback/loopback.go:29 `HostAllowed`, 77 lines, with a table test) and a strict content-security policy; names render with `textContent` | A declared route table where a missing auth level fails the build; the Host table test; a served-HTML test that finds no inline script; an import test that `net/http` appears only under `cmd/cloudcrypt-ui` |
| T4 | Child programs and supply chain | `internal/platform/spawn`, the only code allowed to start a process: argument lists, no shell, scrubbed environment, secrets on standard input. No downloaded programs exist. Go modules limited to an allowlist (11.5) | An import test (only `spawn` imports `os/exec`); a go.mod allowlist test; govulncheck in CI [recall] |
| T5 | Plaintext reaching the synced folder | `internal/plainout`, the only package that writes plaintext to disk, the recovery-key file and restore destinations included (8.6, Condition 6): it compares destinations by file identity (`os.SameFile`, which asks the file system whether two paths are one file) up every parent chain, refuses a mirror that contains or sits inside the vault folder, and refuses a mirror, a recovery-key file or any other plaintext destination inside a detected sync-client root or an ancestor of one (detection covers only known clients and is best effort, C5) | A table over every spelling of a path into the vault root: relative, `..`, symlink, junction, case variants, trailing dots, 8.3 short names [recall]. Today's guard compares path text ([review] go-modernization/plaintext-guard-compares-path-text) |
| T6 | Local state tampering and device compromise | Conceded. The agent's memory, the keychain item, the mirror and local state are readable by anything running as the user | Stated in the first-run text; the page checks and warns about full-disk encryption where detectable (FileVault, BitLocker, LUKS [recall]); never a block |
| T7 | Rollback or replay by the provider | Detection only: key-file generation high-water mark, advanced only from authenticated key files (6.3, Condition 7); per-device sequence and hash chain inside sealed segments; a missing previously-seen segment is reported (6.6) | SC-17 (older folder served after a newer one) |

**Residual metadata** the provider and sync client can learn is listed in 6.7 and justified there.

## 5. Architecture

### 5.1 Processes and the process model

One binary, `cloudcrypt`, in three roles.

- **The agent** (`cloudcrypt agent`), one per user, started at login. A second start connects to the first through its socket and exits. The agent alone holds keys, opens vault folders, writes ciphertext, and writes the mirror. There is no second process that could write, so "one owner per open vault" (the seam the Phase R design wanted, review 5.3) is the whole process model. Inside the agent, a per-vault gate (one mutex and a context) serialises writes; across processes a file lock in the per-user run directory stops two agents.
- **The CLI** (`cloudcrypt <command>`), short-lived, talks only to the agent. Fifteen commands (Conditions 14, 15): `setup`, `open` (starts the page), `status` (`--json`, `--notify`), `list` (the vaults this device knows), `conflicts` (`--json`, `--keep <file>`), `unlock`, `lock` (drops the key and pauses one vault), `password` (change), `recover` (forgot password), `replace-key` (lost device), `restore`, `doctor` (`--repair`, `--accept-folder`), `sync` (with `--now`, `--plan`, `--allow-deletes`), `agent`, `remove`. Moving a vault's mirror (`relocate`) is out of the alpha: `remove` keeping the mirror, then `open` the vault with that folder as the mirror (joining into a non-empty mirror, 8.6). The CLI cannot read or write a vault folder itself except `doctor`, which opens read-only (`--repair` and `--accept-folder` ask the agent).
- **The command-line contract** (Condition 15). Exit codes by name: 0 ok; 1 failed; 2 usage; 3 agent not running; 4 locked; 5 needs attention (conflicts, paused, waiting); 6 refused by a safety guard (placement, mass delete, rollback); 7 this CloudCrypt is too old for the vault. One grammar, `cloudcrypt <verb> [vault] [flags]`. One house flag table: `--json`, `--vault`, `--now`, `--plan`, `--notify`, `--keep`, `--repair`, `--accept-folder`, `--allow-deletes`; no flag skips a safety guard except one that names it. One refusal formatter prints three lines: what happened, why, what to do next. With no agent running: `status` reads `status.json` and says the agent is not running; `doctor` opens read-only; `agent` and `setup` start it; every other command prints the refusal and exits 3. `--json` on `status` and `conflicts` gives the same facts for scripts. One table of commands and one error-to-text function are built from this list in M4.
- **The page helper** (`cloudcrypt-ui`, M5), a separate small program that serves the setup and status page on loopback and is itself a client of the agent socket. It holds no key and never imports the format package; a password typed in the page passes through it to the agent. It is the only code that imports `net/http`. How a browser gets in without a secret on a command line: the CLI asks the agent for a **single-use code of at least 128 random bits, valid 60 seconds** [judgement]; the helper writes a mode-0600 HTML file in the run directory that submits the code by POST to `http://127.0.0.1:<port>/redeem`; the browser is opened on that file's path (a path in a process list is harmless; a code is not). The helper swaps the code for a session token returned in the page body, kept in the tab's `sessionStorage`, and sent in an `Authorization` header on every request. There is no cookie, so no other program on the loopback address inherits access (the shape of [review] webui-backend/session-cookie-leaves-the-port). Whether every browser lets a `file:` page POST to loopback is unverified (spike (b)); the single-use link printed to the terminal is the primary path, and the `file:` page is built only if spike (b) passes in all four browsers (Condition 12). The helper's loopback rules (Condition 12): a random port with an exclusive bind (on Windows `SO_EXCLUSIVEADDRUSE` [recall]); a one-shot listener that stops accepting once a code is redeemed; at most 3 failed redeems, after which the helper exits and the code dies; connections from non-loopback peers dropped.
- **The tray helper** (M9), later, in its own Go module so cgo (a way for Go to call C code; it makes cross-compiling harder) never touches the core.

The agent also writes `status.json` atomically in the run directory so the helpers can show state without holding the socket. It carries a schema version, `updatedAt` (rewritten at least every 30 seconds [judgement]), `lastGoodSync` (the end of the last cycle that left nothing in the mirror uncommitted and the vault folder merged), `filesNotYetInVault` (a count with reason codes; file names are served by the agent over the socket on request and are not stored in this file) and the current status states (7.5). A reader treats an `updatedAt` older than 2 minutes [judgement] as a dead agent and says "CloudCrypt is not running, files are not being protected"; `status` falls back to reading this file when the socket is dead (Condition 10). A minimal notifier in `platform/spawn` (`osascript` on macOS, `notify-send` on Linux, a toast on Windows [recall]) fires rate-limited notices for action-needed states from M4 (7.5). The file's placement and schema rules are in 8.6 (Conditions 6, 15).

### 5.2 Components, owners and one-way dependencies

A package may import only packages lower in this list. A test builds the import graph with `go list -deps` and fails on any edge not in an allowed-edges table in the repository.

```
cmd/cloudcrypt            thin: argument parsing, talks to the agent
cmd/cloudcrypt-ui         page helper (only importer of net/http); client of the agent
internal/agent          single instance, session registry, control socket, jobs, status
internal/mirror         poller, state store, quiesce, plan, apply, conflict materializer, guards
internal/engine         open vault session; Snapshot/Read/Commit; merge; pack and segment writers
internal/store          Store interface (create-only PutNew) and the folder implementation
internal/tree           PURE: records -> current tree + conflicts; no I/O
internal/format         PURE codecs + the spec's byte layouts
internal/wire           the one decoder of untrusted bytes: caps, no panics (format uses it)
internal/seal           AEAD, HKDF, Argon2id calls, constant-time compare; nothing else
internal/names          portable-name rules, NFC, simple case-fold collision keys
internal/plainout       the only writer of plaintext; SameFile and placement checks; trash
internal/state          device-local state (anchors, seen-segment set, vault-folder sentinel, merged-tree cache, mirror state table); versioned; rebuildable, never authoritative
internal/platform       keychain, autostart, atomic write + fsync + Windows rename retry, spawn, run dir, lock, device label
internal/sim            test-only: fake provider and sync client (section 11.4)
```

Owners (the single place for each concern): the vault folder is written only by `engine` through `store`; the mirror and any plaintext only by `plainout` (called by `mirror` and `restore`); every parse of provider bytes only in `wire` and `format`; every child process only in `platform/spawn`; the keychain only in `platform` behind one interface passed in from `main`, so tests never reach the real OS stores ([review] webui-backend/tests-reach-real-keychain). `tree` and `format` import nothing from `platform`, `mirror` or `agent`. These rules turn review themes 5.3, 5.4, 5.5, 5.9 and 5.13 into compile-time or test-time facts.

**Core library API** (`internal/engine`; shape is [judgement], detailed in M2):

```go
type Store interface {                       // the second-backend seam
    List(ctx, kind Kind, shard string) ([]Item, error)
    Get(ctx, kind Kind, name string) (io.ReadCloser, error)
    GetRange(ctx, kind Kind, name string, off, n int64) (io.ReadCloser, error)
    PutNew(ctx, kind Kind, name string, r io.Reader) error   // create-only; error if it exists
}
func Create(ctx, s Store, o CreateOptions) (*Vault, RecoveryKey, error)
func Open(ctx, s Store, u Unlocker) (*Vault, error)          // password, recovery key, or raw vault key
func (v *Vault) Refresh(ctx) (changed bool, err error)        // lists keys/ and new segments; never writes (Condition 8)
func (v *Vault) Snapshot() *Snapshot                          // immutable, shared, not copied
func (s *Snapshot) Stat/List/Conflicts/Heads(...)
func (v *Vault) Read(ctx, e Entry) (io.ReaderAt, int64, error) // decrypts only needed chunks
func (v *Vault) Commit(ctx, b *Batch, planned *Snapshot) (Receipt, error)
```

`Commit` takes the snapshot the caller planned from. It refreshes first; if the head set of any node in the batch differs from that snapshot, it returns `ErrStale` and the caller re-plans (bounded retries). A stale write is therefore not something a caller can forget to prevent: it cannot commit without naming the view it decided from. This is the Phase R "commit routine that reads disk before writing" made structural (review 5.1). `Refresh` also lists `keys/`: if a key file shows a higher key generation than the vault key in use, `Commit` returns `Locked: new vault key` and writes nothing, so a device returning from sleep or from a month offline refuses before it writes under a replaced key (Condition 8). A short window remains between a key replacement and the next refresh (8.5, C8).

### 5.3 Where keys live and for how long

- A random 32-byte **vault key** per **key generation** (generation 0 at creation). Each vault key also has a random 16-byte **key id**; the generation number is for display order only, and files name the key by id (Condition 9). Sub-keys come from HKDF-SHA256 with a distinct label per purpose and per file (6.2), so a hash for one purpose cannot be mistaken for another ([review] vault-crypto/id-hmac-domain-collision).
- The agent holds vault keys in memory while a vault is unlocked. Go cannot guarantee zeroing memory [recall]; this is a stated residual.
- On disk the key exists only as wraps inside key files in the vault folder: one under the password (Argon2id), one under the recovery key (256 random bits, no stretching needed), and an older-keys chain sealed under the newest key (6.3).
- "Remember on this device" (default on) stores the vault key in the OS keychain under an account named by the vault id, not the folder path, so it survives moves (the same property as [review] strength 19). macOS: `security -i` with the secret on standard input ([review] strength 30, about internal/keychain/keychain_darwin.go, which I did not open) (Condition 14). Windows: the credential API. Linux: Secret Service. The password is never stored.
- Locked state: the agent drops the key, stops syncing the vault and leaves the mirror files alone. On a Linux machine without a Secret Service the vault is locked at each login and `cloudcrypt unlock` prompts; this is said once, not hidden.

### 5.4 The control channel

A Unix domain socket in a per-user run directory (0700; socket 0600) on Linux and macOS. On Windows, the same kind of socket in `%LOCALAPPDATA%\CloudCrypt\run` [recall: supported from Windows 10 build 1803]. macOS limits socket paths to about 104 bytes [recall], so the path is short and a test creates it under a long home directory. If the Gate 0 spike (section 12) shows Windows sockets unusable, the fallback is a named pipe through `golang.org/x/sys/windows` with an owner-only access list [recall]; nothing in the core uses TCP. Authentication is the file permission plus the peer-user check on the agent side, and an ownership check on the client side before any secret is sent (below). No secret ever travels in a command line or an environment variable; passwords for create, join, unlock and password change travel as socket messages, which the agent clears after use.

**Run directory and client checks** (Condition 12; the paths are [judgement], to be confirmed in spike (a)). Linux: `$XDG_RUNTIME_DIR/cloudcrypt`, or `/tmp/cloudcrypt-<uid>` created mode 0700 with an owner check when that variable is unset. macOS: `~/Library/Caches/CloudCrypt/run`, with the long-home-directory test above. Windows: `%LOCALAPPDATA%\CloudCrypt\run`. Before sending any secret the client checks that the directory and the socket are owned by the current user, that the directory is mode 0700 and the socket 0600, that neither is a symlink (`lstat`), and, after connecting, that the server's peer user (`SO_PEERCRED` on Linux, `LOCAL_PEERCRED` on macOS [recall]) is the current user. Windows equivalent [recall]: the directory's owner is the current user's security identifier and its access list names no user or group beyond that user, system and administrators; for a named pipe, the pipe server's process owner is checked the same way. A failed check prints the refusal and sends nothing.

**Handshake, handover and replacement** (Condition 12). The first message in each direction carries a protocol version and a build id. A client that sees a different build asks the agent to hand over: the agent finishes the commit in flight, releases the run-directory lock and exits, and the client's build starts a new agent. A new agent refuses to coexist with an older one holding the lock: it waits up to 30 seconds [judgement] and then reports "an older CloudCrypt is running; quit it". Windows cannot replace a running executable, so the replacement procedure is: write the new executable beside the old as `cloudcrypt.exe.new`; ask the running agent to hand over and exit; wait for it; rename the old to `cloudcrypt.exe.old` and the new into place; start the agent; delete the `.old` file at the next start. If the handover fails, the old agent keeps running and status says an update is waiting.

**Updates** (Condition 12). In the alpha an update is a manual reinstall by the owner; there is no downloader (T4). A device whose CloudCrypt is older than a vault's `minWriter` goes read-only and shows "This device needs a newer CloudCrypt" (7.5). The maintainer named in `docs/dependencies.md` checks the three vendored modules and each Go security release at every Go release and at least monthly; a security release means a rebuild and a reinstall on every device.

A control-files-only channel (the runner-up's) was considered. It removes the socket but cannot move a typed password to the agent without writing it to a file, so the socket stays.

### 5.5 The access model, per operating system

**Chosen: mirror folder on all three.** Mirror at `~/CloudCrypt/<name>`, owner-only, created by `plainout`. Change detection in the alpha is a **stat-walk** (a scan that compares each file's size, modified time and file id against state) on all three systems. The poll interval is derived from the measured cost of a full walk, not guessed: the idle interval is 100 times the measured walk time, clamped between 30 seconds and 10 minutes, so the idle duty cycle stays under 1 percent of one core [judgement]; a 2-second re-stat applies only to the few paths changed in the last minute, not to the whole tree. A full content-level re-check runs every 10 minutes. Spike (c) is a numeric gate (section 12) (Condition 13). Event watchers (inotify on Linux, `ReadDirectoryChangesW` on Windows, both through `golang.org/x/sys` [recall]) are a latency optimisation added in M9 behind the same interface once the macOS polling cost is measured; FSEvents needs cgo and kqueue opens a descriptor per file on macOS [recall]. A watcher can miss events, so polling is the correctness path either way.

| System | Chosen | Needs | What breaks or costs |
|---|---|---|---|
| Linux | mirror; GTK bookmark entry so file managers list it [recall: `~/.config/gtk-3.0/bookmarks`] | nothing | some file managers ignore the bookmark |
| macOS | mirror; "Show in Finder" button | nothing (no Full Disk Access for `~/CloudCrypt` [recall]) | polling cost grows with file count; no Finder sidebar entry without private APIs [recall] |
| Windows | mirror | nothing | sharing violations from Office or antivirus; reserved names; MAX_PATH on old setups [recall] |

**Mounts considered and rejected for the alpha** (all [recall]): FUSE on Linux is a good later option (libfuse3 is commonly installed). On macOS, macFUSE needs a kernel-extension approval and sometimes a restart; FUSE-T avoids the kernel extension but is a third-party package; an NFS loopback mount needs no driver but has weak locking, and Office and SQLite saves behave badly on it; Apple File Provider is the native answer but is an app extension in Swift or Objective-C, outside the Go-only rule. On Windows, WinFsp needs an administrator installer; the built-in WebDAV client had a 50 MB limit and Basic-over-HTTP issues (the review's drive review flagged these, [review] drive-review:259 as carried from the proposals; I did not open that line) (Condition 14); the Cloud Files API (placeholders, as OneDrive uses) is native but a large project.

**Why this is future-supportable.** The mirror is the first **materializer**, a plug over one engine and one format. The engine is shaped by node id (6.5), with `Read` random-access and `Commit` batch-atomic, so a FUSE, Cloud Files or File Provider materializer can attach as a separate program or module without any format change, and cgo, if a mount library needs it, stays out of the core. A second storage backend (S3, WebDAV, SFTP) is a `Store` implementation: it needs only "create a file that does not exist" and "read it".

### 5.6 Data flows

**Open (agent start, join, unlock).** (1) Read the vault key from the keychain by vault id, or take the password or recovery key from a client. (2) List `keys/`, parse headers under caps, check the generation high-water mark in local state; if the folder shows an older generation than this device has seen and authenticated, report "an older vault is being served" and stay read-only; a higher generation the device cannot authenticate shows "Locked: new vault key, enter password" and is never recorded (6.3, Condition 7). (3) Unwrap, or trial-decrypt the key check, so a stale keychain item is detected. (4) Load the local merged-tree cache (6.6) or the newest checkpoint if any, list `keys/` and the `log/` shard folders, merge new segments into a `Tree`, and compare the result with the seen-segment set and vault-folder sentinel; if any seen segment is missing, planning is report-only (7.2) (Conditions 1, 8, 13). (5) Verify the mirror sentinel and run the placement guard. (6) Start the poll and sync loops. A half-delivered folder (no key file yet, a zero-byte file, a truncated file) is "waiting", never an error.

**Write (mirror to vault).** The poll finds a changed file; apply the ignore set, file-kind table and per-file rate limit (7.6); quiesce (unchanged 2 seconds; size and time stable across a read); chunk it; write the chunks inline in the segment if the commit is small, otherwise into a pack (6.4, Condition 13); verify the pack exists in the folder by size; build records with `base` taken from what the mirror last saw; `Commit(batch, planned)`; on success update the local base. Packs are written before the segment that names them, so a reader that sees a segment can find its data or knows it is "not yet delivered" (6.6).

**Read (vault to mirror, and restore).** Plan says "write head H into the mirror": `engine.Read` yields the file by chunk; `plainout` applies it in the fixed order of 7.2.2 (temp file in the reserved temp folder, journal entry, re-check, move to trash, compare the trashed file with base, rename into place, update local state), with a crash table per step (Condition 4). Restore of an older version or a deleted file uses the same path to a chosen folder.

**Sync (vault to mirror).** `Refresh` lists changed `log/` shards, merges, and the plan function (7.2) compares mirror, base and head per node.

**Conflict.** Two live heads of one node (7.1) materialize as the winner at the name and each other head as a conflict copy; resolution is one of four user actions, each a plan row (19 to 22 in 7.2), recorded as `discard` where a head is resolved away (Condition 3).

**Garbage collection (after the first alpha cut).** An `intent` record names packs; only after a fence and a final re-merge are they deleted; any device repairs from its mirror (7.4).

## 6. Storage format

This is "format 0": explicitly unstable until the owner freezes it (decision 10), but every file already carries its version so freezing costs nothing later. The outline below is the table of contents of a standalone specification, `docs/format-spec.md`, written in M1 with numbered rules (F-001 and up), each tied to a test.

### 6.1 Principles

1. Everything in the vault folder is immutable once written. Files are made under a temporary name in the same folder, flushed, renamed into place, and the folder flushed ([repo] vault.go:1469-1510 is today's writer; it lacks a flush of a newly created parent folder, review strength 1 "But").
2. A file's name is the first 128 bits (32 hex characters) of the SHA-256 of its exact bytes. The name is not secret and carries no meaning; readers verify it where it fits the pattern and otherwise trust the content. A file with a conflict suffix, a different name, or a copy is read by content and identified by its hash.
3. No two devices write the same name, because a name is a hash of ciphertext that contains fresh random salts.
4. State is a set of records; any device derives the same state from the same set; union, order-independent, duplicates harmless.
5. Every object is authenticated over the exact bytes read. Nothing re-serialises a struct to check a tag ([review] vault-crypto/configmac-struct-remarshal).
6. A missing referent is "not here yet", never corruption.

### 6.2 Keys and labels

- Vault key K_g: 32 random bytes per key generation g.
- Per-file key: HKDF-SHA256 over K_g with the file's 16-byte random salt as HKDF salt and an info string `"cc0/" + purpose` where purpose is one of `pack`, `log`, `footer`, `snap`, `history`, `keycheck`. Each file therefore has its own key, used for a bounded number of frames, so no random-nonce limit is approached. Nonces are 12 bytes: 4 zero bytes then an 8-byte frame counter. Cipher: AES-256-GCM from the standard library.
- Chunk id: HMAC-SHA256 over the plaintext chunk with key HKDF(K_g, `"cc0/chunkid"`); equal content has equal ids only inside this vault.
- Recovery key: 256 random bits shown as 13 groups of four characters with a check character, preceded by a version character (`1`) and a four-character fingerprint of the vault id, so a person holding several keys can tell which vault a key belongs to and a future encoding is not mistaken for this one (recovery_phrase.go, 83 lines by my count, supplies the read-back and constant-time compare idea; the encoding changes). The derivation from the displayed key to the recovery slot's wrapping key, the character set and the check character are frozen in `docs/format-spec.md` with a test vector (6.9) before M1 exits (Condition 11).
- Password key: Argon2id ([review] 6.2 reports today's default as time 3, 64 MiB, 4 lanes, about 50 ms, at crypto.go:27-29, which I did not open). The decoder range-checks the numbers before any work: memory 8 MiB to 1 GiB, passes 1 to 10, lanes 1 to 16 [judgement for the numbers], which closes [review] vault-crypto/kdf-params-unbounded. The numbers for a new vault are chosen by measured time on the creating device, aiming at 0.5 to 1 second of work [judgement] inside those ranges, and stored in the key file; a device that cannot afford them says so. A reader tries at most 8 key files per unlock (highest `fileGen` first) and at most 4 distinct sets of numbers, so the password-hashing work per attempt is bounded; key files beyond those caps are listed by `doctor` and not tried (Condition 15).

### 6.3 Key files (directory `keys/`)

`keys/<fileGen>-<hash32>.cck`. Layout: magic `CCKY`; format major, minor; `minReader`; `minWriter`; required and optional feature lists; vault id (16 random bytes); `fileGen` (increments with every rewrap), `keyId` (16 random bytes naming the vault key; Condition 9) and `keyGen` (the vault-key generation, display order only); names profile id (6.5); Argon2id numbers; then slots: password slot (salt, nonce, wrapped key), recovery slot (nonce, wrapped key); then a `history` block holding K_0 to K_(g-1) sealed under K_g so old data stays readable, with each old key's key id stored next to that key, so a reader maps the key id named in a pack or segment header to its key without trial decryption (revision 3, Condition 9); then a `keycheck` block. The password slot's and the `keycheck` block's associated data (bytes the cipher authenticates but does not encrypt) is a label plus the SHA-256 of the **exact stored header bytes**, so any edit of a header field fails the unwrap or the key check (graft from the engine-clean proposal; binding the key check is Condition 7). The recovery slot's associated data is a label plus the SHA-256 of the **immutable header fields only** (magic, format major, vault id, `keyId`), not `fileGen` or `minWriter`, so a password change copies the recovery slot into the new key file unchanged and does not need the recovery key; the header fields the recovery slot does not bind are still checked by the `keycheck` block after unlock (Condition 11). A password or recovery change writes a **new** file with `fileGen` + 1; it never edits an old one. A device reads all key files, tries the credentials against the highest `fileGen` files first (several files with equal `fileGen` can exist after two concurrent changes; each is tried), and each device records the highest `fileGen` and key generation it has **authenticated** (the idea of [repo] anchor.go:46-51, keyed by vault id not path). A number read from a header is never stored until the key check of that same file has authenticated with a key the device holds or has just unlocked; a key file with a higher `fileGen` that the device cannot authenticate shows "Locked: new vault key, enter password" and is not recorded, so a forged header cannot raise the mark (Condition 7). Create is create-only; setup refuses a folder that already holds key files and offers to join ([review] vault-crypto/init-replaces-only-key-file).

Limits stated now: the header is checked before it is authenticated (the numbers must be read to start work), so it is range-checked, not trusted, and its numbers are never stored before authentication (Condition 7). A hostile provider can still plant a forged key file that makes devices show "Locked" and stop writing; reading continues, and the way out is `cloudcrypt doctor --accept-folder` (7.5). Because old key files remain, **changing the password does not retire the old password** for anyone who kept a copy of an old key file (a provider's version history keeps them); only 8.5's key replacement does that for future data.

### 6.4 Packs (directory `packs/<xx>/`)

`packs/<first two hex of name>/<hash32>.ccp`. Magic `CCPK`, version, `keyId`, 16-byte salt, then sealed frames (one per chunk, associated data = the chunk id), then a sealed footer index listing (chunk id, offset, sealed length, plain length), then a plain length trailer. Many files imported together share one pack (target about 16 MiB). That helps bulk import only: an isolated small edit would otherwise cost one pack plus one segment. So a commit whose new chunk bytes total at most 1 MiB [judgement] writes its chunks inline in the segment's sealed block, with no pack (one synced file per commit; the chunk-list entry then names the segment, and a writer reusing such a chunk checks the segment as it checks a pack), and a device writes at most one segment per 5 seconds (the batching floor) [judgement]. The earlier claim that packs fix today's "two synced files per small file" ([review] perf-scale/two-synced-files-per-small-file) and the 20.4 ms-per-file flush cost (review 6.3: four full flushes per small file, about 96% of put time on macOS) is **withdrawn until M2 measures it**. Caps [judgement]: a record at most 1 MiB, a sealed segment at most 16 MiB, a pack at most 32 MiB; at about 64 bytes per chunk-list entry one record holds about 16,000 chunks, so a file whose chunk list would not fit is refused with a named skip (7.6) until list indirection exists. Segment budget (revision 3, Condition 13): 10,000 segments per device per month is a **target that M2 measures**, not something the batching floor guarantees (one segment per 5 seconds would allow about 518,000 a month). If the measurement shows the target is needed and missed, a longer quiet-coalescing window (merging commits made close together into one segment) enforces it. M2 measures start-up at one year of heavy use (100,000 segments) with the merged-tree cache (6.6) against a target of 10 seconds (Condition 13). The footer is for verification and recovery; reads use offsets from the records. Frame boundaries are visible only in the sealed footer.

Chunking: the existing gear-hash splitter ([repo] chunker.go:15-21 default 2, 8, 16 MiB minimum, average, maximum; the mask is `nextPowerOfTwo(AvgSize)-1` at chunker.go:41, so the average must be a power of two). The alpha uses about 256 KiB, 1 MiB and 4 MiB [judgement] so small and medium files dedup and one pack holds many; M1 measures this before the numbers are frozen.

After decrypting a chunk the reader recomputes its id (today's strength, review strength 15). **Dedup is verified, not assumed**: a writer reuses a chunk only if the pack that holds it is present at its recorded length and its footer authenticates ([review] vault-crypto/dedup-trusts-existence).

### 6.5 Segments (directory `log/<xx>/`), records, identity and names

`log/<xx>/<hash32>.ccj`. Magic `CCLG`, version, `keyId`, 16-byte salt, then one sealed block. Inside: the writer's format level (so a reader can tell which `minWriter` a segment was written under; Condition 11), a random 16-byte **device tag** (never in the file name), a sequence number, the hash of this device's previous segment, a time, a short list of the newest (device tag, sequence) the writer had seen from others, a human device label ("Laptop"), then records. A segment is a whole commit: all of it or none of it. Hiding the device inside the sealed block means the vault format does not reveal how many devices there are (the provider can still estimate from timing, and a sync service may know its own account's devices, 6.7; Condition 14); the chain and the seen-list give the detection that per-device public folders would give (6.6).

**Encoding.** Records are length-prefixed binary: kind byte (high bit = must-understand), varint length, then fields as tag byte, varint length, value. A reader skips an unknown optional field or record, and goes read-only on an unknown must-understand record. Binary tag-length-value was chosen over JSON because duplicate keys and number forms make JSON ambiguous for an independent decoder [judgement].

**Records.**
- `put`: node id (16 random bytes), parent id, name (UTF-8), kind (file, folder), size, modified time, executable bit, chunk list (chunk id, pack id, offset, sealed length, plain length), and `base`, the version ids this edit replaces. The **version id** is the SHA-256 of the record body.
- `move`: a new version of a node with a new parent or name that inherits the chunk list of its base; renaming a folder of 10,000 files is one record ([review] 5.6; today's rename is a re-encrypting copy then delete, webdav/move-is-reencrypting-copy-then-delete as named by the proposals, not re-opened).
- `delete`: node id, `base`. A tombstone is a version.
- `discard`: a version id the user resolved away (J5).
- Reserved now: `intent` (garbage collection; **must-understand**, and enabling garbage collection raises `minWriter`), `checkpoint` reference (optional only while no segment is ever pruned; the alpha never reclaims segments, so its bound is the segment budget of 6.4), `raise-min-writer` (**must-understand**, writable with the vault key alone, so raising the writer level needs no password or recovery key; the effective `minWriter` is the larger of the key file's value and any such record), author signature field, sharing slot kind. A `rekey` cutoff record is reserved but its definition awaits Owner Ruling A (8.5, section 14) (Conditions 11, 9).

**Directory identity.** Every file and folder has a random 128-bit node id; the root has a fixed id. Paths are derived (parent id plus name) and never stored as keys. A rename touches one node. An orphan (a node whose parent is not in the merged records) is first held as "waiting for data" for a grace period (until the next full listing and at least 1 hour [judgement]) and only then shown under a synthetic `Recovered` folder; a cycle goes there at once. Placements under `Recovered` are derived on every merge and never written back, so a late parent brings the node home without a record. Nothing is dropped (7.1; Condition 15).

**Names.** Stored names are NFC (Unicode composed form; needs `golang.org/x/text/unicode/norm` since the standard library has no NFC [recall]). Collision keys use simple case-folding built from the standard library's `unicode.SimpleFold`. Two nodes with the same collision key in one folder are both kept: the earlier version keeps the name and the other is rendered with a deterministic suffix from its node id, so every device renders the same. Windows-illegal names are renamed only when materialized, with the original name kept in the record ([repo] portable.go). Valid Linux names are never refused at write time. The names-profile id in the key file records which rules were used; differing Unicode table versions between devices can change collision decisions and therefore cosmetic names, never data (residual).

### 6.6 Freshness, rollback and half-delivered files

Each device remembers locally the highest key `fileGen` and the newest segment it has seen per device tag. If a segment it already saw later disappears or changes, `status` shows a loud warning. A segment lists the newest segment it saw from each other device, so if device B's segment says "A up to 12" and A's 12 is absent, that is detectable. Withholding the newest segments of everyone is undetectable and stated as a residual. A zero-byte, truncated or hash-mismatched file is treated as "not delivered yet" and retried with back-off; after 24 hours [judgement] `doctor` reports it. An entry whose pack has not arrived shows as "waiting for data" and is retried; it is never an error. A device fork (two writers with one tag, from a cloned disk image) is **kept as data**: both segments are read, and before writing sequence N the engine checks whether the vault already holds a segment from its own tag with sequence N or more that it did not write, and if so adopts a fresh tag ([review] vault-sync/cloned-device-id-silent-drop).

Finding new segments without listing a huge flat folder: segments live in 256 shard folders; the agent lists only shards whose modified time changed and makes a full listing every 10 minutes as a safety net [judgement]. A device-local **merged-tree cache** (in local state, versioned, rebuildable, keyed by the set of merged segment names) bounds start-up cost from M2 on; checkpoints (M8) bound it across devices and are the only thing allowed to prune (Condition 13).

**Seen set and vault-folder sentinel** (Condition 1). Local state keeps the set of segment names this device has merged and, per device tag, the highest sequence seen, together with the vault id: the **vault-folder sentinel**. At Open and at every refresh the folder is compared with it. If any previously seen segment is missing or changed, or the vault id differs, planning becomes **report-only for every node**: nothing is applied in either direction, no delete is committed, and status says "the vault folder is missing files this device saw before" and lists what changed. Exception owed at M8: garbage collection and checkpoints remove segments on purpose; M8's design must say how a deliberate removal is recorded (an `intent` or checkpoint record naming the segments) so that this rule accepts exactly those and no others. Exit: `cloudcrypt doctor --accept-folder` (7.5).

**Healing a lost pack** (Condition 15). A mirror file that still matches its last synced version, whose chunk or pack is missing from the folder after the grace period (24 hours [judgement], as above), is re-chunked and written again into a new pack with a new record pointing at it; `cloudcrypt doctor --repair` does the same on request. `status` shows "N files exist only on this device" for files whose data the vault folder cannot currently supply, so a person is told before the mirror becomes the last copy.

### 6.7 What the provider and the sync client can still learn

That a CloudCrypt vault exists (visible folder, magic bytes); its format version, required features and password-hashing numbers; the vault id; how many key files exist (roughly how often a password or key was changed) and their generations; the key generation of each pack and segment; the number, sizes and modification times of key, pack and segment files (so activity timing, roughly how much changed and when, total size, a large file as a large pack). They cannot read: file names, folder structure, per-file types or which chunks repeat, and the vault format does not reveal how many devices there are or which device wrote what. They can estimate: the size of an isolated write (from the size of a new file), activity timing and volume, and a device count from timing patterns; a sync service may know its own account's devices directly (Condition 14). No padding in the alpha [judgement: Padme-style padding was rejected as gold-plating]. Their active powers (delete, truncate, replay, serve stale, rename) are covered in section 4, T1 and T7.

### 6.8 Versioning for the future

Every file: 4-byte magic, format major and minor, `minReader`. Readers refuse a file whose `minReader` is above theirs with "update CloudCrypt". The key file carries `minWriter` (older writers read but do not write) and required and optional feature lists; a reader that does not know a required feature refuses, one that does not know an optional feature ignores it. Records carry the must-understand bit. Until the freeze (decision 10) breaking changes are free, but each updates the spec and the golden files. Before the freeze, any build refuses a vault whose format minor differs from its own, with a plain message ("this vault was made by a different CloudCrypt build; update it or make a new vault") instead of guessing. The upgrade rule (who may write after a format step, using `raise-min-writer`, 6.5) is written in the spec before the freeze; that is a freeze gate (decision 10) (Condition 11).

### 6.9 Test vectors to publish

`make vectors` writes into `testdata/vectors/` from the reference code with a fixed test-only random source: HKDF sub-keys from a known vault key; an Argon2id password key with small numbers; a sealed slot; a key file; a pack; a segment; a small vault; a vault with a conflict; a vault with a replaced key; a recovery key's displayed form with its version character and fingerprint, and the slot wrapping key derived from it (Condition 11). A test regenerates them and fails on any differing byte. A second, independent decoder written from the spec alone (about 300 lines, test-only, in a different language or package with no imports from `format`) must read the golden vaults.

## 7. Sync, conflicts and garbage collection

### 7.1 Versions, heads and conflicts

For one node the versions form a graph through `base`. The **heads** are the versions no other version names in `base` and that were not discarded. One head: that is the file (or a deletion). Several heads with identical content and placement collapse. Delete beside a live head: **the edit survives** (a delete only beats the versions it names as its base; [repo] PROMPT.md:33 asks for "delete tombstone precedence over stale manifests", which this keeps: stale means an ancestor). Several live heads that differ: a conflict. The winner by (writer's time, version id) keeps the plain name; clocks only choose which copy keeps the name, and a wrong clock costs a name, not data. Each other head renders as `name (conflict from <device label>, <date>).ext`. The rendered name is fixed at first materialization on each device (kept in local state, 7.2.1) and is not renamed when other heads later change; two devices may therefore show different conflict-copy names for the same head, never different data (Condition 3). Device labels and names pass the one sanitiser (7.6). Because a new edit of the winner lists the winner as its `base`, the loser stays a head until deleted, which writes `discard`: a conflict copy cannot be destroyed by the next save ([review] vault-sync/pending-conflict-copy-destroyed-by-next-save). Conflict rendering is computed from the graph, not parked in a file slot. A rename and an edit of the same node made apart show as a conflict in the alpha (both kept); merging placement and content separately is a candidate improvement (decision 15). Moves that would create a folder cycle are skipped in (time, version id) order and the node is shown under `Recovered` (after the orphan hold-back in 6.5; Condition 15). Nothing is dropped.

### 7.2 The mirror plan: a pure three-way function

For each node the engine compares three things: the **mirror** now (M), the **base** B (the version this device last synced, kept in the local mirror state table, 7.2.1), and the vault's **head** H. The plan function is pure (inputs in, actions out) and **total** (Condition 1): its input space is the product of mirror state (missing; matches B; edited since B; equal in content to a current head while B is older or absent; an ignored or refused kind) by base state (none; present in the merged graph; absent from the merged graph) by head state (none; equal to B; live and descending from B through present records; deleted and descending from B; an older ancestor of B; not shown to descend from B). Every cell maps to a numbered row below, a property test enumerates the product and fails on any reachable cell with no row, and cells that cannot occur are listed in the test and asserted impossible. A head **descends from** B only if every record on the path from B to H is present in the merged graph; rows 2, 3, 7 and 8 apply only then.

| # | Mirror vs base | Head | Action |
|---|---|---|---|
| 1 | M matches B | H = B | nothing |
| 2 | M matches B | H live, differs, and descends from B | write H into the mirror in the apply order of 7.2.2 (Conditions 1, 4) |
| 3 | M matches B | H deleted, and descends from B | apply order of 7.2.2 with removal; a folder is removed only after its children's rows have run and only if empty (7.2.1) (Conditions 1, 2, 4) |
| 4 | M edited | H = B | commit M as a new version, base [B] |
| 5 | M edited | H live and differs | commit M with base [B] (the version it last saw), which creates a second head and a visible conflict |
| 6 | M missing, B exists | H = B | commit a delete, base [B] (subject to the mass-delete guard) |
| 7 | M missing, B exists | H live, differs, and descends from B | the edit beats the delete: write H into the mirror (Condition 1) |
| 8 | M missing, B exists | H deleted, and descends from B | nothing; drop the base (Condition 1) |
| 9 | M present, no B | no H | new local file: commit |
| 10 | M present, no B | H exists | same content: adopt as base. Different content: never overwrite; commit M as a new node (name collision rule renders it) |
| 11 | M missing, no B | H exists | new remote file: write into the mirror |
| 12 | M missing, no B | none | nothing |
| 13 (Condition 1) | any M; B absent from the merged graph, or H not shown to descend from B | incomplete view | **incomplete view**: no mirror write, no removal, no delete commit; status "waiting for provider"; a mirror edit may still be committed with base [B], because a commit only adds a version |
| 14 (Condition 1) | any M; B exists | H is an older ancestor of B (the view is behind this device) | nothing is written to the mirror; an edited mirror commits with base [B]; status "the vault view is older than this device's" |
| 15 (Condition 1) | M matches B | no H although B exists (node missing from the merged tree) | nothing; treated as row 13, never as a delete |
| 16 (Condition 1) | M edited | H deleted, descends from B | the edit survives (7.1): commit M with base [B]; the delete stays beside it and the node shows the live head |
| 17 (Condition 4) | M's content equals a current head H; B older or none | H live | **adopt H as base**, no commit (the crash-recovery row of 7.2.2) |
| 18 (Condition 5) | M is an ignored or refused kind (7.6) | any | no action; listed in the skip report |
| 19 (Condition 3) | conflict-copy path C (its own state row, base = a non-winning head Hc): M missing | Hc live | commit `discard` of Hc; the copy does not return |
| 20 (Condition 3) | plain-name path P: M missing, B = the winning head, while another head Hc is live | Hc live | commit `discard` of B (the head at that path); Hc survives and is renamed to P by a checked internal rename (a `move` through 7.2.2). Overrides rows 6 and 8 when the node has another live head |
| 21 (Condition 3) | conflict-copy path C vanished and a new path N appeared with the same file id (rename or move of a copy) | Hc live | commit `discard` of Hc and a new node at N (a `put` reusing the chunks) |
| 22 (Condition 3) | conflict-copy path C: M edited relative to Hc | Hc live | commit a new version with base [Hc]; the copy keeps its own lineage and its fixed name |
| 23 (Condition 1) | any cell not matched above | any | no action on the file; the node is listed "needs attention"; the property test requires that no reachable cell lands here |

**Precedence** (revision 3, Condition 1): where row 13 and row 17 or 18 match the same cell, row 18 applies first (an ignored or refused kind is never acted on), then row 13, then row 17; so an incomplete view blocks adoption of H as base, and row 17 applies only when the view is complete. Rows 19 to 22 are keyed by path: the state table has one row per path, so each rendered conflict copy has its own base. `cloudcrypt conflicts --keep <file>` commits the discards that leave only that file's head live, at the plain name (Condition 3). All four cases are table-tested.

Rule behind rows 9 to 12: **no base means never delete**, so lost local state is safe. Rule behind rows 13 to 15: **no view of the base means never write and never delete** (Condition 1). While any previously seen segment is missing, planning is report-only for every node (6.6). Guards: before replacing or removing any mirror file, re-check it still matches B and move the old file to the local trash (path, age and size caps in 8.6; plaintext, same exposure as the mirror; the vault itself never deletes in the alpha), in the order of 7.2.2. A **sentinel file** in the mirror root holds the vault id and a mirror id; if it is missing (folder moved, drive unplugged, user wiped it) the agent pauses and asks, and never reads "everything is gone" as "delete everything". A plan that would delete more than half the files (when the vault holds at least 10) or more than 200 [judgement] stops and needs `cloudcrypt sync --allow-deletes` or one button; deletions that pair with a rename (7.2.1) are not counted (Condition 2). Files changed within 2 seconds, or changing while read, are retried; locked files (Windows) are retried and reported by name. Rename detection by file identity is part of the plan, not an option (7.2.1; Condition 2). `cloudcrypt sync --plan` prints the plan without acting and is used in the first weeks.

### 7.2.1 Local mirror state and folder rules (Condition 2)

Local state keeps one row per mirror **path**: path -> (node id, base version id, size, mtime, file id, chunk-id digest). The chunk-id digest is the SHA-256 of the file's chunk-id list as last committed or applied. The planner finds files by path and maps them to nodes through this table; a path with no row is "no B" (rows 9 to 12). The table is rebuildable and never authoritative: losing it means "no base", so nothing is deleted.
- **Matches.** "M matches B" means size, mtime and file id equal the row. When the vault head differs from B and stat is equal, the agent also chunks the file and compares its chunk-id digest before any overwrite or removal, so an edit that preserved size and mtime is not overwritten; if the digest differs the row is "M edited".
- **Folders.** A folder is removed only when it is empty after the rows of all its children have run. A child that is live in the mirror but uncommitted, or whose row says edited, wins over the folder's delete: the folder stays, the child is committed, and the folder's delete is not applied.
- **Renames.** Rename and move detection by file id is part of the plan: a path that vanished and a new path with a matching file id, size and mtime (or chunk-id digest) become one `move` record, not a delete plus a create. Paired renames do not count toward the mass-delete guard. A renamed folder of 10,000 files is one `move` of the folder node.
- **Conflict copies** each have their own row (path -> node id, base = that head), which is what lets rows 19 to 22 and row 9 tell a rendered copy from a new user file (Condition 3).

### 7.2.2 Apply order and crash table (Condition 4)

`plainout` writes temp files only inside a reserved folder at the mirror root, `.cloudcrypt-tmp/`, on the same volume so the final rename stays atomic. The poller never descends into reserved names (`.cloudcrypt-tmp/`, the sentinel), the ignore set (7.6) covers them at commit time, and the agent sweeps stale temp files at every start. Order for writing head H over path P (rows 2, 7, 11):
1. Write H to a temp file in `.cloudcrypt-tmp/` and flush it.
2. Write a journal entry in local state (node, P, B, H, trash name) and flush it.
3. Re-check that P still matches B (7.2.1).
4. Move the old file at P to the trash.
5. Compare the trashed file with B (chunk-id digest). If it differs, the user saved between steps 3 and 4: move it back to P if P is empty, delete the temp file, and re-plan the node as "M edited" (row 5, so it commits as a conflict); if P is occupied again, leave the trashed file in the trash, list it in `status` as "saved copy kept in trash", and re-plan.
6. Rename the temp file into place.
7. Update the row's base to H, flush local state, clear the journal entry.

A removal (row 3) uses steps 2 to 5 and 7 with no temp file. **Crash table** (each line is a test, SC-32): after step 1 or 2, a stray temp file and perhaps a journal entry remain; the start-up sweep removes the temp file and nothing else has moved. After step 3, the same. After step 4, P is empty and the old file is in the trash with an open journal entry: at start the agent runs step 5 for every open entry before planning, so a user edit sitting in the trash is put back and committed instead of being overwritten by row 7. After step 5, step 6 resumes from the journal; because the journal does not name the temp file and the start-up sweep has removed it, the resume rewrites the temp file from H (revision 3, Condition 4). An open journal entry left by a crash at step 2 or 3 is discarded at start-up, since nothing has moved (revision 3, Condition 4). After step 6, P holds H's content and the base is still B: row 17 adopts H as base with no commit. After step 7, nothing to do.

### 7.3 Offline device and sync-client behaviour

A device offline for a month returns, merges everything, and its local edits have `base` from what it last saw, so they become second heads where others edited, never overwrites. Sync clients deliver late, out of order, duplicated, renamed ("conflicted copy"), as empty placeholders, and sometimes truncated; all are handled by 6.1 principle 2, 6.6 and the simulator. Online-only placeholders (OneDrive Files On Demand, iCloud Optimise Storage [recall]) read as empty or slow: the reader treats a read error as "not ready", never as corruption, and the page says "waiting for <provider> to download N files".

### 7.4 Garbage collection and history

Deleting writes a `delete` record; chunks stay, and old versions stay so "restore a previous version" is a read of older records. The first alpha cut ships **without** space reclaim: the vault only grows, a stated limit. Segments are never removed either, so the segment budget of 6.4 is the measured bound on growth (Condition 11). Design, so the format does not change: (a) any reference is a record in a visible segment; (b) the mirror holds the plaintext of everything live, so a missing chunk can be rewritten. A pack is eligible only if no live or retained record references it, it was unreferenced at two observations at least the fence apart (default 14 days [judgement]), and the sweeper re-merges all segments immediately before deleting. First it writes an `intent` record naming the packs; writers never reuse a chunk from a pack named in a visible intent (they write it again into a new pack). A device offline longer than the fence repairs before publishing: its commit checks that every pack its records name still exists and re-writes missing chunks from the mirror. Residual: a device that publishes in the seconds between another device's final re-merge and delete; the damage is then a detectable missing chunk that the mirror heals on the next scan, not silent loss. This replaces today's races ([review] architecture/gui-gc-races-in-flight-writes, vault-sync/gc-collects-chunk-reused-by-inflight-put).

### 7.5 What the user sees

The top state reads "Saved to your vault folder; your sync client carries it", not "Up to date", which would suggest the provider already has the files (Condition 10). Status states: Saved to your vault folder; Syncing N files; Waiting for <provider> to deliver N files; Waiting for <app> to finish with <file>; N files are not yet in the vault (committing, skipped or retrying); N files exist only on this device (6.6); CloudCrypt is not running, files are not being protected (stale heartbeat, 5.1); Paused: mirror folder missing (sentinel); Paused: too many deletions, confirm; Paused: the vault folder is missing files this device saw before (report-only, 6.6); Locked: enter your password; Locked: new vault key, enter password (the key was replaced on another device); Disk full on this device; Provider full (the vault folder cannot be written); This device needs a newer CloudCrypt (read-only); An older vault is being served by your provider; Attention: N conflicts (with the list); Skipped: N files (with the list and a reason each, 7.6). Each message says what happened, why, and what to do next (the style of review strength 49).

**Which states stop writes** (Condition 7). These stop writing to the vault folder: both Locked states, all three Paused states, "An older vault is being served", "This device needs a newer CloudCrypt", "Disk full" and "Provider full"; the Paused states and report-only also stop mirror writes. These only warn and work continues: conflicts, Skipped, waiting for the provider or an app, N files not yet in the vault, N files exist only on this device.

**Notices and the last good sync** (Condition 10). The notifier fires for: agent not running (from the 15-minute check job, J1), either Locked state, any Paused state, needs a newer CloudCrypt, disk full, provider full, and a new conflict; at most one notice per state per 6 hours [judgement]. It also warns when `lastGoodSync` is older than 3 days [judgement]: "nothing has been saved for 3 days".

**Getting out of a paused or rollback state** (Condition 7). `cloudcrypt doctor --accept-folder` (a page button from M5) shows what the device last saw and what the folder shows now, requires a typed confirmation phrase and a fresh unlock with the password or recovery key, then resets the anchors and the seen-segment set to the folder's authenticated state. For a key file that cannot be authenticated (for example a planted one), it records that file's hash in local state as "set aside by the user" (revision 3, Condition 7): every later Refresh ignores that file, and `status` lists it with the date it was set aside. A set-aside file that later authenticates (for example once the user types a newer password) leaves the set-aside list. It is the old design's rollback ceremony, renamed and kept as the exit.

### 7.6 What the mirror ignores, refuses and sanitises (Condition 5)

- **Ignore set**, fixed and built in, applied at commit time only (a file already in the vault is never hidden by it): the agent's temp folder and the sentinel, `.DS_Store`, `Thumbs.db`, `desktop.ini`, Office lock files named `~$*`, and partial-download suffixes (`.crdownload`, `.part`, `.tmp`) [judgement for the list]. The ignore set is an input to the plan function (row 18) and is table-tested.
- **Skip report.** Everything skipped is listed by name in `status` with a reason (ignored, refused kind, busy, too large, name refused), so nothing is silently left out.
- **Rate limit.** At most one commit per file per 60 seconds [judgement]; a file that has not been quiet after 10 minutes is skipped as "too busy" and retried at the next full re-check.
- **File kinds.**

| Kind in the mirror | Treatment |
|---|---|
| Regular file | committed |
| Hard link | read as a plain file; the link is not preserved |
| Symlink, junction, device, socket, pipe | refused, named in the skip report (SC-11) |
| Extended attributes, resource forks, alternate data streams | not stored; stated as a residual |
| Online-only placeholder | "waiting for the provider" (7.3), never read as empty |

- **Size alerts.** A single file over 2 GiB [judgement] asks before its first commit; a vault folder over 50 GiB [judgement] shows a size note in `status`; a file whose chunk list would exceed the record cap (6.4) is refused with a named skip.
- **One sanitiser** for names and device labels at materialization: control and bidirectional-override characters, path separators, `..`, trailing dots and spaces, and length limits are handled in one function that is fuzzed (11.2); the original name stays in the record.
- **Materialized modes.** The mirror's files are created mode 0600 (0700 with the executable bit) and folders 0700 on Linux and macOS, set explicitly, not taken from the umask; on Windows they inherit the mirror folder's access list.

## 8. Keys, unlock and recovery

**8.1 Create.** Generate K_0 and a recovery key; write one key file (create-only) with a password slot and a recovery slot; require the typed read-back of two recovery groups before anything is written ([review] strength 21 describes today's read-back discipline and cites recovery_phrase.go:76-83, which I did not open) (Condition 14); store K_0 in the keychain; register the vault with the agent.

**8.2 Join.** Detect key files, take the vault id from the highest-generation file, unlock with the password or recovery key, store the key, start the first sync. Wrong password gives one generic error, no oracle (review strength 16).

**8.3 Change password.** Any unlocked device writes a new key file (`fileGen` + 1) wrapping the same vault key under the new password. Other devices need nothing, since they hold the key in the keychain. The recovery slot is bound only to immutable header fields (6.3), so this change copies it forward and does not need the recovery key (Condition 11). Two concurrent changes make two files with the same `fileGen`; both passwords work until the next change (stated residual). The old password still opens old key files; so does the old recovery key after a recovery-key change (Condition 6).

**8.4 Forgot password.** The recovery key unlocks; the user sets a new password; a new recovery key is offered (default yes). The old recovery key still opens the old key files in the vault folder and its history; a new recovery key does not retire it, and only key replacement (8.5) does that for future data (Condition 6). With neither, the data is unrecoverable and says so before creation.

**8.5 Lost or stolen device.** A single symmetric key cannot be revoked from a device that already holds it. "Replace vault key" (M7) therefore: asks for the current password; requires a **new password and a new recovery key** (otherwise anyone who saw the old password could read the new key file from the synced folder); creates K_(g+1) with a fresh random key id (Condition 9); writes a key file whose slots wrap K_(g+1) only, and whose `history` block seals K_0 to K_g under K_(g+1), each with its key id beside it (revision 3, Condition 9) (so new devices read old data, while the lost device's old key cannot open the new one); and uses K_(g+1) for all new packs and segments. The lost device keeps reading all data it already had and everything written before the replacement; the sync provider's version history also keeps the old ciphertext. An optional "re-encrypt old files too" background rewrite (M9, default off, with a cost estimate) can close the first gap. Two replacements run within one sync delay can fork the key. Because keys are named by key id and not by the counter, a fork is detectable: the **winner** is the key file with the highest `fileGen`, ties broken by the larger file hash, so every device derives the same winner, and the next key file written chains the loser's key into its `history` block so data written under it stays readable (Condition 9). The pre-check ("another device already replaced the key") makes a fork rare, and a device that cannot read the other's data re-uploads from its mirror under the winning key (conditional on the mirror holding the plaintext).

**Stale writers and the sleeping device** (Condition 8). Every refresh lists `keys/`; a running or sleeping agent that sees a higher key generation stops writing, shows "Locked: new vault key, enter password" and notifies, and `Commit` fails closed the same way (5.2). A short window remains between the replacement and the next refresh (one poll interval, or the length of a sleep) in which an old-key segment can still be written; it is valid data under the old key and every device reads it through the history chain.

**Write authority of a retired key: OPEN, Owner Ruling A (section 14)** (Condition 9). A thief or a forgotten device that still holds K_g can write valid segments under K_g after the replacement, and other devices would accept them. Two ways to settle it, for the owner to choose: (1) a must-understand `rekey` record carrying a cutoff map (the highest sequence seen per device tag at the replacement), after which readers quarantine retired-generation segments above the cutoff or from unlisted tags (the review recommends this, since layouts are cheapest to change now; it needs a format field and rules in 6.5 and M1); or (2) state the limit plainly in J7 and C8, that a lost device keeps write authority over its key generation until the sync account is cut off. Revision 2 keeps provisional option (2) wording in J7 and C8 and does not choose.

**8.6 Local files: placement, limits and versions (Conditions 6, 15).** One rule: every file on this device that can hold plaintext or a secret lives on a per-machine, non-roaming path, is written only through `plainout` (or the log sink), and passes the T5 placement check (not inside the vault folder, the mirror, a detected sync root or an ancestor of one).
- **Where.** macOS `~/Library/Application Support/CloudCrypt`; Windows `%LOCALAPPDATA%\CloudCrypt` (never `%APPDATA%`, which roams); Linux `$XDG_STATE_HOME/cloudcrypt`, default `~/.local/state/cloudcrypt` [recall for all three]. The run directory (5.4) is separate. Holding plaintext or file names: the trash, the mirror state table and journal (paths are names), the logs, restore destinations (chosen by the user and placement-checked) and the recovery-key file if the user saves one. `status.json` holds counts and reason codes only (5.1).
- **Logs.** No file names above debug level; rotation at 1 MiB with 5 files kept [judgement].
- **Trash.** 30 days and a size cap of the smaller of 5 GiB or 10 percent of free space [judgement], oldest evicted first, never an entry named in an open journal (7.2.2).
- **Free-space floor.** When free space on the mirror's volume drops below 1 GiB or 2 percent [judgement], writes from the vault into the mirror stop, the trash is purged first (entries that hold a copy saved by 7.2.2 step 5 are purged last; revision 3, Condition 6), and status shows "Disk full on this device"; reads are not blocked. When the vault folder cannot be written, status shows "Provider full" and files stay in the "not yet in the vault" count (7.5).
- **Removal.** J8 purges the trash, local state, logs, status file, keychain items, login entry and check job.
- **Disk cost** is the mirror plus the vault folder plus history plus the trash (section 0, item 7).
- **Sync-root claim.** "Plaintext never enters a synced place" holds only where the sync-root detector recognises the place; the marker table is an M1 deliverable, unknown clients are a residual named in the first-run text, and OneDrive can redirect Documents and Desktop (C5).
- **Versions.** The registry (the vaults this device knows: vault id, vault folder, mirror, name), the state table, the merged-tree cache and `status.json` each begin with a schema version number and are written atomically (temp file, flush, rename, flush folder). A file with a newer version than this build understands is not written, and status says "This device needs a newer CloudCrypt"; an older version of a rebuildable file (state, cache) is discarded and rebuilt, and the registry is migrated by code shipped in the same release.
- **Joining into a non-empty mirror.** Rows 10 and 11 of 7.2 apply: nothing is overwritten, a file with different content at the same path becomes a new node, and the first sync shows the plan and asks once before acting.

## 9. What comes from the predecessor, and what this repository starts with

*(Restated in revision 5, 2026-11-14, by the main Claude session, at the owner's request, for this repository. The earlier section 9 planned the rewrite inside the private predecessor's repository; it is kept in the private original.)*

This repository starts empty: no predecessor code is in it, and nothing here is tagged or deleted. The predecessor stays unchanged in its own private repository. Counts are non-test Go lines from the original author's script over the predecessor's working copy on 2026-11-14: 22,683 source and 13,565 test lines in total. The bullets below come from the earlier section, the first seven verbatim and the last adapted, so "I" in them means its author.

**Porting rule, an owner decision before M1 code.** The predecessor's files are not published and are not under this repository's license. Copying any of them here would publish them under this repository's license, which only their rights holder can allow, so a piece is ported only with that holder's written permission, noted in the commit that brings it in; otherwise it is rewritten from its description here, without copying. Ideas and test designs carry over either way [judgement for this paragraph; not legal advice]. Rewriting the roughly 1,200 lines below instead of porting them adds about one to two weeks to M1 [judgement].

**Worth carrying over, by porting or rewriting (about 1,200 lines):**
- `internal/vault/chunker.go` (97): the splitter; keep, change sizes (chunker.go:15-21); lines 43 and 53 show the per-file 16 MiB buffer that the new sizes shrink ([review] 6.6).
- `internal/vault/portable.go` (100) into `names`; `internal/vault/path.go` (43) as a source of the rejected-segment rules.
- `internal/vault/rename.go` (82), `rename_other.go` (9), `rename_windows.go` (31), `fsync_unix.go` (23), `fsync_windows.go` (9), and `atomicWriteFile` at `vault.go:1469-1510` (42 lines): into `platform` atomic write, with the missing parent-folder flush added.
- `internal/vault/recovery_phrase.go` (83): the read-back and constant-time compare; encoding changes.
- `internal/passphrase` (279): hidden prompt with its Windows table test ([review] strength 67, citing classify.go:53-68, which I did not open).
- `internal/keychain` (369): the three back ends, changed to an injected interface with one conformance table ([review] 5.13).
- `internal/loopback` (77): the Host allowlist for the page helper.
- Ideas, not code: injectable clock and seen store ([review] strength 44, citing gc.go:42-55, which I did not open); the device freshness anchor ([repo] anchor.go:46-51); the id-bound AEAD and id recheck (review strength 15); the leak walk ([repo] leak_test.go:23); the fuzz habit; the 0700 and 0600 mode discipline of appdir.go:89 and :101; the test discipline of the predecessor's CLAUDE.md, written into `CONTRIBUTING.md` here. Test ideas to carry over as scenarios: tamper tests, rollback tests, the compact race seam, `testmain` app-data isolation (review strengths 79, 80, 84; I did not open those files).

**Possibly carried over (decision 3):** `internal/xcrypto/argon2` (516) and `blake2b` (482) with their vector tests; not needed if the owner allows `golang.org/x/crypto`. Otherwise they come under the porting rule or, if they are copies of the Go project's own code (not checked here), from that code under its own license; crypto code is not rewritten from scratch [judgement].

**Left behind (about 21,480 lines):** everything else, including the browser GUI, WebDAV, rclone and rsync management, SSH keys, remote profiles, process guarding, vault moves, the old command-line tool (2,411 lines; its replacement is about a third the size [judgement]) and the old format code, with about 13,250 of the 13,565 test lines. None of it comes here.

**New code** [judgement]: about 10,000 to 13,000 lines of Go plus a small embedded page, about half the predecessor's non-test size, with tests larger than its ratio.

**What M1 creates, for every top-level file and `docs/` entry** (Condition 15). Register: judgement.

| Path | At M1 |
|---|---|
| `README.md`, `LICENSE` | exist; README.md grows with each milestone |
| `go.mod`, `go.sum`, `vendor/` | create: module `github.com/alexdimarco/cloudcrypt` at `go 1.26`, with the three allowed modules vendored (decision 3) |
| `cmd/`, `internal/` | create: the package skeleton of 5.2 with the allowed-edges test |
| `Makefile` | create: `make test` (`go test ./...` with no filters) and `make vectors` |
| `.github/workflows/` | create: the three-operating-system required jobs of 11.5 on GitHub's hosted Linux, macOS and Windows runners (decision 8), plus the nightly fuzz and benchmark jobs. Because this repository is public: only GitHub's own checkout and Go-setup actions, pinned to full commit ids and listed in `docs/dependencies.md`; a read-only token; no secrets [judgement] |
| `CONTRIBUTING.md` | create, before the first code commit: section 12's process rules for every contributor (every fix proven red, then green; no gate weakened, no test deleted, no suppression, no fixture edited to make a test pass; the review gates), so they no longer live only in the predecessor's private files |
| `CLAUDE.md` | create: a short working agreement for AI agents that points to `CONTRIBUTING.md` |
| `SECURITY.md` | create from section 4 (T1 to T7), with a private way to report a flaw |
| `THIRD_PARTY_NOTICES.md` | create: the three modules, and any ported code that needs a notice |
| `.gitignore` | create |
| `docs/design-alpha-architecture.md`, `docs/review-alpha-architecture-predesign.md` | keep as the record |
| `docs/format-spec.md`, `docs/scenarios.md`, `docs/dependencies.md` | create (11.1, 11.3, 11.5) |
| `PROMPT.md` or another purpose statement | Owner Ruling B: whether the predecessor's founding document comes here, edited, under the porting rule, or a short purpose statement replaces it; README.md states the purpose until then |

## 10. The review's findings against this design

I read review sections 1, 2, 5, 6 (3 to 6.4), 7 and the P0 and P1 headings of section 4; I did not open the P2 and P3 entries (4.3, 4.4), so the pre-code review owes a per-id pass for them.

**Designed out on paper, conditional on the tests and conditions named on each line** (the mechanism is absent, or an invariant is the only way to write; nothing is proven until the M2 and M3 tests pass, section 13) (Condition 14):
- (C2, C3) Blind overwrite: vault-sync/blind-manifest-write-stale-snapshot, perf-scale/serve-stale-index-overwrites-remote-edit, webdav/serve-never-reloads-index (write-once files; `Commit` needs a planned snapshot and returns `ErrStale`; one agent keeps the view fresh; `base` from the mirror's last-seen version).
- (C4) vault-sync/pending-conflict-copy-destroyed-by-next-save (7.1, plan rows 19 to 22).
- (removal; no condition) Remote copies: cli-ux-docs/remote-pull-push-overwrites-edits-and-config, external-tools/pull-push-overwrite-loses-edits, vault-sync/remote-pull-overwrites-unpushed-edits (no pull or push; every incoming byte arrives through the sync client and the one merge).
- (removal; no condition) platform-secrets/move-replace-deletes-aliased-source, go-modernization/vault-move-replace-deletes-source, platform-secrets/move-copy-fallback-unverified (no move-with-replace exists; relocation is "re-point"; no code deletes a destination).
- (removal; no condition) webdav/move-onto-parent-deletes-tree and the review 5.6 namespace items and the WebDAV P1s (architecture/gui-webdav-private-mutex, go-modernization/webdav-copy-move-refused-name-deadlock, webdav/copy-pipe-wedges-server, go-modernization/webdav-path-decoded-twice, webdav/path-decoding-aliases-names, webdav/put-during-move-loses-upload): no WebDAV; rename is one record; the OS does mirror moves and we only observe them.
- (T2 and T4 tests; C11, C12) platform-secrets/launch-secret-in-process-arguments, webui-backend/launch-secret-in-argv-multi-use (no secret in any argument; the single-use code exchange of section 5.1; T3 in section 4).
- (C2, C3) vault-crypto/stale-config-lost-update, architecture/stale-config-snapshot-reverts-rotation (key files are immutable; newest generation wins).
- (SC-04 and the M1 range tests) vault-crypto/kdf-params-unbounded, go-modernization/kdf-numbers-from-vault-json-unbounded, vault-crypto/vaultid-path-traversal, vault-crypto/rotation-inherits-untrusted-kdf, vault-crypto/init-replaces-only-key-file, tests-assurance/manifest-name-parser-crash (range-checked header; ids are fixed-width bytes, never paths; create-only).
- (M1 vector and tamper tests, C13) vault-crypto/id-hmac-domain-collision, configmac-struct-remarshal, untagged-config-any-shape, seal-keyless-unseal (HKDF labels; AEAD over exact bytes; no untagged form).
- (C4, SC-03) vault-sync/cloned-device-id-silent-drop and platform-secrets/cloned-device-id-silent-overwrite (a fork is data); vault-sync/mixed-clock-comparator-not-transitive (no clock comparator decides data).
- (C12 for the page; removal for the rest) webui-backend/session-cookie-leaves-the-port, get-executes-named-binary, zip-download-silent-truncation, jobs-outlive-close-and-die-on-shutdown, webui-frontend/xss-filename-inline-handlers, webui-frontend/recovery-id-from-untrusted-config (no cookie, no binary execution, no ZIP route; agent jobs carry contexts; `textContent`).
- (removal; no condition) The whole external-tools theme (review 5.12 and 5.5's download items, including external-tools/rclone-checksum-not-bound-to-signature): the programs are gone.
- (no deleting code in the alpha; the later design is conditional, C9) architecture/gui-gc-races-in-flight-writes and vault-sync/gc-collects-chunk-reused-by-inflight-put: the first alpha cut has no deleting code; the later design is 7.4, conditional.
- (C5, C2) go-modernization/plaintext-guard-compares-path-text and platform-secrets/no-vault-owner-stale-handles-write-ghost-tree, architecture/gui-session-lifecycle-not-modelled: `plainout` (SameFile) and one owner with a context; conditional on the tests in section 4.

**Moot without compatibility:** legacy wraps and "rotation after seal restores the legacy wrap" (P1-20), seal and unseal, the `content/` migration, both metadata-folder names, mixed clocked and clockless records, scrypt and PBKDF2 hygiene, go-modernization/go-1-23-unsupported-toolchain (the minimum moves, 11.6), tests-assurance/interop-016-test-uses-015-binary and the skipped cross-version tests, the usage-drift and duplicate-flag items (the CLI is replaced), cli-ux-docs/remote-failure exit codes (no remote ops: go-modernization/cli-remote-failure-exits-zero, tests-assurance/remote-ops-exit-zero-on-failure). The `--accept-rollback` ceremony is not moot: it is kept as `cloudcrypt doctor --accept-folder`, the exit from a rollback or forgery state (6.6, 7.5; Condition 7).

**Still need care** (reduced, not removed):
- Mirror sync correctness: half-written saves, rename detection, locked files, mass deletes, case and Unicode differences, online-only placeholders. The largest new risk; section 7.2 and the simulator.
- Scale (review 5.8; vault-sync/index-costs-grow-with-vault-size, perf-scale/whole-index-deep-copy-per-call, external-tools/large-import-stale-and-quadratic): packs and immutable shared snapshots fix the shapes (review 6.3 and 6.8); log growth, memory and the macOS poll cost need measured budgets (11.4).
- Tests that touch the real keychain ([review] webui-backend/tests-reach-real-keychain): one injected interface and one shared test main that points app data at a temp folder.
- CI on three systems: today only Linux runs automatically ([review] section 2) although CLAUDE.md says all three; needs runners (decision 8); tests-assurance/green-does-not-mean-ran needs the must-run list (11.5).
- No fuzz targets (tests-assurance/no-fuzz-targets), spec drift (cli-ux-docs/normative-docs-describe-an-older-system; DESIGN.md is today's normative text): 11.1 to 11.3.
- Windows file modes protect nothing (platform-secrets/windows-file-modes-not-protection): the run directory relies on the profile access list; a Windows runner must test it.
- Nextcloud and hidden folders ([repo] drive-review:35-37, P0-4: `.cloudcrypt` is not synced by default): solved by the visible `CloudCrypt` folder; still needs a real Nextcloud drill, pending.
- Error taxonomy, exit codes, logging (review 5.16): one error-to-text function, one table of commands, one log sink from the first commit.

## 11. Supportability

### 11.1 The format specification

`docs/format-spec.md`, numbered rules F-001 and up, one test per rule citing its number; a test fails when a rule has no test or a test cites no rule (review 5.17 proposal). `DESIGN.md` is rewritten as a short pointer to the spec and this design, since it has been stale since the first commit ([review] cli-ux-docs/normative-docs-describe-an-older-system). The independent decoder and golden vaults (6.9) prove the spec is followable without the Go code.

### 11.2 Fuzzing

Go native fuzz targets: key file, pack, segment outer and inner decode (inner with a fixed key so the fuzzer reaches the parser), record decode, `names` parse and collision key, the tree merge (random permutations, duplications and truncations of record sets must give identical trees), state-file decode, and the one name sanitiser of 7.6 (revision 3, Condition 5). Rule for every parser: typed error, never panic, no allocation above a stated bound. CI runs each target 30 seconds per change and 10 minutes nightly [judgement]; corpora are checked in. Today's fuzz targets ran clean at 16.2 million inputs ([review] strength 15): weak evidence of parser robustness, not proof. The new formats start with no corpus, and each decoder gets a coverage or mutation signal (lines reached, or a planted bug the fuzzer must find) beside its run count (Condition 14).

### 11.3 The scenario catalogue

`docs/scenarios.md`: rows SC-01 and up, each with the situation, the expected result, the test name, whether it runs on real files or the simulator, and a **"break it by"** note saying how to neutralise the guarded code so prove-fail is repeatable under the repository's rule (CLAUDE.md: neutralise the fix, see the right red, restore, see green). A test parses the file and fails on a row with no test or a test with no row. Seed rows: two devices edit one file (SC-01); delete against edit (SC-02); cloned device tag (SC-03); hostile key-file Argon2 numbers (SC-04); segment before its pack (SC-05); truncated pack (SC-06); sync client renames to "(conflicted copy)" (SC-07); empty placeholder (SC-08); duplicate delivery (SC-09); mirror inside the vault (SC-10); symlink in the mirror pointing out (SC-11); names differing by case (SC-12); Windows reserved name (SC-13); file changes during a scan (SC-14); crash between pack and segment (SC-15); lost local state (SC-16); older folder served after a newer one (SC-17); mirror folder missing (SC-18); mass-delete guard (SC-19); password change while another device runs (SC-20); no plaintext anywhere in the vault folder (SC-21); 10,000-file import within budget (SC-22); folder cycle by concurrent moves (SC-23); rename plus edit (SC-24); recovery key (SC-25); wrong password gives no oracle (SC-26). Revision 2 rows (condition in brackets): half-delivered restore (SC-27, 1); sync client drops a seen segment (SC-28, 1); folder rename of more than 200 files, and an edit that keeps size and mtime (SC-29, 2); conflict resolution, all four cases (SC-30, 3); temp-file feedback loop (SC-31, 4); crash injection after each apply step (SC-32, 4); junk, locked, hot and special files and the skip report (SC-33, 5); recovery-key file refused in the vault folder, the mirror and a sync root (SC-34, 6); planted forged key file, which `doctor --accept-folder` sets aside so the next Refresh no longer shows "Locked: new vault key", and which leaves the set-aside list if it later authenticates (SC-35, 7; revision 3); device running or asleep across a key replacement (SC-36, 8); key fork and its winner (SC-37, 9); kill the agent mid-commit (SC-38, 10) and mid-apply (SC-39, 10); agent and CLI of different builds (SC-40, 12); lost pack healed from the mirror (SC-41, 15); late parent and orphan hold-back (SC-42, 15); write with a retired key (SC-43, 9; its shape depends on Owner Ruling A); segment budget and start-up at one year (SC-44, 13). Each gets a break-it-by note when written. The plan matrix also has a property test that fails on any cell without an action (Condition 1). Review P0 scenarios and J1 to J8 are added in M2 and M6.

### 11.4 The simulator, invariants and scale budgets

`internal/sim` is a fake provider and sync client between two or three simulated agents: delay, reorder, duplicate, drop then deliver, conflicted-copy rename, truncate, empty placeholder, clone, offline gaps. It runs with the race detector on Linux and states five invariants checked after every scenario: (1) no version is lost; (2) every device that has all files shows the same tree; (3) no plaintext in the vault folder; (4) a sweeper (M8) never deletes a referenced pack; (5) a replayed older folder is reported. Scale budgets (my judgement, from review 6.3 where 2,000 small files took 40.8 s to put): 10,000 small files imported in at most 30 seconds on macOS; an idle full stat-walk of 100,000 files in at most 1 second of processor time, so that with the derived idle interval (5.5) the duty cycle stays under 1 percent; start-up at 100,000 segments in at most 10 seconds (6.4) (Condition 13); under 150 MiB memory at 200,000 files; a nightly CI benchmark fails over budget.

### 11.5 CI, dependencies, Go version

- **CI on three systems.** Required jobs on Linux, macOS and Windows: format, `go vet`, `go test ./...` with `CGO_ENABLED=0`, import-graph test, vector test, scenario-catalogue test, fuzz smoke, an end-to-end run of the built binary against temp folders. Race detector on Linux (it needs cgo [recall]). A **must-run list** makes a skipped suite fail instead of pass. No test may reach the real keychain or real app data. Needs runners the project does not have today (review section 2; decision 8).
- **Dependency policy.** Standard library first. Allowlist: `golang.org/x/crypto` (Argon2id), `golang.org/x/sys` (file lock, Windows calls, watchers later), `golang.org/x/text/unicode/norm` (NFC). Each is pinned in go.sum, vendored, listed in `docs/dependencies.md` with a reason and an exit plan; a test fails if go.mod changes without a matching entry. Anything else needs an owner decision. Patch owner: the maintainer named in `docs/dependencies.md` watches security releases of the three modules and of Go (5.4) (Condition 12). This ends today's zero-module rule ([review] go-modernization/handwritten-crypto-and-zero-module-policy; go.mod lists no modules, [repo] go.mod:3 says `go 1.23`). cgo is forbidden in the core.
- **Go version.** `go.mod` states the older of the two Go releases the Go project supports [recall for the policy]; today the review shows 1.26.0 selected and 1.27.2 installed, both building and passing (review strength 76), so the line is `go 1.26`. CI runs the oldest and the newest; a named owner checks the Go release page at each release. Needed features: `os.Root` and `crypto/hkdf` (1.24 [recall]). Newer Go releases raise the minimum macOS and Windows versions they run on [recall]; that is the real cost.

## 12. Build plan to a usable alpha

Effort scale (my judgement): S under a week, M one to three weeks, L three to six, XL six to twelve, for one maintainer. Process gates, from CLAUDE.md: the 10-lens pre-code design review for this design before any code; an adversarial pass for key handling and parsers; a friction review for every shippable feature; every fix prove-fail then prove-pass; no gate weakened. "Usable alpha" means M1 to M6, about 25 weeks of effort [judgement], about six months; a CLI-only owner-usable cut exists at M4.

**Gate 0 (no product code): the pre-code review and three spikes.** Owes the 10-lens pre-code review (docs/review-alpha-architecture-predesign.md). Spikes, each a throwaway program reported in the review: (a) Unix socket on Windows and the peer-user check on macOS; (b) a `file:` page posting to `127.0.0.1` in Chrome, Firefox, Safari and Edge; the terminal link is the primary path and the `file:` page is built only if this passes in all four (Condition 12); (c) the macOS stat-walk cost on a 50,000-file tree, a numeric gate: the walk must cost at most 0.5 seconds of processor time, which gives an idle interval of 50 seconds at under 1 percent duty cycle (5.5); a result over 1 second moves the watcher work (M9) before alpha use (Condition 13). Also: owner decisions 1 to 6, 8 and 13. *(Note beside the original, 2026-11-14, by the main Claude session, revision 5: in this repository decision 13 is not needed (see its note), and section 9's porting decision joins this list.)*

| # | Milestone | Scope | Tests red-first | Review gates | Effort | Depends on |
|---|---|---|---|---|---|---|
| M1 | Foundations and format kit | below | below | adversarial pass (key handling, parsers) | L | Gate 0, decisions 3, 4 and 8, section 9's porting decision (revision 5; decision 13 is not needed here) |
| M2 | Tree and commit | `tree` merge with permutation property tests; `engine`, `store`, Commit/ErrStale, segment batching, reader; the anchor half of `state` (anchors, seen-segment set, vault-folder sentinel, merged-tree cache; Conditions 1, 7, 13, 15); inline small chunks, batching floor and caps; segment-budget and start-up measurements (Condition 13); `internal/sim` and the five invariants | SC-01, 02, 05, 06, 09, 15, 17, 23, 24, 27, 28, 35, 44 | 10-lens delta check; adversarial pass (data loss) | L | M1 |
| M3 | Mirror engine | poller, the mirror half of `state` (path table, journal), quiesce, the total plan matrix (7.2) with its property test and table test, apply order with crash table, trash, sentinel, mass-delete guard with rename pairing, ignore set and file-kind table (7.6), conflict rows 19 to 22 and `conflicts --keep`, `plainout` and placement guard with the sync-root detector wired in (marker table from M1), name and label sanitiser, Windows retry, `sync --plan`, heal step and `doctor --repair` (Conditions 1 to 6, 15) | SC-10 to 14, 16, 18, 19, 21, 22, 29 to 34, 41, 42 | adversarial pass (path handling, plaintext output); friction review of conflict and lock messages; macOS poll cost measured | XL | M2 |
| M4 | Agent, CLI, unlock | single instance, autostart with restart policy and the 15-minute check job, socket, handshake and client ownership checks, keychain, spawn helper, minimal notifier (pulled forward from M9), `status.json` heartbeat and dead-agent status, the fifteen commands and the command-line contract (5.1), recovery (J6) (Conditions 10, 12, 15) | SC-20, 25, 26, 38 to 40; secret-in-argv test | adversarial pass (key handling, process guarding, local channel); friction review (CLI) | L | M3 |
| M5 | Setup and status page | `cloudcrypt-ui`: single-use code exchange (terminal link primary, 5.1), J2, J3, J5, J6, J8 screens, status, conflict list | route-table test; Host table; no-inline-script test | friction review; adversarial pass (web surface) | M | M4, spike (b) |
| M6 | Packaging and three-OS gates | unsigned installers or install scripts, autostart, required CI on three systems, real-OS journey tests J1 to J6 and J8, real drills on Nextcloud, Dropbox, OneDrive (marked pending until done) | journey tests | friction review of the full install-to-use path; must-run list green on all three | M | M5, decision 8 |
| M7 | Lost device | key ids and generations, `replace-key`, re-prompt on other devices (J7), concurrent-replace pre-check, fork winner and history chaining, the Owner Ruling A mechanism (the `rekey` cutoff record, if chosen) (Condition 9) | SC-36, 37, 43 | adversarial pass (key handling) | M | M6 |
| M8 | GC and checkpoints | `intent` records, fence, sweeper, checkpoints for load time | invariant 4 | 10-lens delta review (data-deleting) | L | M7 |
| M9 | Watchers, tray, re-encrypt sweep | inotify and `ReadDirectoryChangesW`; tray helper in its own module; optional re-encrypt of old files | watcher parity with polling | friction review | M | M8 |
| M10 | Optional materializers | Linux FUSE first; Cloud Files and File Provider as separate designs | | own 10-lens review each | XL each | M9 |
| M11 | Signing and installers | Apple Developer ID and notarization; Windows code signing; real installers (a beta gate, not alpha) | | friction review | L | decision 9 |

### M1 in detail (so its own design can start)

*(Restated in revision 5, 2026-11-14, by the main Claude session, at the owner's request, for this repository: items 1, 2, 4, 8 and 11 and the exit changed, and the rest stands. The earlier text is kept in the private original.)*

**Goal.** A library and a `doctor` command that create a vault in a folder, write and read sealed key files, packs and segments, and prove the format against hostile input, on three operating systems.

**Scope.**
1. The repository files of section 9's table, with `CONTRIBUTING.md` in place before the first code commit; module `github.com/alexdimarco/cloudcrypt` at `go 1.26`; package skeleton of 5.2 with the allowed-edges test; `docs/dependencies.md`. Nothing is tagged or deleted: no predecessor code is here.
2. `seal`: Argon2id (from `x/crypto` if decision 3 allows, else the copy section 9 describes), HKDF, AES-256-GCM per-file keys, nonce counters, constant-time compare.
3. `wire` and `format`: key file, pack, segment and record codecs with the caps of section 6; the version, `minReader`, `minWriter`, feature lists; typed errors.
4. `names` (NFC, simple fold, the portable rules of the predecessor's portable.go, ported or rewritten under section 9's rule); `platform` atomic write with the parent-folder flush and Windows retry.
5. `store` folder implementation with `PutNew` (create-only by temp file plus rename-if-absent semantics; the Windows and network-share behaviour is to be checked [recall]).
6. `docs/format-spec.md` first draft with F-numbered rules, `make vectors`, the independent decoder skeleton.
7. `cloudcrypt doctor` reading a vault offline: lists key files, segments, packs, anything misnamed or damaged.
8. Three-OS CI on GitHub's hosted Linux, macOS and Windows runners, set up as section 9's table says, running the import-graph, vector, scenario-catalogue and fuzz-smoke jobs with the must-run list of 11.5.
9. The sync-root detector's marker table, as data with a test, for the owner to check; M3 wires it into `plainout` (Condition 15).
10. The format items that must exist before layouts freeze: the `raise-min-writer` record, the writer's format level in the sealed segment header, key ids, the slot associated-data rules of 6.3, the recovery key's version character and fingerprint with its test vector, and the refusal of other format minors (Conditions 9, 11).
11. Section 9's table applied: every path it lists exists, and each piece of predecessor code was either ported under the porting rule or rewritten (Condition 15).

**Tests, red first** (each with a break-it-by note): key file round trip and every-byte-region tamper (SC-04 included); Argon2 numbers out of range refused before work; segment and pack tamper over every region; the pack footer authenticates; a file named by its hash is verified and a renamed copy is read by content; name collision keys; a decoder never panics (fuzz); a planted-plaintext walk over a sample vault finds nothing (SC-21 base); `net/http` and `os/exec` import tests; the vector regeneration test; the allowed-edges test.

**Decisions that M1's design fixes before code:** chunk sizes after measuring; the binary tag-length-value field tags; the key file byte layout; whether PutNew uses hard-link or exclusive rename per OS.

**Exit.** The vectors are written; the independent decoder skeleton reads the small golden vault; CI is green on Linux, macOS and Windows (GitHub's hosted runners; decision 8); the adversarial pass on key handling and parsers is filed.

## 13. Honesty of claims

Labels: **proven** (a test or run in this repository shows it, or I proved it by reading the lines I opened), **conditional** (true if the condition next to it holds), **aspirational** (a goal; nothing shows it yet). No code of this design exists, so nothing about the new system is proven; "proven" below applies to claims about today's code.

| # | Claim | Label and condition |
|---|---|---|
| C1 | Today's writer flushes a file, renames it, and flushes the folder; four full flushes per small file cause about 96% of put time on macOS. | Proven by reading [repo] vault.go:1469-1510; the 96% is review-reported (6.3), one Mac. |
| C2 | A stale write cannot occur because `Commit` needs the planned snapshot and refreshes first. | Conditional on no code path bypassing `engine` (the import test) and on the sync client being an uncontrolled writer only of files we treat as immutable. Aspirational until M2's permutation and simulator tests pass. |
| C3 | No overwrite by the sync client is possible. | Conditional on write-once hash-named files; a client that rewrites file contents in place is detected by the hash check and treated as damaged, not as a new version. |
| C4 | Conflicts keep both versions, and every device shows the same data; conflict-copy **names** may differ across devices, because they are fixed at first materialization. | Conditional on the merge being a pure function (permutation fuzz) and on the same name rules; different Unicode tables across devices, and first-materialization naming, can change cosmetic names (Condition 3). |
| C5 | Plaintext never enters the vault folder. | Conditional on `plainout` being the only plaintext writer (import test) and on the sync-client list being complete for the root check (detection is best effort, and the recovery-key file uses the same check, 8.6; Condition 6). Aspirational until SC-10, 11, 21, 34 pass. |
| C6 | The vault format does not reveal how many devices write to it or which wrote what; the provider can still estimate a device count from timing, and a sync service may know its own account's devices. | Conditional on tags living only in sealed blocks and names being content hashes; the size of an isolated write and activity timing remain visible (6.7) (Condition 14). |
| C7 | Withheld or truncated files from others are detected. | Conditional and partial: only for segments a device has already seen or that another device's seen-list names; withholding the newest of everyone is not detected. |
| C8 | A lost device cannot read data written after key replacement. | Conditional on a new password and new recovery key being used, on the lost device not obtaining the new password, and on no key fork; it can read everything it already held. A short window before the next key-folder refresh remains in which a running or sleeping old device can still write under the old key (8.5; Condition 8). **Write limit, open (Owner Ruling A):** a device holding the old key can write valid old-key segments until the sync account is cut off or a `rekey` cutoff record exists; this is provisional wording pending the ruling (Condition 9). |
| C9 | The mirror never deletes unintended files. | Conditional on the total plan matrix and its property test, the descent precondition and the report-only rule, the sentinel, the base rule, the mass-delete guard, the apply order with its journal, and the trash being implemented as in 7.2 to 7.2.2; the riskiest new code (M3) (Conditions 1, 2, 4). |
| C10 | Mirror polling costs acceptably on macOS. | Aspirational: unmeasured; Gate 0 spike (c) and the budgets in 11.4. |
| C11 | Windows sockets and macOS peer checks work as described. | Aspirational [recall]; Gate 0 spike (a). |
| C12 | A `file:` page can POST to loopback. | Aspirational [recall]; spike (b); fallback named. |
| C13 | The format spec is followable without the Go code. | Aspirational until the independent decoder reads the golden vaults. |
| C14 | The size and effort estimates. | Aspirational; [judgement]. |
| C15 | The review's line numbers and statuses carried here. | Aspirational as the review is language-model self-report (its own warning); I re-opened the lines marked [repo] and carried the rest by id. |
| C16 | Restructuring keeps the review's strengths. | Conditional on the fixture suite and the "no test deleted or weakened" rule (CLAUDE.md). |
| C17 | Nextcloud and other clients sync the visible `CloudCrypt` folder. | Conditional [recall] on default settings; real drills pending (M6). |
| C18 | A dead or stuck agent becomes visible to the person. | Conditional on the heartbeat, the restart policies and the 15-minute check job being registered and not disabled; a person who disables autostart gets no alert (J4). Aspirational until SC-38 to 40 pass; the operating-system facts are [recall] (Condition 10). |
| C19 | Local and vault growth stays bounded in the alpha. | Aspirational: the segment budget, batching floor, caps and start-up target of 6.4 and 11.4 are [judgement] until M2 measures them (Condition 13). |

Things this design does not claim: it removes none of the conceded boundaries; it does not make two devices share a lock; it does not protect against a user with the password and hostile intent; it does not prove anything about the platform behaviours marked [recall].

## 14. Decisions for the owner

1. **Access model: mirror folder on all three systems.** It needs no driver, no administrator prompt, and every app works because it is a real folder.
2. **Accept plaintext at rest in the mirror, with a disk-encryption check and warning.** The alternatives that avoid it need drivers and still cache plaintext.
3. **Allow exactly three external Go modules (`golang.org/x/crypto`, `golang.org/x/sys`, `golang.org/x/text/unicode/norm`), vendored.** NFC has no standard-library form, and hand-owning 998 lines of Argon2 and BLAKE2 buys nothing now; if you refuse, keep the vendored copies and drop NFC to collision detection by simple fold only.
4. **No cgo in the core; an optional tray module may use it.** Cross-compiling stays trivial for the part that touches your data.
5. **Interface: terminal wizard first, then a small page served by a separate `cloudcrypt-ui` helper, then a tray helper; no native toolkit and no browser file manager.** Setup is the one moment a person needs a screen; day to day they use their file manager.
6. **Cut rclone, rsync, SSH keys, remote profiles, WebDAV and scrypt/PBKDF2 completely, and change PROMPT.md accordingly.** The sync client is the transport and each cut piece is a second product with its own supply chain. The lines of PROMPT.md that stop matching (I read the file): 10 (encrypted sharded manifests; this design uses a record log in segments), 12 (scrypt alternative and PBKDF2), 14 (conflicted manifest files and tombstones; now segments and delete records), 15 (the local WebDAV endpoint), 31 (scrypt and PBKDF2 vectors) and 32 (manifest conflict preservation; now conflicts in the record graph). Lines 13 and 33 stay consistent (key wrapping and rewrap; delete precedence in 7.1). Whether to edit those lines or replace PROMPT.md with a short purpose statement is Owner Ruling B and is not applied here (Condition 15).
7. **Go minimum 1.26 with CI on the oldest and newest supported release.** It gives `os.Root` and HKDF in the standard library and ends the unsupported 1.23 pin.
8. **Register macOS and Windows CI runners before M3.** "Three first-class systems" is not true until those tests run on every change. *(Note beside the original, 2026-11-14, by the main Claude session, revision 5: the restated M1 runs CI on GitHub's hosted Linux, macOS and Windows runners, which GitHub's billing documentation says are free for public repositories (seen through a web search on 2026-11-14; the page itself not opened). If that holds, nothing needs registering, and this decision becomes accepting those runners from M1 rather than M3. It stays the owner's.)*
9. **Obtain Apple and Windows signing before beta, not before alpha (M11).** The alpha has one user, you; signing widens delivery without adding safety.
10. **Declare the format frozen at beta, after golden files, an independent decoder and two months of your own use, and only once the upgrade rule is written in the spec (a freeze gate, 6.8; Condition 11).** Pre-alpha freedom is the main thing your correction bought; spend it getting the format right.
11. **Make the recovery key mandatory at creation, with typed read-back.** There is no server to reset a password, so an optional key is a data-loss switch.
12. **Name the vault folder `CloudCrypt` (visible), not a dot-folder.** Some sync clients do not sync hidden folders by default ([repo] drive-review:35-37).
13. **Tag `legacy/0.17.0` and delete the old tree at M1.** Nothing needs migrating and a half-alive old tree invites reuse of its bugs; the tag is a git action only you should run. *(Note beside the original, 2026-11-14, by the main Claude session, revision 5: this repository has no old tree, because the predecessor's code comes here only under section 9's porting rule, so the decision's aim holds without it and Gate 0 and M1 no longer wait on it. Tagging the predecessor in its own repository is the owner's affair, outside this project.)*
14. **Promise only "replace the key, new password, new recovery key" for a lost device, and say it protects future data.** A true remote revoke is impossible with a symmetric key.
15. **Accept that rename-plus-edit of one file on two devices shows as a conflict in the alpha.** It loses nothing and merging placement from content can wait until real use shows it matters.

**Decisions awaiting the owner, from the pre-code review (not applied in revision 2; each as the review stated it).**
- **A. Lost device write authority (Condition 9).** Add a must-understand `rekey` cutoff record (recommended by the review, since layouts are cheapest to change now) or state the limit in J7 and C8. Blocks the final text of 6.5, J7, C8 and the M1 format layout. Revision 2 carries provisional limit wording in J7, 8.5 and C8.
- **B. PROMPT.md is the owner's founding document (Condition 15).** Edit the listed lines (10, 12, 14, 15, 31, 32) or replace it with a short purpose statement. Blocks only the PROMPT.md edit. *(Note beside the original, 2026-11-14, by the main Claude session, revision 5: PROMPT.md is the predecessor's and is not in this repository. The ruling now also covers whether it comes here, edited, under section 9's porting rule, or a short purpose statement replaces it; until then README.md states the purpose.)*
- **C. Decision 5, browser page versus native tray.** Owner's soft constraint; the review recommends deciding after using the M4 CLI cut and building only one. Blocks M5 and M9 only.
- **D. Decision 2, plaintext mirror.** Accepted by the owner; an optional one-line first-run note about backup and indexing tools. Blocks nothing. *(Correction beside the original, 2026-11-14, by the main Claude session: the owner has not ruled on this. Decision 2 is a recommendation awaiting the owner, and it is the largest trade in this design: plaintext of every vault file on every device's disk, and about twice the disk space. If the owner rejects it, the access-model parts change (sections 2, 5.5, 7.2, 8.6 and milestone M3); by this design's own claim, the engine and format do not.)*

## 15. Provenance

**Revision 2 (2026-11-14).** Applies the 15 conditions of docs/review-alpha-architecture-predesign.md (verdict GO_WITH_CONDITIONS), each marked "(Condition n)" where it changed the text. Owner rulings A to D are not applied (section 14). No code exists and none of the added mechanisms is tested; the review's addendum holds the re-check. The rest of the document stands as in revision 1.

**The panel.** Four architects wrote independent proposals; two judges (product-friction and integrity-supportability) scored six criteria each out of 10 (60 per judge). All agents in this design run ran Sonnet 5.5 on 2026-11-14: medium effort for proposals, this design, lens finders, synthesis and revision; high effort for the judges, the skeptic, the rescue pass and the condition re-check.

| Proposal | Judge 1 | Judge 2 | Total (of 120) | First-place votes |
|---|---|---|---|---|
| friction-first (winner) | 47 | 47 | 94 | 1 (judge 1) |
| minimal-supportable (runner-up) | 46 | 48 | 94 | 1 (judge 2) |
| engine-clean | 41 | 46 | 87 | 0 |
| restructure-free | 39 | 38 | 77 | 0 |

The top two tied on score and on votes; the harness named friction-first. *(Correction beside the original, 2026-11-14, by the main Claude session, which wrote the harness: its tie-break was the order in which the proposals were listed, not a judgement. friction-first was third in that list and minimal-supportable fourth, so the harness's naming carries no weight; the author's reasons that follow are the only argument for the choice.)* I accept that on the merits: your words stress low friction, and the runner-up's weakest points (command line only; one mutable file) are the ones the grafts repair. The judges named no fatal flaw in the winner. Their stated weaknesses, and what this design does:

| Weakness raised against friction-first | Response |
|---|---|
| Two-way mirror is the riskiest code (M3 is XL) | Pure three-way plan function with a row table; no base means never delete; poll-only in the alpha; simulator first; `sync --plan` (7.2, 11.4, M3) |
| macOS polling cost unmeasured | Gate 0 spike (c); budget test; watchers deferred and optional (M9) |
| `file:` page posting to loopback untested | Gate 0 spike (b); fallback named; the page is a separate helper so the core keeps no listener |
| Setup page reintroduces a loopback listener | Moved out of the agent into `cloudcrypt-ui`; `net/http` banned elsewhere by test (T3) |
| Alpha list includes signing and installers | Signing and real installers moved to M11 (beta gate); alpha builds unsigned |
| Lost-device key generation only sketched | Specified in 8.5, with a new password and recovery key required and the key chain in 6.3 |
| Flat log folder: whole listing, nothing detects a withheld segment | Shard folders and modified-time probe (6.6); per-device sequence, hash chain and seen-list inside sealed segments |
| Hybrid logical clock added without need | Dropped; plain time chooses only which copy keeps the name |
| Read fewer review sections (not 6; part of 4) | I read sections 1, 2, 5, 6 (to 6.4), 7 and the P0 and P1 headings; P2 and P3 remain unread here (section 10) |

**Grafts taken.**
- From minimal-supportable (both judges): the pure three-way plan with "no base means never delete", edit beats delete, re-check before replace and `sync --plan`; `Commit` that refreshes and returns `ErrStale` (adapted to "against the snapshot it planned from", because row 5 deliberately commits a non-head base); allowed-edges import table; no `net/http` in core packages; `Store` with create-only `PutNew`; `os.Root`-style confinement in `plainout` (needs Go 1.24 [recall]); scenario rows with "break it by"; tag `legacy/0.17.0` and delete the old tree at M1; shard listing; no file-watcher dependency in the alpha; `status.json`.
- From engine-clean (both judges): the deterministic simulator with five invariants and a scale benchmark with budgets; wrap associated data bound to the exact header bytes; a fork kept as data; required and optional features and a critical bit on records; reserved checkpoint and GC record kinds; NFC plus simple case-fold with collisions rendered as conflicts, not refusals; dedup that verifies a chunk before reuse.
- From restructure-free: `plainout` as the only plaintext writer with a SameFile test over every path spelling (both judges); spec ids tied to tests (judge 1); the honest statement that a password change does not retire the old password (judge 1); two-tier lost-device wording (judge 1); `minWriter` and the must-understand bit and the hash in the segment name (judge 2; judge 1 listed them as best ideas, not contradicted); the `Recovered` placement for cycles and orphans (judge 1's best idea, not contradicted).
- From friction-first itself, kept: journeys, mandatory recovery key, sync-folder detection, mirror outside sync roots, sentinel and trash, per-generation key files, independent second decoder, tray helper in its own module.

**Grafts rejected, with why.**
- Per-device public log folders with contiguous numbers (engine-clean, judge 2): judge 1 noted they reveal device count; replaced by the in-block chain and seen-list.
- Causal delivery, Lamport clocks and context vectors, Padme padding, per-object fixed-nonce keys, keyed gear table, heartbeats and 90-day lease (engine-clean): gold-plating for a pre-alpha; causal delivery stalls on one late file.
- Restricting names to the all-OS intersection (engine-clean): refuses valid Linux names; names are rendered, not refused.
- The single mutable `cloudcrypt.header` with compare-and-swap (minimal-supportable): a sync folder cannot do compare-and-swap; immutable key files replace it.
- CLI-only alpha (minimal-supportable): kept only as the owner-usable cut at M4; the page follows at M5.
- Control files instead of a socket (minimal-supportable): cannot move a typed password to the agent without writing it to a file.
- Path-keyed versions (minimal-supportable, judge 2): replaced by node ids.
- Browser file manager with edit sessions, the launch-file auth flow as the daily surface, mount deferred behind WebDAV (restructure-free): more friction than a folder; the single-use code survives only for the setup page, pending spike (b).

**Sources reached.** The two proposals read in full (friction-first, minimal-supportable); the other two proposals only through the judges' text, which I did not re-check against them. In the repository I opened: go.mod; chunker.go (lines 1 to 97); vault.go 1465 to 1512; anchor.go 35 to 65; leak_test.go and loopback.go function lists; line counts of the surviving files; PROMPT.md in full; DESIGN.md lines 1 to 40; the review header, sections 1, 2, 5 opening, 6 (to 6.4) and all of 7; the Phase R design lines 1 to 30 and 521 to 590 for format; the A2 design opening; drive-review lines found by keyword search (11, 35-37, 290); [repo] README.md:3 and [repo] internal/keychain (opened by the condition re-checker on 2026-11-14; revision 3, Condition 14). Not reached: review sections 3, 4.3, 4.4, 6.5 onward, 8 and 9; all files I cite only through the review's strengths (for example gc.go:42-55, classify.go:53-68, keychain_darwin.go, tamper and rollback tests); drive-review:259; the whole Phase R body; every [recall] platform fact. Smallest check for the owner: run `wc -l` on `internal/webui/*.go` (non-test) and confirm 5,397, and open `internal/vault/vault.go` lines 1469 to 1510; then have someone on Windows and one on macOS confirm the [recall] facts in 5.4 and 5.5 before decision 1 is final.
