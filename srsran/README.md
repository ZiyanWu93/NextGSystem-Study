# srsran/ — the code under study

The [srsRAN Project](https://github.com/srsran/srsRAN_Project) gNB, to be pinned
here as a git submodule at a fixed commit so every teammate studies the exact
same version. Read-only baseline — we analyze it, we do not fork it.

**Not added yet.** Once this project is its own git repo, pin it with:

```sh
git submodule add https://github.com/srsran/srsRAN_Project srsran
cd srsran && git checkout <commit-sha> && cd ..
git add srsran .gitmodules && git commit -m "Pin srsRAN core under study"
```

Empty until then — expected at this stage.
