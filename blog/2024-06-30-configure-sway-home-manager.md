---
slug: sway-on-nix-os
title: Install Sway on Nix OS
authors: [giri]
tags: [sway, nixos]
---
# Configuring Sway in NixOS (using Home Manager)

At the time of writing writing this, the NixOS wiki was some kind of tricky and was outdated. So I had to install and configure sway by trial and error. Can't lie, it did take few "rebuilds" to get everything working (fonts was the main pain).

### Prerequisites (Fonts)

By default, Sway tries to find a monospace font (DejaVu) to render. If you haven't installed it, we need to install that first through NixOS configuration.

```nix title=configuration.nix
fonts.packages = with pkgs; [
  dejavu_fonts.fll-ttf
  liberation_ttf
  noto-fonts
}

# OpenGL was renamed to `graphics`
hardware.graphics.enable = true;

# This needs to be added in if we're going to install sway via hm
security.polkit.enable = true;
```

I added `noto-fonts` for wider Serif support and `liberation_ttf` as that was one of the fonts got installed by default if we try to install sway via NixOS configuration.

> [!NOTE]
> Ideally, managing the fonts via `home-manager` should also work via `fonts.fontconfig.enable` option. But it didn't work for me at the first time, so that's why I went with the NixOS configuration. 


### Install

```nix title=home.nix
wayland.windowManager.sway = {
  enable = true;
  wrapperFeatures.gtk = true;
  config = null;
  extraConfig = lib.fileContents ./config;
}

home.packages = with pkgs; [
  dmenu # Run
  grip # Screenshots
  slurp # Screenshots
  wl-clipboard # Copy / Paste from stdin / stdout
  mako # Notification System 
  wmenu # Run
];
```
> [!NOTE]
> **Sway External Configuration**
> If you look at the definition, you'll notice that I'm setting `config` to `null` and passing all the config as a normal sway.config file via `extraConfig` as opposed to "NixOS" convention. As much as I love the "NixOS" way, I had to write the dotfiles as nix-independant so that I can reuse the same config through other sytems (especially at workplace). 
> Also, I'm a distro-hopper, so can't stick config files to nix.

Now execute:

```bash
# Rebuild the NixOS to install fonts and OpenGL
$ sudo nixos-rebuild switch --flake .

# Reload Home Manager
$ home-manager switch --flake .
```

That's it!, if all goes well, you can start the sway via

```bash
sway
```

End.

