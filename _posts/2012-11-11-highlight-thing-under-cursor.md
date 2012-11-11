---
author: feng
date: '2012-11-11 08-21-00'
layout: post
status: publish
desc: Emacs高亮光标下的单词, Emacs hilight thing under cursor, jumping bettween them, and replace
title: Emacs高亮光标下的单词
categories: ['emacs']
---

用Emacs一年半了，刚开始时用了
[Emacs Starter Kit](https://github.com/technomancy/emacs-starter-kit)，
它很不错，帮了我很多：

>The Starter Kit provides a more pleasant set of defaults than you get
>normally with Emacs. It was originally intended for beginners, but it
>offers a nicely augmented working environment for anyone using Emacsø

最近，产品经理在构思产品思路，拉大家开会。**工欲善其事，必先利其器**，
我忙里偷闲，打磨打磨工具，为接下来的战斗积蓄势能。
重新整理了Emacs的配置文件，去掉了Emacs Starter Kit，写了一些elisp函数
提高我作为程序员的生活品质，
[在github的代码](https://github.com/shenfeng/dotfiles/tree/master/emacs.d)

## [feng-lastchange](https://github.com/shenfeng/dotfiles/blob/master/emacs.d/feng-lastchange.el)

简化并改进了[原来写的](http://shenfeng.me/emacs-last-edit-location.html)类似于
Eclipse `Ctrl-q` ， Intellj `Shift+Command+Delete` Last Edit Location

## [feng-highlight](https://github.com/shenfeng/dotfiles/blob/master/emacs.d/feng-highlight.el)

Eclipse和Intellj都有高亮光标下变量，函数等功能，挺实用。
![image](/imgs/eclipse-hl.png)

Emacs也有类似的：`idle-highlight-mode` 。

我常常这样干活：一个变量，`M-b…`到开始，`C-s`，`C-w….`选择，`C-s`搜索定位，如
果`idle-highlight-mode`有这样的附加功能：

**高亮当前buffer中光标下的东东，能快速到上一个/下一个出现的地方，有替换功能**

我的生活品质会提高不少。

*自己动手，丰衣足食*。

昨晚，光棍节的前一晚，熬到两点，敲了几十行
[elisp代码](https://github.com/shenfeng/dotfiles/blob/master/emacs.d/feng-highlight.el)
实现了我想要的快捷键

{% highlight scheme %}
(global-set-key (kbd "M-i") 'feng-highlight-at-point)
{% endhighlight %}

* `M-i` 执行 `feng-highlight-at-point` 启动高亮
* `M-n` 上一个
* `M-p` 下一个
* `M-r` 替换

![image](/imgs/emacs-hl.png)

今天光棍节，上午试着用了一会儿，截图就是应用于golang。

向上/向下/替换是不错的附加功能，多么希望Eclipse/Intellj也有类似顺手的功能和快捷键。

对比Eclipse/Intellj， 他们对代码有语义分析，我这仅基于文本。也有好处：
`if`, `else`照样高亮。

下午陪女朋友去逛街买衣服了。

[完整代码](https://github.com/shenfeng/dotfiles/blob/master/emacs.d/feng-highlight.el)：
{% highlight scheme %}
(require 'thingatpt)

(defface feng-highlight-at-point-face
  `((((class color) (background light))
     (:background "light green"))
    (((class color) (background dark))
     (:background "royal blue")))
  "Face for highlighting variables"
  :group 'feng-highlight)

(defvar current-highlighted nil)
(make-variable-buffer-local 'current-highlighted)

(defun feng-at-point-prev ()
  (interactive)
  (when current-highlighted
    (unless (re-search-backward
             (concat "\\<" (regexp-quote current-highlighted) "\\>") nil t)
      (message "search hit TOP, continue from BOTTOM")
      (goto-char (point-max))
      (feng-at-point-prev))))

(defun feng-at-point-next ()
  (interactive)
  (when current-highlighted
    (forward-char (+ 1 (length current-highlighted))) ; more out of current
    (if (re-search-forward
         (concat "\\<" (regexp-quote current-highlighted) "\\>") nil t)
        (backward-char (length current-highlighted))
      (backward-char (+ 1 (length current-highlighted)))
      (message "search hit BOTTOM, continue from TOP")
      (goto-char (point-min))
      (feng-at-point-next))))

(defun feng-at-point-replace ()
  (interactive)
  (when current-highlighted
    (save-excursion
      (goto-char (point-min))           ;; back to top
      (feng-at-point-next)              ;; find first
      (setq isearch-string current-highlighted)
      (isearch-query-replace))))

(defvar feng-highlight-at-point-keymap
  (let ((map (make-sparse-keymap)))
    (define-key map (kbd "M-n") 'feng-at-point-next)
    (define-key map (kbd "M-r") 'feng-at-point-replace)
    (define-key map (kbd "M-p") 'feng-at-point-prev)
    map))

(defun feng-highlight-at-point ()
  (interactive)
  (remove-overlays (point-min) (point-max) 'feng-highlight t)
  (let* ((target-symbol (symbol-at-point))
         (target (symbol-name target-symbol)))
    (when target-symbol
      (setq current-highlighted target)
      (save-excursion
        (goto-char (point-min))
        (let* ((regexp (concat "\\<" (regexp-quote target) "\\>"))
               (len (length target))
               (end (re-search-forward regexp nil t)))
          (while end
            (let ((ovl (make-overlay (- end len) end)))
              (overlay-put ovl 'keymap feng-highlight-at-point-keymap)
              (overlay-put ovl 'face 'feng-highlight-at-point-face)
              (overlay-put ovl 'feng-highlight t))
            (setq end (re-search-forward regexp nil t))))))))

(provide 'feng-highlight)

{% endhighlight %}
