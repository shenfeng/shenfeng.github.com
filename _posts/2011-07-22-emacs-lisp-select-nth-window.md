---
author: feng
date: '2011-07-22 20-21-00'
layout: post
status: publish
desc: A piece of elisp code to quickly switch to the nth window
title: Elisp, select the nth visible window quickly
categories:
- emacs
---

我们在昆山的团队中另外3个都用[GNU Emacs](www.gnu.org/software/emacs/)，
为了更容易的和他们的代码风格保持一致，
我也试着学了一下。一学也有三个月了。

开发用的显示器分辨率是1920x1080,
Emacs可以很好的使用这些空间，
通过[hjiang](http://hjiang.net)的
[smart-split](http://hjiang.net/archives/253)，可以把
一个Emacs frame分成[3个window](/imgs/emacs-3-window.png)，
通过other-window(C-x o)在这些window之间进行切换。
因为需要按Ctrl, 折磨小拇指， 并且也不是很方便。

我琢磨着写了下面的函数，可以快速切换到指定窗口。

{% highlight scheme %}
(defun select-nth-window (n)
  "Select the nth visible window of current frame,
window are ordered by top-left point"
  (let* ((cmp (lambda (l r)
                (if (= (second l) (second r))
                    (< (third l) (third r))
                  (< (second l) (second r)))))
         (windows (sort
                   (mapcar (lambda (w)
                             (cons w (window-edges w)))
                           (window-list)) cmp))
         (index (- (min n (length windows)) 1)))
    (first (nth index windows))))

(defun select-first-window ()
  "Select the top-left most window"
  (interactive)
  (select-window (select-nth-window 1)))

(defun select-second-window ()
  (interactive)
  (select-window (select-nth-window 2)))

(defun select-third-window ()
  (interactive)
  (select-window (select-nth-window 3)))

(global-set-key (kbd "M-1") 'select-first-window)
(global-set-key (kbd "M-2") 'select-second-window)
(global-set-key (kbd "M-3") 'select-third-window)
{% endhighlight %}

我改了一个[hjiang](http://hjiang.net)的
[smart-split](http://hjiang.net/archives/253)
来更好的满足我的需要：
1. 尽量等分Frame， 在1920x1080的显示器上为78个字符（依赖字体，字号）
2. 给分出的window，装上不同的user buffer(名字不是以\*开始的）

{% highlight scheme %}
(defun smart-split ()
  "Split the window into near 80-column sub-windows, try to
equally size every window"
  (interactive)
  (defun compute-width-helper (w)
    "More than 220 column, can be split to 3, else 2"
    (if (> (window-width w) 220) 80
      (+ (/ (window-width w) 2) 1)))
  (defun smart-split-helper (w)
    "Helper function to split a given window into two, the first of which has
    80 columns."
    (if (> (window-width w) 130)
        (let* ((w2 (split-window w (compute-width-helper w) t))
               (i 0))
          (with-selected-window w2
            (next-buffer)
            (while (and (string-match "^*" (buffer-name)) (< i 20))
              (setq i (1+ i))
              (next-buffer)))
          (smart-split-helper w2))))
  (smart-split-helper nil))

;; bind to F12 for quick access
(global-set-key [f12] 'smart-split)
{% endhighlight %}

我还bind了F1为delete-other-windows（default C-x 1）
{% highlight scheme %}
(global-set-key [f1] 'delete-other-windows)
{% endhighlight %}

这样就可以F1, F12, M-1, M-2, M-3进行快速的切换。倒是挺方便。
