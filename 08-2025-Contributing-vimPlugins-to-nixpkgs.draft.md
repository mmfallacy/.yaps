Nixpkgs is a community-maintained collection of software packages installble with `nix`. It serves as the main software repository for Nix, relying on community contributions to provide broad package coverage.

Although it already has broad coverage for Vim and Neovim plugins, some new plugins like [`brianhuster/live-preview.nvim`](https://github.com/brianhuster/live-preview.nvim) are still yet to be introduced. This yap documents what I did to introduce the `live-preview.nvim` package into Nixpkgs.

# Fork and clone `nixpkgs/master`

Since I already have an existing fork [mmfallacy/nixpkgs](https://github.com/mmfallacy/nixpkgs) which I used for introducing `mini.pick` a few months ago, we can sync and reuse that.

- Sync mmfallacy/nixpkgs via GitHub's interfaceo
- Clone mmfallacy/nixpkgs via `git clone github:mmfallacy/nixpkgs --depth 1`
  > [!NOTE] > `depth=1` ensures that we shallow clone the repository and include only the latest commit. This allows for faster cloning as we will not fetch unnecessary information. This flag also implies `--single-branch`, where we only clone the `master` branch and not the other unrelated branches

# Add the package using `nix run .#vimPluginsUpdater`.

Nixpkgs fortunately provides an easy way to add Vim and Neovim plugins via the `pkgs/applications/editors/vim/plugins/utils/updater.py`. This is reexported top-level as `packages.${system}.vimPluginsUpdater`[^1].

[^1]: https://nixos.org/manual/nixpkgs/stable/#updating-plugins-in-nixpkgs

Adding the `live-preview.nvim` vim plugin is done by running `nix run .#vimPluginsUpdater -- add "brianhuster/live-preview.nvim"`. This adds the repository URL to `**/vim/plugins/vim-plugin-names` and generates an attribute onto the attrset of `**/vim/plugins/generated.nix`. Afterwards, the updater script also adds the commit itself for our convenience. We can view the patch below:

```diff
diff --git a/pkgs/applications/editors/vim/plugins/generated.nix b/pkgs/applications/editors/vim/plugins/generated.nix
index 58f4d38bf..1815d5657 100644
--- a/pkgs/applications/editors/vim/plugins/generated.nix
+++ b/pkgs/applications/editors/vim/plugins/generated.nix
@@ -7496,6 +7496,19 @@ final: prev: {
     meta.hydraPlatforms = [ ];
   };

+  live-preview-nvim = buildVimPlugin {
+    pname = "live-preview.nvim";
+    version = "2025-08-17";
+    src = fetchFromGitHub {
+      owner = "brianhuster";
+      repo = "live-preview.nvim";
+      rev = "5890c4f7cb81a432fd5f3b960167757f1b4d4702";
+      sha256 = "0gr68xmx9ph74pqnlpbfx9kj5bh7yg3qh0jni98z2nmkzfvg4qcx";
+    };
+    meta.homepage = "https://github.com/brianhuster/live-preview.nvim/";
+    meta.hydraPlatforms = [ ];
+  };
+
   live-rename-nvim = buildVimPlugin {
     pname = "live-rename.nvim";
     version = "2025-06-23";
diff --git a/pkgs/applications/editors/vim/plugins/vim-plugin-names b/pkgs/applications/editors/vim/plugins/vim-plugin-names
index 954c16213..68358bfff 100644
--- a/pkgs/applications/editors/vim/plugins/vim-plugin-names
+++ b/pkgs/applications/editors/vim/plugins/vim-plugin-names
@@ -575,6 +575,7 @@ https://github.com/ldelossa/litee-filetree.nvim/,,
 https://github.com/ldelossa/litee-symboltree.nvim/,,
 https://github.com/ldelossa/litee.nvim/,,
 https://github.com/smjonas/live-command.nvim/,HEAD,
+https://github.com/brianhuster/live-preview.nvim/,HEAD,
 https://github.com/saecki/live-rename.nvim/,HEAD,
 https://github.com/azratul/live-share.nvim/,HEAD,
 https://github.com/ggml-org/llama.vim/,HEAD,

```

# Add overrides if necessary

Within `buildVimPlugin`, checks are run to ensure that `neovim` can require the module[^2]. This hook discovers all modules part of the plugin and attempts to require it via `require()`. This poses a problem with `live-preview.nvim` as the hook attempts to require `live-preview._spec` which runs tests upon requiring. To circumvent this issue, we can ignore it via `nvimSkipModules`.

Additionally, `live-preview.nvim` optionally depends on a picker plugin. As shown in the official repository, the user can opt to use any picker like `telescope.nvim`, `fzf.lua`, `mini.pick`, `snacks.nvim`. We add these as part of the `checkInputs` so that we can mark them as optional dependencies.

[^2]: https://github.com/NixOS/nixpkgs/pull/171064; https://nixos.org/manual/nixpkgs/stable/#testing-neovim-plugins-neovim-require-check

# Creating a PR

After the previous two steps, we can now push to our fork and proceed to open a pull request.
