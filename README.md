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

  nixosConfigurations = { self }: {
    default = self.inputs.nixpkgs.lib.nixosSystem {
      modules = [ self.modules.default ];
    };
  };

  pkgsConfig = { allowUnfree = true; cudaSupport = true; };
  pkgsOverlays = { self, overlays }: [
    self.inputs.foo.overlays.default
    self.overlays.default
  ];

  # Optional explicit import of nixpkgs, allows for further customization
  pkgsForSystem = { self }:
    import self.inputs.nixpkgs {
      config = self.pkgsConfig;
      overlays = self.pkgsOverlays;
    };

  outputs = { pkgs, ... }: with pkgs; {
    formatter = nixpkgs-fmt;

    checks.default = myPackage;

    packages.default = myPackage;

    # Instead of using { type = app; program = <store path>;}, just use <store path>
    app.default = myPackage;
    app.other = "${myPackage}/bin/other-bin";

    devShell.default = mkShell { name = "dev"; inputsFrom = [ myPackage ]; };
  }

  # These can just import a whole directory from a path
  templates.default = ./templates/default;
}
```

## Minimal Example
```nix
{

  description = "Minimal better flake";

  inputs.nixpkgs.url = "github:nixos/nixpkgs/unstable";

  # Overlay defines myPackage
  overlays.default = import ./nix/overlay.nix;

  outputs = { pkgs }: with pkgs; {
    packages.default = myPackage;
    devShell.default = mkShell { name = "dev"; inputsFrom = [ myPackage ]; };
  }
}

# nix/overlay.nix
final: prev: {
  myPackage = final.callPackage ./package.nix { };
}
```

The evaluation of these results would roughly be:
```nix
# The ? here is that these could be optionally imported if defined
flake = let
  # Initial import with unknown attrs defined, inputs would also need to be resolved as part of this "import"
  raw = import ./flake.nix;
  # Evaluate enough to get a nixpkgs import
  pkgsConfig = if raw ? pkgsConfig then pkgsConfig { self = raw; } else { };
  pkgsOverlays = if raw ? pkgsOverlays then pkgsOverlays { self = raw; } else [ ];
  # Replace pkgsConfig and pkgsOverlays functions with their arguments applied results
  callRawFlake = callPackageWith (raw // { inherit pkgsConfig pkgsOverlays; });

  mkFlake = {
    self,
    pkgsConfig,
    pkgsOverlays,
    # Could be defined by user, but if not, do an expected import
    pkgsForSystem ? { self, system }: import self.inputs.nixpkgs { config = pkgsConfig; overlays = pkgsOverlays;);
  }: let
    pkgs = pkgsForSystem { inherit self; system = builtins.getCurrentSystem; };
    # TODO: nixosConfigurations and other attrs would optinally need to be called if they're functions and present
  in raw.outputs pkgs // { inherit pkgsConfig pkgsOverlays; };

  # Create self fixed point, and pass as flake output
  final = raw // (callRawFlake mkFlake) // { self = final; };
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

