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

- As per the Niri GitHub wiki's page, we are able to use `niri` inside an existing desktop session where it runs as its own window.
- We will then leverage this fact and run `niri` as a Wayland app inside WSLg by running `niri &`.
- In this nested compositor mode, `Alt` is used as the default modifier.

## Known Issues:

- Niri does not offer out-of-the-box XWayland compatibility.

  > [!NOTE]
  > Niri will introduce built-in XWayland support through `xwayland-satellite` in the next release (post v25.05.1; Niri bases its release on date like NixOS).
  > This means that this item point is no longer needed as Niri itself will spawn `xwayland-satellite` on startup and configure the `DISPLAY` variable.

  > [!HINT]
  > Niri recommends `xwayland-satellite` for introducing XWayland supaport. `xwayland-satellite` implements rootless XWayland without the host compositor's involvment.

- Clipboard isn't synced between WSLg's weston and the nested Niri instance.o

  > [!HINT]
  > Since WSLg already handles clipboard syncing between the Windows, X11 and Wayland clipboards, we only need to sync WSLg's weston's clipboards and niri.
  > Also, as we will be using `xwayland-satellite` for XWayland support in Niri, clipboard syncing between X11 and Wayland inside Niri is already handled.
  > We only need to sync WSLg weston and Niri's wayland clipboards. Fortunately this can be done via `wl-clipboard`'s `wl-paste --watch`' and `wl-copy`.

- Running X11 apps inside Niri hooks it up to WSLg's XWayland server.

  > [!HINT]
  > Setting XWayland up via `xwayland-satellite` and configuring the `DISPLAY` variable would fix this issue. Since WSLg handles setting `DISPLAY` up, by default, X11 apps that run on WSL will run on WSLg's XWayland.

## Nice to Haves:

- Ability to suspend `niri` and its child processes to save up on resources when not in use.
