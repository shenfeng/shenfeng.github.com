---
author: feng
date: '2011-12-24 20-21-00'
layout: post
status: publish
desc: A piece of elisp code to jump to the last edit position scroll session
title: Elisp, jump to last edit location across whole session
categories:
- emacs
---

### Scenario

Coding, editing ---> an interesting function ---> anther interesting function
---> ...

"My god, where have I been, how can I jump right back to the last edit position"

### How other handle it
Eclipse has a handing feature: `Last Edit Location`, binding to `Ctrl+Q`,
It works `globally`:  all visited files, even those being edited, but
currently closed.

### Exiting solution

* [EmacsSession](http://emacs-session.sourceforge.net/) has a function
  called `session-jump-to-last-change`, but per buffer
* [GotoLastChange](http://www.emacswiki.org/emacs/GotoLastChange), but
  per buffer

### Elisp to the rescue
I've being using emacs for several month, it serves me well, it lacks
this very handing feature. Thanks to elisp, I can implement
it myself.

### Code
{% highlight scheme %}

;;; record two different file's last change. cycle them
(defvar feng-last-change-pos1 nil)
(defvar feng-last-change-pos2 nil)

(defun feng-swap-last-changes ()
  (when feng-last-change-pos2
    (let ((tmp feng-last-change-pos2))
      (setf feng-last-change-pos2 feng-last-change-pos1
            feng-last-change-pos1 tmp))))

(defun feng-goto-last-change ()
  (interactive)
  (when feng-last-change-pos1
    (let* ((buffer (find-file-noselect (car feng-last-change-pos1)))
           (win (get-buffer-window buffer)))
      (if win
          (select-window win)
        (switch-to-buffer-other-window buffer))
      (goto-char (cdr feng-last-change-pos1))
      (feng-swap-last-changes))))

(defun feng-buffer-change-hook (beg end len)
  (let ((bfn (buffer-file-name))
        (file (car feng-last-change-pos1)))
    (when bfn
      (if (or (not file) (equal bfn file)) ;; change the same file
          (setq feng-last-change-pos1 (cons bfn end))
        (progn (setq feng-last-change-pos2 (cons bfn end))
               (feng-swap-last-changes))))))

(add-hook 'after-change-functions 'feng-buffer-change-hook)
;;; just quick to reach
(global-set-key (kbd "M-`") 'feng-goto-last-change)

{% endhighlight %}
