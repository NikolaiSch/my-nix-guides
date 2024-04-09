# How to package nextjs app with dream2nix

- Structure of the project

```bash
.
├── my-next-app
│   ├── package.json
│   ├── package-lock.json
│   ├── ...
├── default.nix
└── flake.nix
```

- Create a `default.nix` file in the root of the project

```nix
# default.nix
{ config, dream2nix, ... }:
{
  # Import the dream2nix modules we need
  imports = [
    dream2nix.modules.dream2nix.nodejs-package-lock-v3
    dream2nix.modules.dream2nix.nodejs-granular-v3
  ];

  # Define the derivation for the project
  # only need the src attribute
  mkDerivation = {
    src = ./my-next-app;
  };

  # if the project has any dependencies add them here
  deps =
    { nixpkgs, ... }:
    {
      inherit (nixpkgs) stdenv nodejs;
    };

  # define where the output package-lock.json is
  nodejs-package-lock-v3 = {
    packageLockFile = "${config.mkDerivation.src}/package-lock.json";
  };

  # can parse json from package.json to set these values
  name = "my-next-app";
  version = "0.1.1";
}

```

- Create a `flake.nix` file in the root of the project

```bash
  nix flake init
```

- Add the following to the `flake.nix` file

```nix
{
  description = "My flake with dream2nix packages";

  # Define the inputs to the flake
  inputs = {
    dream2nix.url = "github:nix-community/dream2nix";
    nixpkgs.follows = "dream2nix/nixpkgs";
  };

  outputs =
    inputs@{
      self,
      dream2nix,
      nixpkgs,
      ...
    }:
    let
      # add the system to be built on
      system = "aarch64-darwin";
      # get the package set for the system
      pkgs = import nixpkgs { inherit system; };
      # create your nextjs package
      package = dream2nix.lib.evalModules {
        packageSets.nixpkgs = inputs.dream2nix.inputs.nixpkgs.legacyPackages.${system};
        modules = [
          ./default.nix
          {
            paths.projectRoot = ./.;
            paths.projectRootFile = "flake.nix";
            paths.package = ./.;
          }
        ];
      };

      # create a shell script to run the nextjs app
      shellScript = pkgs.writeShellApplication {
        name = "start-next";

        # give it nodejs when running
        runtimeInputs = [ pkgs.nodejs ];


        text = ''
          cd ~/.nix-profile/lib/node_modules/my-next-app
          ${pkgs.nodejs}/bin/npm run start
        '';
      };
    in

    {
      # Define the outputs of the flake
      packages.${system} = {
        default = package;
        start-next = shellScript;
      };
    };
}
```

- Run the following command to build the nextjs app

```bash
nix build
```

- Install the package to the system

```bash
nix profile install .
nix profile install .#start-next
```

- Run the nextjs app

```bash
start-next
```
