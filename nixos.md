Introduction

> Software deployment is about getting computer programs from one machine to another,
> and having them still work when they get there. - E. Dolstra

* The Nix deployment system is a unique take on software package management.
Inspired by the concept of functional purity found in some programming
languages, it side-steps many problems that dog traditional approaches.

* The Nix eco-system consists of a programming language, a package
 manager, a package repository, a Linux distribution, and more.

* The minimum useful setup consists of the Nix package manager running on
  a non-NixOS machine, like RedHat or Ubuntu.

---

The Nix Programming Language

* Nix is a programming language for describing how to build and compose software.
* Characterisation: pure, lazy, functional, dynamically typed, domain-specific

Nix Packages

* Nixpkgs is a collection of Nix expressions, which specify how to build
  thousands of software packages commonly found in various Linux distributions
  (libc, gcc, firefox, etc.)
* You are encouraged to write Nix expressions to build your own software.

The Nix Package Manager

* The Nix package manager evaluates Nix expression, downloading, compiling, and
  installing software.

---

How is Nix different?
* Packages are installed into `/nix/store`; they are hidden
  from the standard Unix environment as specified by the Filesystem Hierarchy
  Standard.

* Each package is assigned a 160-bit cryptographic hash value computed from
  it's source, build options, and, recursively, from its build-time and
  run-time dependencies.

* Packages cannot have build-time or run-time dependencies outside of the nix
  store.

* Dependency specifications are structural and complete.

* Everything in `/nix/store` is stored read-only; if a package was installed
  successfully, it is unlikely that it will ever stop working.

* New versions of packages are installed along-side old versions.

---

How is Nix different?

* Standard shared library directories do not exist, and `LD_LIBRARY_PATH` is
  cleared before each build. Binaries' rpaths contain absolute paths into the
  nix store.

* User environments, kept in the nix store as trees of symbolic links,
  assemble read-only views of activated packages. They contain the usual FHS
  subdirectories `/bin`, `/lib/`, etc.

* Packages are activated and deactivated by creating new environments,
  along-side the old.

* Live environments are pointed to from outside the nix store. Every
  modification can be reliably activated and rolled back in constant
  time, because creating a symbolic link on Unix is an atomic operation.

---

How is Nix different?

* To copy a package and its dependencies to another machine, Nix computes the
  package's *closure*.  Closures are computed by recursively scanning package
  contents for bit patterns that look like paths in the nix store, which are
  easily recognisable.

  ```
  /nix/store/xlzxy4rjm573kjhw4w86xkdz3n9gb4wn-libpng-1.2.56
  ```

* Packages remain present in the nix store until they become un-reachable from
  outside the store, at which point they are garbage-collected.

* This is exactly how some languages with weak pointer disciplines implement
  conservative garbage collection. The execution stack is scanned for bit patterns
  that look like addresses. Any object allocated on the heap
  and not, directly or indirectly, pointed to from the stack is garbage collected.

---

Deployment as Memory Management

* Nix treats the file system as a purely functional programming language
  treats its memory space.

* File systems and RAM are both storage mechanisms; programs manipulate memory,
  deployment operations manipulate the file system.

> This analogy reveals that the safeguards against abuse of memory applied by
> programming languages are absent in deployment, and suggests that we can make
> deployment safe by trying to impose similar safeguards. - E. Dolstra

---

NixOS

* NixOS is a Linux distribution built with the Nix package manager.

* The entire state of a NixOS machine can be built from one system configuration
  file (written in Nix) and a snapshot of Nixpkgs.

* NixOS provides safe atomic updates and rollbacks of an entire system.

---

NixOS, configuration.nix
```
{ config, pkgs, ... }:

let
  user = {
    username = "lazar";
    unumber = "u62208";
  };

in

{
  imports =
    [ ./hardware-configuration.nix
      ../../mixins/postgres/postgres-service.nix
      ../../mixins/java-env.nix
    ];

  virtualisation.virtualbox.guest.enable = true;
  boot.initrd.checkJournalingFS = false;
  boot.loader.grub.enable = true;
  boot.loader.grub.version = 2;
  boot.loader.grub.device = "/dev/sda";

  networking.hostName = "dev";
  networking.proxy.default = "http://localhost:3128";

  time.timeZone = "Australia/Canberra";

  nixpkgs.config = import ../../nixpkgs-config.nix;

  # List packages installed in system profile. To search by name, run:
  # $ nix-env -qaP | grep wget
  environment.systemPackages = with pkgs; [
    ctags
    file
    firefox
    gcc
    git
    gitAndTools.hub
    gnumake
    keychain
    nix-prefetch-scripts
    nix-repl
    nload
    telnet
    tmux
    tree
    unzip
    vim_configurable
    wget
    zip
  ];

  # Use your own CNTLM. Set username to your u-number
  # and put your password into /etc/cntlm.password.
  # Remember to 'chmod 0600 /etc/cntlm.password'.

  services.cntlm = {
    enable = true;
    username = user.unumber;
    domain = "PROD";
    password = import /etc/cntlm.password;
    proxy = ["proxy.ga.gov.au:8080"];
    port = [3128];
    netbios_hostname = "127.0.0.1";
  };

  services.openssh.enable = true;

  # Enable the X11 windowing system.
  services.xserver.enable = true;

  # Enable the KDE Desktop Environment.
  # services.xserver.displayManager.kdm.enable = true;
  # services.xserver.desktopManager.kde4.enable = true;

  services.tomcat = {
    enable = true;
    package = pkgs.tomcat8;
  };

  security.sudo = {
    enable = true;
    wheelNeedsPassword = true;
  };

  # Define your user account. Don't forget to change your password.
  users.extraUsers = {
    ${user.username} = {
      password = "change-me";
      isNormalUser = true;
      uid = 1000;
      extraGroups = [ "wheel" ];
    };
  };

  system.activationScripts = {
    dotfiles = pkgs.lib.stringAfter [ "users" ]
      ''
      # Import /etc/nixos/nixpkgs-config.nix from users private ~/.nixpkgs/config.nix
      # so that nix-env commands can find packages defined globally in nixpkgs-config.nix.
      if [ ! -e ~${user.username}/.nixpkgs/config.nix ]; then
        mkdir -p ~${user.username}/.nixpkgs
        cat > ~${user.username}/.nixpkgs/config.nix << EOF
      import /etc/nixos/nixpkgs-config.nix // {
        allowBroken = false;
      }
      EOF
      fi

      # Handover /etc/nixos to ${user.username}
      chown ${user.username}.users /etc/nixos
      '';
  };

  # The NixOS release to be compatible with for stateful data such as databases.
  system.stateVersion = "16.03";
  system.autoUpgrade.enable = true;
}
```

---

NixOS at GA

https://github.com/GeoscienceAustralia/nixos-config

* NixOS lets us manage developers' workstations just as we manage code

* In GitHub repository `GeoscienceAustralia/nixos-config`, the `master` branch defines a
  generic workstation configuration.

* A CI job continuously builds the latest generic machine as a VirtualBox image and uploads
  it to an S3 bucket.

* Developers branch off `master` to create their customised workstation definitions.

* Developers keep up-to-date with the generic configuration by rebasing their branches.

---

Nix Shell, Project Environments

* For project-specific development environments, create `shell.nix` files in your
projects' root directories.

```
let
  pkgs = import <nixpkgs> {};
  geodesyDevEnv = with pkgs; buildEnv {
    name = "geodesyDevEnv";
    paths = [
      openjdk8
      maven
      graphviz
      python3
      awscli
    ];
  };
in
  pkgs.runCommand "setup-geodesy-dev-env" {
    buildInputs = [
      geodesyDevEnv
    ];
  } ""
```

* Running `nix-shell` will drop you into a bash shell with the specified
tools installed and in your path. Exiting the shell will de-activate the
temporary environment.

---

Nix Shell, Found a Bug in AWS CLI?

* Run: `nix-shell <nixpkgs> --attr awscli` to get the environment that the
  package manager would use (and has used) to build the AWS command line
  interface tools.

* Sources are at `$src`, patches at `$patches`, bash functions
  `unpackPhase`, `patchPhase`, `buildPhase`, etc., are bound to various
  build phases used by the package manager.
