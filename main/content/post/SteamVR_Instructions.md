+++
author = "Emily Miller"
title = "SteamVR on NixOS"
date = "2026-11-16"
description = "Documenting my VR config for NixOS so you don't have to spend 4 hours on this."
categories = [ "NixOS", "Linux Gaming" ]
+++

# SteamVR on NixOS for the Valve Index with an NVIDIA GPU

## Architecture

- This uses Monado as the compositor with XRizer as a compatability layer.
- This has been tested to work on NixOS Stable Yarara with KDE Plasma.
- This was specifically built to run SteamVR games on a Valve Index on a PC with a NVIDIA GeForce 3060 12GB card. The computer also uses a Ryzen 3600X and 32GB of DRAM, but that part shouldn't impact the configuration.
- Pavlov, The Lab, VTOL VR, and Google Earth VR have all been tested to work with this configuration. I've not tried any other games yet. 
- *(Limitations & Known Bugs)* I should mention that The Lab has had some instability, crashing when switching scenes occasionaly, I suspect this is a Proton (compatability layer that lets us play Windows games on Linux) issue however, and no other game (including Pavlov) has had this issue. I've also been unable to get audio working through the headsets inbuilt speakers.

## Starting a game

- Plugin the Index, get it powered on, and get both controllers and base stations paired (all should have a green indicator light)
- *(Testing only)* Open a terminal emulator (e.g. Konsole or Kitty), and run the following commands:
    - `systemctl --user restart monado` *(This starts monado as a user daemon)*
    - `journalctl --user -fu monado` *(This lets you see what monado is doing)*
    - `nix-shell -p xrgears --run xrgears` should display a scene with gears rotating in the headset.
- Start Steam, and launch the game you wish to play **without starting SteamVR**. Monado should auto-start if you haven't started it already, though I have found it convienent to start it manually if a game is having trouble starting (you can see in the logs that the headset and game have connected properly, for instance)

## Installation (do this once before trying to play a game)

Add the below to your Nix config (you'll also need the *rest* of your `configuration.nix`, so use this more as a template)

```nix
let
    user = "emily747";
    nixpkgs-xr = import (builtins.fetchTarball
        "https://github.com/nix-community/nixpkgs-xr/archive/main.tar.gz");
    openvrPaths = pkgs.writeText "openvrpaths.vrpath" (builtins.toJSON {
        version = 1;
        jsonid = "vrpathreg";
        external_drivers = "null";
        config = [ "/home/$(user)/.local/share/Steam/config" ];
        log = [ "/home/$(user)/.local/share/Steam/logs" ];
        runtime = [ "${pkgs.xrizer}/lib/xrizer" ]; # xrizer converts SteamVR commands to Monado commands
    });
in {
    # Add the nixpkgs-xr overlay to get up-to-date builds
    nixpkgs.overlays = [ nixpkgs-xr.overlays.default ];

    # Enable nvidia drivers
    hardware.graphics.enable = true;
    hardware.graphics.enable32Bit = true;
    services.xserver.enable = true;
    services.xserver.videoDrivers = [ "nvidia" ];
    hardware.nvidia = {
      modesetting.enable = true;
      open = true; # the 3060 has open source drivers, other cards might not.
    };
    
    # Installs steam
    programs.steam = {
        enable = true;
        remotePlay.openFirewall = true;
        dedicatedServer.openFirewall = true;
    };


    # Configures monado as compositor instead of SteamVR
    services.monado = {
        enable = true;
        defaultRuntime = true;
        highPriority = true;
    };
    systemd.user.services.monado.environment = {
        STEAMVR_LH_ENABLE = "1"; # Uses the SteamVR driver for controller tracking, more performant but potentially less stable?
        XRT_COMPOSITOR_COMPUTE = "1";
        WMR_HANDTRACKING = "0";
        XRT_COMPOSITOR_USE_PRESENT_WAIT = "1";
        U_PACING_COMP_TIME_FRACTION_PERCENT = "90";
    };
    # This asks steam nicely to use an OpenXR runtime.
    programs.steam.package = pkgs.steam.override {
        extraProfile = "export PRESSURE_VESSEL_IMPORT_OPENXR_1_RUNTIMES=1";
    };
    # This does all the OpenXR configuration, and blocks steam from overwriting it.
    systemd.tmpfiles.rules = [
        "d /home/$(user)/.config/openvr 0755 $(user) users -"
        "d /home/$(user)/.config/openxr 0755 $(user) users -"
        "d /home/$(user)/.config/openxr/1 0755 $(user) users -"
        "L+ /home/$(user)/.config/openvr/openvrpaths.vrpath - - - - ${openvrPaths}"
        "L+ /home/$(user)/.config/openxr/1/active_runtime.json - - - - ${config.services.monado.package}/share/openxr/1/openxr_monado.json"
    ];

    # Installs KDE as a desktop manager
    # You can use a lot of things but GNOME Wayland won't work
    services.desktopManager.plasma6.enable = true;
    services.displayManager.plasma-login-manager.enable = true;

    # The rest of your config here...
}
```

## Tips & Quirks

- I've found that restarting with the headset connected is the most reliable way of getting games to launch and everything working first try.
- On the Valve Index, the audio is done over DisplayPort, and on an NVIDIA GPU all ports show as one audio device with multiple streams, so you may have to go into a secondary menu to see the option to select the headset.
- You need to have SteamVR installed using a Linux Runtime as the compatibility layer for the lighthouse driver to be installed, though it shouldn't matter which version.

## Useful Resources

- https://vronlinux.org/docs/distros/nixos/
- https://wiki.nixos.org/wiki/VR

