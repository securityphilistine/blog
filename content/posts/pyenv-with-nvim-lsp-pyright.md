---
title: "Pyenv with Neovim Pyright LSP"
date: 2022-10-01T18:02:06+01:00
tags: ['snippets', 'neovim', 'pyenv']
---

# Getting Pyright LSP server in Neovim to work with Pyenv .python-version file

For python development projects at home, I'm using a tool called [Pyenv](https://github.com/pyenv/pyenv) to manage my installed python versions and an plugin for Pyenv called [pyenv-virtualenv](https://github.com/pyenv/pyenv-virtualenv) to manage my python virtual environments for various projects.

Pyenv-virtualenv in particular has some neat functionality where if you add a `.python-version` file to a directory containing the name of the pyenv virtualenv to use, it will automatically activate the virtual environment upon entering the directory and deactivate when leaving the directory. 

I'm also beginning my Vim journey, and I'm trying to set up Neovim as an IDE that can replace Pycharm. I've been following [chris@machine's](https://www.youtube.com/c/ChrisAtMachine) tutorial series [Neovim from Scratch series](https://www.youtube.com/watch?v=ctH-a-1eUME&list=PLhoH5vyxr6Qq41NFL4GvhFp-WLd5xzIzZ) which I highly recommend.

I'm at the stage where I'm looking at [Neovim LSP servers](https://www.youtube.com/watch?v=6F3ONwrCxMg&list=PLhoH5vyxr6Qq41NFL4GvhFp-WLd5xzIzZ&index=8) to get syntax highlighting in my code, I wanted to use the Pyright LSP server for my Python projects. 

However I realised that the Pyright LSP server does not pick up pyenv-virtualenv environments. I've seen a bunch of posts online trying to deal with the issue:
- https://stackoverflow.com/questions/65847159/how-to-set-python-interpreter-in-neovim-for-python-language-server-depending-on
- https://www.reddit.com/r/neovim/comments/n5gcsx/pyright_luaconfig_not_getting_my_pyenv_environment/

Generally folks online have been setting up a `.pyrightconfig.json` file in their project root that sets the `venvPath` and `venv` settings to pick up the intended project directory. E.g:
```
{
    "venvPath": "/home/USERNAME/.pyenv/versions/",
    "venv": "MY-VENV"
}
```
This works completely fine and allows you to set other custom settings for Pyright on a per project basis. 

However at this stage I don't have the need to customize Pyright beyond just getting it working with pyenv-virtualenv. 
I just want Pyright in Neovim to pick up the pyenv-virtualenv set in the `.python-version` file. 

I've found that you can get this working by setting the `venvPath` setting in the LSP config Lua. In the Neovim from scratch Lua code, when the various LSP servers are initialised they get set up [with custom settings on initialisation](https://github.com/LunarVim/Neovim-from-scratch/blob/06-LSP/lua/user/lsp/lsp-installer.lua).

In [user.lsp.settings.pyright](https://github.com/LunarVim/Neovim-from-scratch/blob/06-LSP/lua/user/lsp/settings/pyright.lua) we can set the same Pyright settings as we would in a `.pyrightconfig.json`, except these settings would be applied globally for Neovim. 

```
return {
	settings = {

    python = {
      analysis = {
        typeCheckingMode = "off"
      },
      venvPath = "/home/USERNAME/.pyenv/versions/"
    }
	},
}
```

Additionally, we only need to set the `venvPath` variable as when we're in a pyenv-virtualenv the `$VIRTUAL_ENV` environment variable will be set to the virtualenv name. 
As such Pyright should be able to pick up and use the correct virtualenv for the project. 

I hope this helps other folks who are similarly at the beginning of their Neovim journey! 
