
# Abstract

Diese Seite beschreibt die Konfiguration des vi Standardverhaltens.

# Konfiguration

Die Einstellungen werden im File ~/.vimrc festgelegt:

```bash
$ cat ~/.vimrc

" deactivate syntax highlighting
syntax off

" deactivate auto ident
set noai

" expand tabstops to whitespaces
set expandtab

" use two spaces as tabstop
set tabstop=2

" disable comment continuation on new line or paste
autocmd FileType * setlocal formatoptions-=c formatoptions-=r formatoptions-=o
```

### Tags:

[[HowTo]] - [[Vim]] - [[Linux]]