+++
title = "Org Mode - Pasting with Formatting"
date = 2025-01-16
draft = false
+++

This is a script that lets you paste rich text from other applications such as web browsers or word processors into Emacs Org Mode with formatting included (such as italics, bold, and headings).
I have configured the script in a way that it detects the OS and defines the corresponding script accordingly, therefore you can use the same Emacs init file on both Windows and Linux.

Please note that for both OS, `pandoc` needs to be installed and in the PATH.
As for Linux, `xclip` also needs to be installed.

```elisp
;; Command for Windows 11 (as of 24H2, which uses Windows PowerShell 5.1), adapted from https://emacs.stackexchange.com/questions/12121/org-mode-parsing-rich-html-directly-when-pasting
(when (string-equal system-type "windows-nt")
            (defun org-formatted-yank ()
  "Convert clipboard contents from HTML to Org and then paste (yank)."
  (interactive)
  ;; Uses PowerShell to get the clipboard text in HTML format
  ;; Pandoc first simplifies to JSON, before converting to Org Mode format and pastes.
  ;; --wrap=none is used so the text isn't artificially wrapped.
  (kill-new (shell-command-to-string "powershell -command \"Get-Clipboard -TextFormatType html\" | pandoc -f html -t json | pandoc -f json -t org --wrap=none"))
  (yank)))

;; Command for Linux X11 Systems with xclip installed (tested on Linux Mint 22 Wilma, Cinnamon Edition)
(when (string-equal system-type "gnu/linux")
            (defun org-formatted-yank ()
  "Convert clipboard contents from HTML to Org and then paste (yank)"
;; (interactive) flags the code as a command that can be executed via M-x
  (interactive)
  ;; Creates the clipboard-contents variable and sets it to be an empty string
  (defvar clipboard-contents "")
  ;; Changes the clipboard-contents variable by running a shell command and getting the output of that to the variable.
  ;; If xclip (which grabs the clipboard contents) doesn't have the right format available, it outright freezes. Hence "timeout 0.5".
  ;; Why this is run: it's checking the clipboard to see if it has html saved inside.
  ;; If there is html, clipboard-contents will be filled with the clipboard contents.
  ;; If there is NO html, clipboard-contents will remain blank.
  (setq clipboard-contents (shell-command-to-string "timeout 0.5 xclip -o -selection clipboard -t text/html"))
  ;; If the variable clipboard-contents is NOT blank (in other words, if the clipboard had html)
  (if (not (equal "" clipboard-contents))
          ;; Execute these two commands:
          ;; 1. Use pandoc to convert the html in the clipboard first to simplified JSON, then to org mode. Copy the result to the clipboard. --wrap=none is used so the text isn't artificially wrapped.
          ;; 2. yank pastes it to the current document
          ;; NOTE: progn is used because otherwise, only one command can actually be run inside the conditional.
          (progn
            (kill-new (shell-command-to-string "xclip -o -selection clipboard -t text/html | pandoc -f html -t json | pandoc -f json -t org --wrap=none"))
            (yank)
            )
            ;; If the clipboard-contents IS blank (in other words, if it does not have html), then Emacs will display in the messages pane, "No html in clipboard."
          (message "No html in clipboard.")
        )))

;; org-formatted-yank common settings
;; Making sure that Org Mode parses Pandoc's non-breaking spaces properly
;; This one is for the beginning char
(setcar org-emphasis-regexp-components " \t('\" {")
;; This one is for the ending char.
(setcar (nthcdr 1 org-emphasis-regexp-components) "- \t.,: !?;'\")}\\")

;; Setting the shortcut to Ctrl-Shift-y
(global-set-key (kbd "C-S-y") 'org-formatted-yank)
```
