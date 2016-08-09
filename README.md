Syntax highlighting is great. Emacs has lots of modules for such. And
Emacs extensibility allow one to tweak it to ones liking. Here's a
quick screenshot demonstrating how I use colors in my source code, and
how color schemes are created with some fairly simple algorithms and
code:

![](scratch.png?raw=true)

The image above demonstrates two things; 1) My `gen-col-list` function
that creates color palettes, using the Golden Ratio method as
descriped here:

http://martin.ankerl.com/2009/12/09/how-to-create-random-colors-programmatically/

and 2) Such a color palette used with Emacs 'rainbow-delimiters-mode'
(look at the end of lines with repeating parenthesis in different colors).

Rainbow-delimiter schemes are typically either made to be really
visually distinct or use some kind of gradient. Distinct colors make
it easy to recognize pairs, while gradients look nice, but are
probably less useful for identifying matching pairs.

I like to use it with distinct colors, but have never quite found a
scheme I like. From some data visualization work I've done I know
basic color theory, and there are actually methods where a computer
can generate "visually optimal" distinct colors, even without actually
knowing the number of different colors needed in advance.

So for the heck of it I decided to use one such method for generating
colors schemes for rainbow-delimiters in emacs. To make it easy to
understand the examples I also wanted to find a way to actually
colorize the result which was quite easy as emacs supports CSS style
color descriptors.

To colorize CSS coded colors in the current buffer, install the
following function into your .emacs and activate it on any buffer you
want to colorize:

```lisp
;; Takes a color string like #ffe0e0 and returns a light
;; or dark foreground color to make sure text is readable.
(defun fg-from-bg (bg)
  (let* ((avg (/ (+ (string-to-number (substring bg 1 3) 16)
                    (string-to-number (substring bg 3 5) 16)
                    (string-to-number (substring bg 5 7) 16)
                    ) 3)))
    (if (> avg 128) "#000000" "#ffffff")
    ))

;; Improved from http://ergoemacs.org/emacs/emacs_CSS_colors.html
;; * Avoid mixing up #abc and #abcabc regexps
;; * Make sure dark background have light foregrounds and vice versa
(defun xah-syntax-color-hex ()
  "Syntax color text of the form 「#ff1100」 and 「#abc」 in current buffer.
URL `https://github.com/mariusk/emacs-color'
Version 2016-08-09"
  (interactive)
  (font-lock-add-keywords
   nil
   '(
     ("#[ABCDEFabcdef[:digit:]]\\{6\\}"
      (0 (progn (let* ((bgstr (match-string-no-properties 0))
                       (fgstr (fg-from-bg bgstr)))
                  (put-text-property
                   (match-beginning 0)
                   (match-end 0)
                   'face (list :background bgstr :foreground fgstr))))))
     ("#[ABCDEFabcdef[:digit:]]\\{3\\}[^ABCDEFabcdef[:digit:]]"
      (0 (progn (let* (
                       (ms (match-string-no-properties 0))
                       (r (substring ms 1 2))
                       (g (substring ms 2 3))
                       (b (substring ms 3 4))
                       (bgstr (concat "#" r r g g b b))
                       (fgstr (fg-from-bg bgstr)))
                  (put-text-property
                   (match-beginning 0)
                   (- (match-end 0) 1)
                   'face (list :background bgstr :foreground fgstr)
                   )))))
     ))
  (font-lock-fontify-buffer))
```
      
When you see screenshots with CSS color code in color, the code above
is what does the actual coloring.

So what does the gen-col-list look like? Here:

```lisp
;; Generates a list of random color values using the
;; Golden Ratio method described here:
;;   http://martin.ankerl.com/2009/12/09/how-to-create-random-colors-programmatically/
;; The list will be length long. Example:
;;
;; (gen-col-list 3 0.5 0.65)
;; => ("#be79d2" "#79d2a4" "#d28a79")
(require 'color)
(defun gen-col-list (length s v &optional hval)
  (cl-flet ( (random-float () (/ (random 10000000000) 10000000000.0))
          (mod-float (f) (- f (ffloor f))) )
    (unless hval
      (setq hval (random-float)))
    (let ((golden-ratio-conjugate (/ (- (sqrt 5) 1) 2))
          (h hval)
          (current length)
          (ret-list '()))
      (while (> current 0)
        (setq ret-list
              (append ret-list 
                      (list (apply 'color-rgb-to-hex (color-hsl-to-rgb h s v)))))
        (setq h (mod-float (+ h golden-ratio-conjugate)))
        (setq current (- current 1)))
      ret-list)))
```
      
It makes a list of CSS color codes, taking three parameters. The
algorithms I linked to earlier refers to methods using the HSV color
model, but in my code I'm using the HSL model. Why? Because Emacs
already had nice methods to convert from HSL to RGB (yeah, a
conversion to HSV is probably trivial). You can read all the details
you want at:

http://en.wikipedia.org/wiki/HSL_and_HSV

For my purpose, I just wanted to palettes using algorithms with
parameters, so whether it is this or that type of cylinders matters
less.

The important thing in this method is that it takes three obligatory
parameters and one optional. The first parameter is the number of
colors you want to generate. The second and third describe the
saturation and lightness of the palette you want generated. The final
optional parameter h describes the starting color value (the actual
color). If you do not supply one, it will use a random starting color.

Referencing the screenshot above, the first set of code shows how to
call it to generate totally random colors at a certain saturation and
lightness. You will see it generated five random palettes (one for
each line).

The second code shows non-random starting color, but shows how
saturation can be varied to obtain different colors.

And the third and final piece of code shows how lightness can be
varied.

Great, so now we can generate palettes. The final piece of code that I
use is to make 'rainbow-delimiters-mode' actually use my palette. This
is what it looks like in my .emacs file:

```lisp
(defun set-random-rainbow-colors (s l &optional h)
  ;; Output into message buffer in case you get a scheme you REALLY like.
  ;; (message "set-random-rainbow-colors %s" (list s l h))
  
  (rainbow-delimiters-mode t)

  ;; I also want css style colors in my code.
  (xah-syntax-color-hex)
  
  ;; Show mismatched braces in bright red.
  (set-face-background 'rainbow-delimiters-unmatched-face "red")

  ;; Rainbow delimiters based on golden ratio
  (let ( (colors (gen-col-list 9 s l h))
         (i 1) )
    (let ( (length (length colors)) )
      ;;(message (concat "i " (number-to-string i) " length " (number-to-string length)))
      (while (<= i length) 
        (let ( (rainbow-var-name (concat "rainbow-delimiters-depth-" (number-to-string i) "-face"))
               (col (nth i colors)) )
          ;; (message (concat rainbow-var-name " => " col))
          (set-face-foreground (intern rainbow-var-name) col))
        (setq i (+ i 1))))))

(add-hook 'js-mode-hook '(lambda () (set-random-rainbow-colors 0.5 0.49)))
(add-hook 'emacs-lisp-mode-hook '(lambda () (set-random-rainbow-colors 0.5 0.49)))
(add-hook 'lisp-mode-hook '(lambda () (set-random-rainbow-colors 0.5 0.49)))
```

The first part is the function setting the rainbow colors using a
generated palette. The second part ('add-hook' calls) shows how I
activate the colors for various programming modes.

Here's a final example from my '.emacs' file, showing more colored
parenthesis and colorized CSS color codes. Needless to say, it works
for all languages supported by Emacs, not just Lisp..

![](dotemacs.png?raw=true)
