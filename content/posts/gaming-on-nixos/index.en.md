+++
title = 'Gaming on Nixos'
date = 2024-06-26T18:52:55+07:00
author = 'Vicyann'
featuredImage = 'featured.png'
draft = false
+++

I want to write this post to help me in the future, in case there are things happen, I need to reinstall NixOS again or figure out what happens with my systems. Because of the non-FHS and declarative nature of NixOS, it is harder than other distros when we want to do something or install some apps. I also hope that this post can also be a reference for other users too. Note that this is not a tutorial post, always follow with the main wiki for up-to-date information for each part.

## 1. Enable the Nvidia driver

If your laptop or desktop doesn't have a discrete Nvidia card, feel free to skip this part. Both of my laptops have a Nvidia card so I need to install this Nvidia thing. You can check more information at [nix wiki nvidia pages](https://nixos.wiki/wiki/Nvidia).
Add the following code to the configuration files.

```configurations.nix
{ config, lib, pkgs, ... }:
{

  # Enable OpenGL
  hardware.opengl = {
    enable = true;
    driSupport = true;
    driSupport32Bit = true;
  };

  # Load nvidia driver for Xorg and Wayland
  services.xserver.videoDrivers = ["nvidia"];

  hardware.nvidia = {

    # Modesetting is required.
    modesetting.enable = true;

    # Nvidia power management. Experimental, and can cause sleep/suspend to fail.
    # Enable this if you have graphical corruption issues or application crashes after waking
    # up from sleep. This fixes it by saving the entire VRAM memory to /tmp/ instead
    # of just the bare essentials.
    powerManagement.enable = false;

    # Fine-grained power management. Turns off GPU when not in use.
    # Experimental and only works on modern Nvidia GPUs (Turing or newer).
    powerManagement.finegrained = false;

    # Use the NVidia open source kernel module (not to be confused with the
    # independent third-party "nouveau" open source driver).
    # Support is limited to the Turing and later architectures. Full list of
    # supported GPUs is at:
    # https://github.com/NVIDIA/open-gpu-kernel-modules#compatible-gpus
    # Only available from driver 515.43.04+
    # Currently alpha-quality/buggy, so false is currently the recommended setting.
    open = false;

    # Enable the Nvidia settings menu,
	# accessible via `nvidia-settings`.
    nvidiaSettings = true;

    # Optionally, you may need to select the appropriate driver version for your specific GPU.
    package = config.boot.kernelPackages.nvidiaPackages.beta; # I choose the beta branch.
  };
}
```

Then, we need to add BUS ID values. To do this, you need to install the `lshw` package and run `sudo lshw -c display` in the terminal. The output will be something like this.

```bash
*-display
       product: i915drmfb
       physical id: 2
       bus info: pci@0000:00:02.0
       logical name: /dev/fb0
       version: 02
       width: 64 bits
       clock: 33MHz
       capabilities: pciexpress msi pm bus_master cap_list rom fb
       configuration: depth=32 driver=i915 latency=0 resolution=1920,1080
       resources: irq:150 memory:b2000000-b2ffffff memory:c0000000-cfffffff ioport:5000(size=64) memory:c0000-dffff
  *-display
       physical id: 0
       bus info: pci@0000:02:00.0
       version: a1
       width: 64 bits
       clock: 33MHz
       capabilities: pm msi pciexpress bus_master cap_list rom
       configuration: driver=nvidia latency=0
       resources: irq:151 memory:b3000000-b3ffffff memory:a0000000-afffffff memory:b0000000-b1ffffff ioport:4000(size=128)
```

My Nvidia BUS ID is `02:00.0` and my Intel BUS ID is `00:02.0`
**Convert those numbers to decimal (because they are in heximal format)** and add to configuration files

```configurations.nix
hardware.nvidia.prime = {
# Make sure to use the correct Bus ID values for your system!
	nvidiaBusId = "PCI:2:0:0";
	intelBusId = "PCI:0:2:0";
    # amdgpuBusId = "PCI:54:0:0"; For AMD GPU
};
```

Next, add sync mode or offload mode for the Nvidia card (read the wiki). I will use sync mode in my case because my laptop is charged all the time. I don't need to worry about battery life.

```configurations.nix
hardware.nvidia.prime = {
# Make sure to use the correct Bus ID values for your system!
	sync.enable = true;
	nvidiaBusId = "PCI:2:0:0";
	intelBusId = "PCI:0:2:0";
    # amdgpuBusId = "PCI:54:0:0"; For AMD GPU
};
```

## 2. Install Steam, proton and steam-run

Check out the NixOS wiki about [steam](https://nixos.wiki/wiki/Steam).
To play Linux-supported games on Steam, we just need to install Steam. But if we want to install non-supported games, we need Proton (a compatibility layer for Windows games to run on Linux-based operating systems). But that's still not enough, I also want to play games downloaded from the internet, not only from Steam (Genshin in this case). And because NixOS has a non-FHS environment, we need to create the typical environment expected by proprietary games and software on regular Linux, allowing us to run such software without patching. That's when steam-run comes into play. Summary:

- Steam: play a Linux game from Steam.
- Proton with Steam: play Windows games from Steam.
- steam-run + proton: play Windows games from the internet.
  With them, nothing can stop me from playing anything I want except kernel-level anti-cheat like Vanguard.

### Install steam and steam run

Add the following code to the configuration files to enable steam.

```configurations.nix
programs.steam = {
  enable = true;
  remotePlay.openFirewall = true; # Open ports in the firewall for Steam Remote Play
  dedicatedServer.openFirewall = true; # Open ports in the firewall for Source Dedicated Server
};
```

We also need to enable some unfree packages.

```configurations.nix
{
  nixpkgs.config.allowUnfreePredicate = pkg: builtins.elem (lib.getName pkg) [
    "steam"
    "steam-original"
    "steam-run"
  ];
}
```

Install steam-run

```configurations.nix
  environment.systemPackages = with pkgs; [
    steam-run
  ];
```

Last but not least, enable the option below to ensure the mouse doesn't disappear when playing the game.

```configurations.nix
environment.sessionVariables = {
	WLR_NO_HARDWARE_CURSORS = "1";
};
```

### Install proton

Follow [vimenjoyer's video](https://youtu.be/qlfm3MEbqYA?si=WA6UtJ8NqBnE7aGp), we will use an imperative way to install proton for steam. Add proton up to your package manager and path to compatibility tool for steam

```configurations.nix
environment.systemPackages = with pkgs; [
    protonup
];

home.sessionVariables = {
	STEAM_EXTRA_COMPAT_TOOLS_PATHS = "\${HOME}/.steam/root/compatibilitytools.d";
};
```

Then run the command `protonup` in the terminal to install Proton.

## 3. Install games (genshin, don't starve together)

### Don't starve together

Because `don't starve together` can be played natively on Linux, all we need to do is install and launch the game. But by default, this game will use iGPU instead of dGPU (I don't know why). So we need to add this launch command for don't starve to make the game using your Nvidia card.

```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia %command%
```

### Genshin Impact

This is where the hard part comes. But we can solve it with proton and steam-run.

- Download the Genshin Impact installer for Windows from the Hoyoverse website.
- Add the installer to Steam and choose Properties -> Compatibility -> Force the use of a specific Steam Play compatibility tool. Then choose Proton Experimental.
- Choose Properties -> Shortcut and use the steam-run command for the launch option so the installer will work with a non-FHS environment.
  `steam-run %command%`
- Then launch the installer and install it like on Windows.

After installation

- Open `~/.steam/steam/steamapps/compatdata/` and then search for GenshinImpact.exe file. Copy the file's path and add it to Steam as a non-steam game.
- Add Proton Compatibility and steam-run to that `GenshinImpact.exe`
- Launch and enjoy the game.
