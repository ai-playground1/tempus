# CLAUDE.md

## Environment

- This is a Debian 13 virtual machine dedicated to development.
- You are running as the user `coder`, who has root access (via `sudo`).
- This file is the authoritative policy for what you may and may not do on
  this machine, and describes the project you are building.

## Allowed

- **Package installation via `apt`:** You may install any package from the
  configured Debian repositories using `apt` / `apt-get` (e.g.
  `sudo apt install <package>`), including updating package lists with
  `sudo apt update`.
- **Running local programs:** You may run any program that is installed locally
  on this machine.
- **Project dependencies and tooling:** You may install anything needed to
  continue development of the current project. This includes language package
  managers and their registries (e.g. `composer`/Packagist, `npm`), compilers,
  build tools, linters, debuggers, test frameworks, Docker, and PHP tooling.
- **Git:** You may create git commits and push to the project's configured
  remotes (`git commit`, `git push`).
- **Root access:** You have root access via `sudo`. Use it responsibly and only
  when necessary (e.g. installing packages, running Docker, managing services
  required by the project). Prefer running as `coder` without elevation
  whenever possible.
- **Querieng Websites**: You can query websites to retrieve information or if
  you want to inspect how they look.

## NOT Allowed

- **Network interactions:** Do not initiate any network interactions with other
  devices or hosts. The only exceptions are those required by the "Allowed"
  section above: fetching packages via `apt`, downloading project dependencies
  from official package registries, pulling Docker base images, and
  pushing/pulling to the project's git remotes. No scanning, probing,
  connecting to, or exfiltrating data to any other machine or service.
- **Destructive or malicious actions:** Never perform destructive operations
  such as `rm -rf /` (or anything with similarly broad destructive effect),
  and never download, install, write, or execute malware, viruses, backdoors,
  or any similar malicious software or code.
- **Operating system manipulation:** Do not manipulate the operating system.
  Do not modify system configuration outside the scope of the project, do not
  change bootloader/kernel settings, do not alter system users, groups, or
  permissions, do not disable or reconfigure security mechanisms, and do not
  modify files outside the project directory unless it is a direct,
  unavoidable consequence of an allowed action (such as `apt` installing a
  package).

# Your Instructions

- **Develop on your own.** Do not ask any questions about how anything should
  be. Create a complete prototype first; the owner will review it and then
  provide instructions for desired changes.

- **Usage notifications:** Notify the owner via `telegram-send-msg` every
  time your usage limit decreases by 10%, and once it is about to reach 0%.
  The script takes exactly one parameter: the text message to send. Example:
  `telegram-send-msg "Usage at ~70% remaining, continuing with the product page."`

- **Progress notifications:** Notify the owner via `telegram-notify-commit`
  every time you create a git commit. The script takes 2 parameters, the first
  one is the commit hash and the second one is the message. Example:
  ```
  telegram-notify-commit "f7819ab4d9124500cbae0b42bc1fce5e1a928665" "Track usage-notification hooks referenced by settings.json; ignore hook state files\n\nCo-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
  ```

- **Git workflow:** Create a git commit for every minor change, with a clear
  message, and push each commit upstream on the current branch `feat-avrcp` on the origin `fork`.

- **Tooling:** Install packages or development tools if necessary (within the
  rules under "Allowed" / "NOT Allowed" above).

- **After Task complete:** Whenever you complete a bigger assignment (may
  contain multiple commits), notify the owner that the assignment is done via
  `telegram-send-msg`
