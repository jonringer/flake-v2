## Flake Schema redefined

This document is to propose a less awkward way to do flake evaluation.

### Current problems with existing flake schema

- Embedded system in attr paths (e.g. `packages.x86_64-linux.default`)
- Hard to combine non-system specific items with system specific items (e.g. overlays vs packages)
- "Diamond dependency issues" when bringing libraries from many flakes (each has an opinionated dependency tree)

## Proposed Solution

- Make overlays a hard requirement for composition
- Split up outputs, some specific to a system, and other are more generalized (e.g. overlays, NixOS modules)

## Example potential flake structure.

```nix
{

  description = "Example better flake";

  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/unstable";
  };

  overlays = {
    default = import ./nix/overlay.nix;
  };

  # Attrset of modules
  modules = import ./nixos;

  nixosConfigurations = { inputs, self }: {
    default = inputs.nixpkgs.lib.nixosSystem {
      modules = [ self.modules.default ];
    };
  };

  pkgsForSystem = { nixpkgs, system, self }:
    import nixpkgs {
      inherit system;
      config = { allowUnfree = true; cudaSupport = true; };
      overlays = [ input2.overlays.default self.overlays.default ];
    };

  outputsForSystem = { nixpkgs, pkgs, self }: {
    formatter = pkgs.nixpkgs-fmt;

    checks.default = pkgs.myPackage;

    packages.default = pkgs.myPackage;

    # Instead of using { type = app; program = <store path>;}, just use <store path>
    app.default = pkgs.myPackage;
    app.other = "${pkgs.myPackage}/bin/other-bin";

    devShell.default = pkgs.mkShell { name = "dev"; inputsFrom = [ pkgs.myPackage ]; };
  }

  # These can just import a whole directory from a path
  templates.default = ./templates/default;
}
```

## Minimal Example
```nix
{

  description = "Minimal better flake";

  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/unstable";
  };

  overlays = {
    default = import ./nix/overlay.nix;
  };

  pkgsForSystem = { nixpkgs, system, self }:
    import nixpkgs {
      inherit system;
      overlays = [ self.overlays.default ];
    };

  outputsForSystem = { nixpkgs, pkgs, self }: {
    packages.default = pkgs.myPackage;
    devShell.default = pkgs.mkShell { name = "dev"; inputsFrom = [ pkgs.myPackage ]; };
  }
}
```

The evaluation of these results would roughly be:
```nix
flake = let
  raw = import ./flake.nix;
  self = { inherit (raw) inputs overlay modules nixosConfigurations; };
  systemInputs = inputs // { system = builtins.getCurrentSystem; inherit self; };
  pkgs = raw.pkgsForSystem systemInputs;
  systemOutputs = raw.outputsForSystem (systemInputs // { inherit pkgs; });
  final = systemOutputs // self // { self = final; };
in
  final
```

To change the system, usage of the --system argument would be made
```bash
nix eval --system aarch64-linux .#myPackages.drvPath
```

### How were the problems solved?

- Embedded system in attr paths (e.g. `packages.x86_64-linux.default`)
  - Make system selection implicit in the structure, passed through --system flag
- Hard to combine non-system specific items with system specific items (e.g. overlays vs packages)
  - non-system specific items now have their own dedicated top-level attribute.
- "Diamond dependency issues" when bringing libraries from many flakes (each has an opinionated dependency tree)
  - Usage of overlays to give a single pkgs instance avoids the "1,000 glibc" and diamnond dependency issues.

