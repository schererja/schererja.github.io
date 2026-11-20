# Hugo blog trial

The sample uses PaperMod, pinned to commit `118119693d8b5ac211b939775678310973bcb232` as a Git submodule. It was verified with Hugo v0.170.1 Extended. The trial lives in `hugo/` and the existing Jekyll workflow still builds from the repository root.

Run these commands from the repository root.

```sh
git submodule update --init hugo/themes/PaperMod
hugo server --source hugo
```

The Hugo trial contains a snapshot of the published Markdown articles and the About page. Use the root Jekyll files for production content edits while evaluating the trial. The existing unpublished drafts remain in `_drafts/`.
