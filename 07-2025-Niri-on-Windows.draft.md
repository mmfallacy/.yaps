- Niri is a scrollable-tiling window manager.
- It is a Wayland compositor written in Rust.
- Unlike tiling window managers like Sway and Hyprland, it orders windows into columns on an infinite scrolling strip.
- Workspaces are arranged vertically, with unique independent sets for every monitor[^1]

[^1]: https://github.com/YaLTeR/niri

- WSL, or the Windows Subsystem for Linux is a compatibility layer that allows developers to use Linux distributions inside Windows.
- WSL operates as a type 1 hypervisor that aims to be more performant than traditional virtual machines.
- WSL2 introduces major improvements to its predecessor. It now ships with a Full Linux Kernel, with full syscall compatibility as well as systemd support. [^2]

[^2]: https://learn.microsoft.com/en-us/windows/wsl/compare-versions

- As of 2024, WSL now allows running Linux GUI applications (Both X11 and Wayland) through WSLg.
- WSLg operates by containerizing XWayland, Weston, and the Pulse Audio server into what we call the WSLg System Distro.
- This component serves as the intermediary layer which bridges user-level X11 and Wayland apps with the Windows Host through RDP connection.[^3]

[^3]: https://github.com/microsoft/wslg?tab=readme-ov-file
