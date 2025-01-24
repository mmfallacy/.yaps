# How did I port my Neovim configuration to NixOS

Neovim is a powerful editor that lets you configure it to meet your needs. It has support for a variety of lua plugins in addition to Vim's already powerful capabilities. This has been my go-to-choice for an editor as I became more interested in tinkering with my developer tools.

After a few months using Neovim, I started to long for a more configurable environment compared to Windows hence I ventured off on the rabbit hole we call Linux. I created an [Arch setup in 2023](https://github.com/mmfallacy/.dotfiles) to suit my needs as a developer. As I went on, configuring my system felt unsatisfactory due to the number of changes I have to make. Sure, tools like GNU Stow gave me hope in centralizing the changes I have to make to a single `.dotfiles` repository. But I still felt the need for a better solution -- a system that would let me declaratively configure everything in a single source of truth. After a lengthy search of potential solutions, I came across NixOS

[NixOS](https://nixos.org) is a Linux distribution that revolves around the nix package manager. Its philosophy can be summarized into a few words: declarative, immutable, and reproducible systems. Nix achieves this by placing all package installs in its `/nix/store`, and upon a `nixos-rebuild`, linking and composing the stored package into what eventually becomes as your system. This is what makes NixOS powerful, but it is also its main constraint. To further expound, software like my current Neovim configuration cannot install its own dependencies and plugins.

## How do NixOS users configure Neovim then?

Scouring through numerous tutorials, and even resorting to asking for help in NixOS's unofficial discord, we get a few promising solutions in order of decreasing complexity:

1. Wrapping neovim on your own.
2. [nixpkgs provided wrapper](https://github.com/NixOS/nixpkgs/blob/nixos-24.11/pkgs/applications/editors/neovim/utils.nix#L72)
3. `programs.neovim` of [NixOS](https://github.com/NixOS/nixpkgs/blob/nixos-24.11/nixos/modules/programs/neovim.nix)
4. `programs.neovim` of [Home-manager](https://github.com/nix-community/home-manager/blob/release-24.11/modules/programs/neovim.nix)
5. [NixVim](https://github.com/nix-community/nixvim),
   [nixCats](https://github.com/BirdeeHub/nixCats-nvim)

Option 1 requires the most effort initially as it requires basic understanding on how Neovim natively loads plugins on top of knowledge on how Nix handles things. Though, itt also gives you a more meaningful experience in writing wrappers for Nix compatibility. On the other hand, option 2 provides abstractions on concepts you'll normally directly deal with when going through Option 1, but with fancy helpers. You'll need less effort to do the wrapping yourself however you aref still required to pry apart how the helper functions work. Options 3 and 4 are the standard approaches to configuring software with Nix. NixOS provides module options for system-wide Neovim configurations, while Home-Manager allows for user-specific setups. Finally, option 5 subjects you into a lot more complexity and limitations (i.e. NixVim with its pure Nix neovim configurations).

Out of all these options, I decided to go with the fourth one as it would allow me to maintain a user-level Lua-based configuration without sacrificing simplicity.

## My own home-manager Neovim configuration

> For reference, here is the link for my [.nixconfig's neovim user module](https://github.com/mmfallacy/.nixconfig/tree/main/user/neovim) which is loosely based on this [LazyVim home-manager configuration](https://github.com/LazyVim/LazyVim/discussions/1972)

My home-manager Neovim configuration starts in [`user/neovim/default.nix`](https://github.com/mmfallacy/.nixconfig/blob/e69d9fdca1eab29c1ba7106aa2c1b7ca895c5b8f/user/neovim/default.nix). As I wanted my configuration to support a non-Nix setup, I decided to write two `init.lua`s which serves as the entry point.

One of these entry points (`user/neovim/config/init.lua`) is meant to be used by Neovim as the configuration folder is symlinked/stowed to a normal system's `.config/nvim`. This file contains the code required to install and set up my package manager of choice. Also, this lua code also checks whether it is being run on a NixOS system (thru the `IS_NIXOS`), showing a warning when run in a NixOS environment.

The other one, which is defined inside of the home-manager's `extraLuaConfig`, is the actual entry point used by home-manager. In addition to this entry point, we also symlink `user/neovim/config/lua` to `xdg.configFile."nvim/lua"` so neovim will be able to see our configuration.

> OPTIONAL! Since I tinker with my neovim config a lot, doing a normal (in-store) symlink would require me to rebuild my configuration through nix at every change. To circumvent this behavior, we can use an out-of-store symlink as such:

```nix
{
  xdg.configFile."nvim/lua".source =
    config.lib.file.mkOutOfStoreSymlink "${dotfiles}/user/neovim/config/lua";
}
```

> Observe here that we are required to pass the absolute file as we are using a flake-based configuration. Paths in a flake-based configuration are included in the store to keep your configuration pure. This is the behavior by default, hence passing that path to `mkOutOfStoreSymlink` will still create a symlink to a location in the store.

To further understand how the second entry point works, we classify neovim packages into four categories: Normal Lua plugins, Parsers installed by `nvim-treesitter`, LSPs, and external packages (e.g. `ripgrep`) which we can call as Extras. These packages are all installed through nix, but they vary on how we link them to Neovim.

### Normal Lua plugins

Normal Lua plugins are standard packages that are normally installed and loaded by a package manager like `lazy.nvim`. `lazy.nvim` is a modern plugin manager for Neovim. Among its features are plugin installation and updates, lazy loading, lockfiles, and version locking with full Semver support. It handles the plugin installation by doing partial clones of the plugin's git repository somewhere inside the user's home directory (typically in `~/.local/share/nvim/lazy` or `$XDG_DATA_HOME/nvim/lazy`). We can then replicate this plugin directory structure in Nix as such:

1. Create a `plugins.nix` file which returns a list of the plugins

```nix
# plugins.nix
{ pkgs, ... }:
with pkgs.vimPlugins;
[
  # Insert packages here!
  # someplugin-nvim
]
```

> TIP! You can check for available neovim plugins [here](https://github.com/NixOS/nixpkgs/tree/master/pkgs/applications/editors/vim/plugins). Most packages are in `generated.nix` but some are located elsewhere in the directory.

2. In another file `lazy.nix`, we can then transform this list into a list of entries.

```nix
# lazy.nix
{ lib, pkgs, ... }:
let
  # Load the plugin list from previous step
  plugins = import ./plugins.nix;

  # mapper function: element -> entry
  # an entry is an attrset { name=<name of symlink>; path=<source>; };
  mkEntryFromDrv =
    drv:
    if lib.isDerivation drv then
      {
        name = "${lib.getName drv}";
        path = drv;
      }
    else
      drv;

  entries = builtins.map mkEntryFromDrv plugins;
in
{ }
```

3. Create the link farm `lazy-plugins` and return the path;

```nix
# lazy.nix
{ lib, pkgs, ... }:
let
  # Previous steps
  #...

  lazypath = pkgs.linkFarm "lazy-plugins" entries;
in
lazypath
```

These steps gives us the in-store location (`lazypath`) of our replicated `lazy-plugins` directory full of symlinks. However, this is not yet enough to let Lazy handle the plugins. If we simply symlink `lazypath` to the normal plugin directory location (`$XDG_DATA_HOME/nvim/lazy`), Lazy would still not recognize these plugins as installed.

> Why? Refer to `lazy.nvim`'s source code on how it handles plugin loading.

Luckily, Lazy also allows the user to set a `dev` option containing a `path` and a `patterns` field. When set, lazy will source all plugins with names that match `dev.patterns` from the provided `dev.path`. We will use this to our advantage by setting `dev.path = lazypath` with `dev.patterns = {" "}` to target all packages. This will let Lazy know that we expect to source all plugins from a local directory in the nix store.

### Treesitter Parsers

While normal lua packages are handled by something like `lazy.nvim`, Treesitter parsers are also installed through `nvim-treesitter`'s `:TSInstall` command. Again, this is problematic as we want Nix to handle all package installations. Luckily, we can mimic how `nvim-treesitter` installs parsers in a similar fashion:

1. Create `treesitter.nix` to contain all treesitter related sections.

> Do observe that `nvim-treesitter` exists as a package available in Nixpkgs. This makes our lives a bit more easier.

```nix
# treesitter.nix
{ pkgs, ... }:
let
  inherit (pkgs.vimPlugins) nvim-treesitter;
in
{ }
```

2. Specify all grammars you want to include.

The `nvim-treesitter` nix package provides us a neat way of creating a derivation of `nvim-treesitter` along with the grammars.

```nix
# treesitter.nix
{ pkgs, ... }:
let
  inherit (pkgs.vimPlugins) nvim-treesitter;

  TSwithAllGrammars = nvim-treesitter.withAllGrammars;
  TSwithGrammars = nvim-treesitter.withPlugins (
    grammars: with grammars; [
      # Insert grammars you want to include
      # This list is conceptually equivalent to nvim-treesitter's ensure_installed
      # where you can define which grammars will be installed.
    ]
  );
in
{ }
```

3. Create the parser directory with all grammar parsers symlinked.

```nix
# treesitter.nix
{ pkgs, ... }:
# Previous steps
# ...
pkgs.symlinkJoin {
  name = "nvim-treesitter-grammars";
  # You can also use TSwithAllGrammars!
  # We use the dependencies attribute of the variants of treesitter we've created previously.
  paths = TSwithGrammars.dependencies;
}
```

> Do not forget to import this to get the path!

We then obtain the in-store location of `nvim-treesitter-grammars` as `parserpath`. Like `lazy-nvim`'s `dev.path`, `nvim-treesitter` also allows the user to set a `parser_install_dir` path. We can then use this to tell Neovim to look for parsers in `parserpath`. Lastly, ss per the documentation, we also need to add `parserpath` in neovim's runtimepath.

### LSPs

Before, I typically use an LSP manager like `mason.nvim` and `mason-lspconfig.nvim` to set up LSPs. [Mason](https://github.com/williamboman/mason.nvim) is a portable package manager for Neovim that helps you install LSP servers, DAP servers, linters and formatters. Since it handles the installation of these packages, we again cannot use it with nix. Unlike Lazy and `nvim-treesitter` though, we will drop these plugins as we will install the LSPs through nix and configure them through `nvim-lspconfig`. Compared to the previous package types, this is simpler as we only need to do the following:

1. Create `lsp.nix` to hold a list of all the LSPs we want to install.

```nix
{ pkgs, ... }:
with pkgs;
[
  # Insert LSPs, DAPs, formatters, here!
  lua-language-server
  stylua
]
```

2. Add this list to the `programs.neovim.extraPackages` to let `home-manager` handle the installation.

```nix
{ pkgs, ... }:
let
  # Get the list of LSPs
  lsps = import ./lsp { inherit pkgs; };
in
{
  programs.neovim = {
    enable = true;
    extraPackages =
      with pkgs;
      [
        # Other packages
      ]
      ++ lsps;
  };
}
```

Consequently, after installation of these packages, `home-manager` will also add their binaries to Neovim's PATH.

> We can also check if these binaries (e.g. Lua LSP) are available through running the following in Neovim
> `:echo executable('lua-language-server')`

# Extras

Some lua plugins may have external dependencies. Binaries like `ripgrep` and `fd` can be used with `telescope.nvim` as a faster `grep` and `find`. Like the LSPs, these packages will be installed by `home-manager` through the `programs.neovim.extraPackages`

```nix
{ pkgs, ... }:
let
  # Get the list of LSPs
  lsps = import ./lsp { inherit pkgs; };
in
{
  programs.neovim = {
    enable = true;
    extraPackages =
      with pkgs;
      [
        # Other packages
        ripgrep
        fd
      ]
      ++ lsps;
  };
}
```

> Include also a clipboard provider (`xclip` for X11; `wl-clipboard` for wayland) to enable yanking to system clipboard

# Other steps

Lastly, we need to set a few overrides to ensure that our neovim configuration will not misbehave in a NixOS system.

1. `lazy.nvim` should not fallback to git nor install missing plugins.
   This is set by `lazy.opts.dev.fallback = false` and `lazy.opts.install.missing = false`.

2. Include `parserpath` in the runtimepath.
   Simply adding it through `vim.opt.rtp:prepend` will fail as Lazy clears the runtimepath upon setup for performance reasons.
   We can set this through `lazy.opts.performance.rtp.paths = { "${parserpath}" }`

   > Do note that we can interpolate `parserpath` in the `extraLuaConfig` portion by using nix's `${}`. Do not forget to surround it in quotes!

3. `nvim-treesitter` should not auto install parsers!
   This is set by clearing `nvim-treesitter.opts.ensure_installed`.
   > `ensure_installed` is a lua table that contains the parsers `nvim-treesitter` expects to install. I created a utility function in my own `treesitter.lua` that notifies the user if a parser is missing.


