# Docgen

Recentemente stavo giocherellando con ChezScheme, un'implementazione efficente
per Scheme R6RS con un'[interessante storia](https://legacy.cs.indiana.edu/~dyb/pubs/hocs.pdf).

Una cosa che mi ha infastidito è che ChezScheme non include un binario od una libreria per generare la documentazione: neanche gli standard
Scheme specificano un metodo standard per documentare il proprio codice..

In Common Lisp puoi semplicemente includere la documentazione direttamente nelle funzioni,
in questo modo:

```common-lisp
(defun foobar ()
  "Una stupida funzione che non fa nulla di utile."
  (+ 2 2))
```

Ecco perché ho creato [**docgen**](https://github.com/rc-05/docgen),
una libreria per generare documentazione interattiva con ChezScheme.

Spero che questo possa essere utile non solo a me, ma anche a qualcun'altro! ☺️

---
*Scritto il Venerdì 20 Marzo 2020*

[🏠 Ritorna alla pagina iniziale](https://rc-05.github.io/index-it)
