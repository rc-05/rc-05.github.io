Recently I've been playing around with ChezScheme, a very efficent Scheme R6RS
compiler with an [interesting story](https://legacy.cs.indiana.edu/~dyb/pubs/hocs.pdf).

One thing that bugged me tho is that ChezScheme doesn't include a documentation
generation binary or library: not even the Scheme standards specify a standard
way of documenting your code...

In Common Lisp you can simply embed documentation directly into the functions
itself, like this:

```common-lisp
(defun foobar ()
  "A stupid function that does nothing useful."
  (+ 2 2))
```

This is why I've created [**docgen**](https://github.com/rc-05/docgen),
a library for generating interactive documentation with ChezScheme.

I hope that it can be useful not only for me but for somebody else! ☺️

---
*Written on Friday 20th March 2020*

[⬅ Previous Article](https://rc-05.github.io/articles/hello-world)
[🏠 Return to the Home Page](https://rc-05.github.io)
