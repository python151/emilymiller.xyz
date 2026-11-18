+++
author = "Emily Miller"
title = "SkyrimVR on NixOS"
date = "2026-11-18"
description = "Documenting my Skyrim VR config for NixOS so you don't have to spend 8 hours on this."
categories = [ "NixOS", "Linux Gaming" ]
+++

This assumes you're using Monado and XRizer, as seems to be the recommended system for NixOS with an NVIDIA GPU. This is also intended for the Valve Index, and configuration for other headsets may vary. [See my SteamVR guide for more details](https://www.emilymiller.xyz/post/steamvr_instructions/).

To start, you need to download Skyrim VR via Steam. Note that this is a separate game from Skyrim Special Edition (don't pay full price, wait for a sale, it still default sells for 50gbp which is far too much). After this, you'll need to boot up the game at least once.

If you're using XRizer, you'll be soft-locked when you first launch the game when you need to select your dominant hand, but this can be bypassed using a physical keyboard attached to your PC using the arrow keys. The next time you'll be soft-locked is in character creation, where the game will send an OpenVR command to XRizer for a keyboard, but no implementation exists on stock XRizer. [I've forked XRizer](https://github.com/python151/xrizer-keyboard/tree/main) and added support for a minimal keyboard that appears on your desktop to bypass these soft locks, and am working on getting it merged. In the meantime, if you're using the `nixpkgs-xr` overlay, you can manually pull in my fork by including the following in your configuration:
```nix
nixpkgs.overlays = [    
       nixpkgs-xr.overlays.default  
       (final: prev: {  
         xrizer = prev.xrizer.overrideAttrs (old: rec {  
           version = "b63fbc5";  
           src = final.fetchFromGitHub {  
             owner = "python151";  
             repo = "xrizer-keyboard";  
             rev = "b63fbc57dc1e4ec6629c7a973ce921f22ad2afa8";  
             hash = "sha256-KxEAhsooqzjkwZKC5vTmE0t0JxQq1PaGFDxuacfUans=";  
           };  
           
           cargoDeps = final.rustPlatform.importCargoLock {  
             lockFile = "${src}/Cargo.lock";  
           };  
         });  
       })  
   ];
```

From here, we'll want to setup Mod Organizer 2 for Nix. MO2 has a Linux based installer, which configures wine and such for us, called `mo2-lint` (MO2 Linux Installer).  `mo2-lint` in turn has a Nix flake built for it. A very minimal `flake.nix` file including the `mo2-lint` flake can be seen below:
```nix
# /etc/nixos/flake.nix  
{  
 description = "flake for nixos";  
  
 inputs = {  
   nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";  
   mo2-lint.url = "github:ulfurloyd/mo2-lint-flake";  
 };  
  
 outputs = { self, nixpkgs, mo2-lint, ... }: {  
   nixosConfigurations = {  
     nixos = nixpkgs.lib.nixosSystem {  
       system = "x86_64-linux";  
       specialArgs = { inherit mo2-lint self; };  
       modules = [  
         ./configuration.nix  
       ];  
     };  
   };  
 };  
}
```
To actually have the package, you'll need to edit your `configuration.nix` to add `mo2-lint` as an input (so your first line should look something like `{ config, pkgs, mo2-lint, ... }:`), and add `mo2-lint.packages.x86_64-linux.default` to your system packages. After rebuilding, you should now have `mo2-lint` installed. The flake also offers an overlay, but I haven't been successful in using it, and using the package seems to just work.

Now you can run `mo2-lint install skyrim_vr` to setup an MO2 instance for Skyrim VR, and you'll need to follow the instructions given, which amounts to changing your compatibility layer in Steam to a newly created MO2 Skyrim VR option.

After doing this, you should be able to launch Skyrim VR from Steam, and get an MO2 pop-up, which functions very similarly to a Windows Installed version. You'll need to install some mods into the game in order for it to be at all playable, most of which you can download from [Nexus](https://www.nexusmods.com/) after making an account.

The first mods you'll want to install are pre-requisites for many other mods. To start, you'll want to install [SKSE VR](https://skse.silverlock.org/) (be sure to get the VR build). To do so, download the build, extract the 7z archive, and copy the files directly into the install directory in Steam (which you can get to by clicking on the gear icon in the top right, then Manage > Browse local files). After this is installed, you should see a new launch option in MO2 after re-starting through Steam. Next, you'll want to add the VR Address Library and Skyrim VR Tools, both of which can be gotten from Nexus (I use the Manual Download option). To install them in MO2, you can go to File > Install Mod and select your the file you downloaded from Nexus (you can also authenticate MO2 with Nexus and use a handler, which does work with MO2-Lint and the flake, but I find that more effort than its worth). At this point you should launch the game to make sure it still launches and works correctly. 

Next, we'll install the bare minimum mods, which is HIGGS, PLANCK, SkyUI, and VRIK. All of these can be installed via Nexus. Again, launch and test. I'd recommend setting up your a keybind for "Activate" now, as this is how you loot and interact with objects, and will soft-lock you if your controller doesn't have a binding by default (e.g. the Knuckles without a translation layer -- Steam VR seems to include in-built translation, but that won't launch on Nix by default, so it's less than ideal. There is also some suggestions to implement this into XRizer for supporting Legacy games on the Steam Frame, which would hopefully make it easy-ish to do so for the Knuckles -> Wand, but at the time of writing no such solution exists). 

Now I'd very-strongly-highly recommend installing No Stagger Mod, Menu Mouse Fix, and Stable Hands (and their pre-requisites), which are all on Nexus and will greatly improve the usability of the menus. I'd also strongly recommend installing Spell Wheel VR and Mage VR, both of which greatly improve usability of the game. From here, you can get away without installing these, but I'd still recommend them, which is Weapon Throw VR, Simple Realistic Archery Fix, and Archery, and Static Mesh Improvement Mod.

All of the former mods still feel very vanilla, and simply make the game more usable and more immersive, without significantly altering the experience. There are *a lot* of mods that add content and improve gameplay and make the game look and feel better that were't covered here, and I'd recommend looking at some other mod lists for inspiration, and checking out Cangar on YouTube who has some very good Skyrim VR modding guides (though all Windows focused, so some things will be different).

Now, once all of this is done, you should be able to get through to the end of the Skyrim Tutorial at least, and, well, play the game.


