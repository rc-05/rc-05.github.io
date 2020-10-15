# Introduzione ad ARC ed ORC in Nim

## Introduzione

Cominciamo dagli inizi - Nim è stato tradizionalmente un linguaggio con gestione
automatica della memoria.
Buona parte della libreria standard di Nim fa leva sul Garbage Collector. 
Ovviamente puoi disattivare il GC e gestire manualmente la memoria ma così
facendo perdi l'accesso a gran parte della stdlib
(la quale è [molto grande](https://nim-lang.org/docs/lib.html])).

Il GC di default in Nim è stato `refc` (contatore di riferimenti con una fase
di raccolta e pulizia con collezione ciclica) per molto tempo,
con altre opzioni disponibili ovvero `markAndSweep`, `boehm`, `go`.

Però negli ultimi anni sono venute alla luce nuove idee riguardanti
distruttori, riferimenti "con proprietario" (newruntime) e simili per Nim:

* https://nim-lang.org/araq/destructors.html
* https://nim-lang.org/araq/ownedrefs.html

Alcune di queste idee sono state integrate in ARC.

## Cos'è ARC?

ARC è un modello di gestione della memoria basato sul conteggio automatico
di riferimenti (alcune persone scambiano l'ARC di Nim con quello di Swift;
però c'è una grande differenza -
ARC in Nim non usa un contatore atomico come in Swift) con distruttori e
semantiche di spostamento della memoria.

Il conteggio di riferimenti è uno degli algoritmi di gestione della memoria
più famosi per rilasciare le risorse inutilizzate in un programma.
**Il conteggio di riferimenti** per ogni riferimento tracciato
(controllato dal runtime) indica quante volte uno specifico riferimento viene
utilizzato in altri punti.
Quando quel contatore arriva a zero il riferimento e tutte le risorse associate
ad esso vengono rilasciati.

La differenza principale tra ARC ed i GC di Nim è che ARC è completamente
**deterministico** - il compilatore *initetta* automaticamente quando determina
che una variabile (stringa, sequenza, riferimento o quant'altro)
non è più necessaria: praticamente è molto simile al RAII del C++ con i suoi
distruttori.
Per illustrare quanto sto dicendo possiamo usare `expandArc`
(sarà disponibile con Nim 1.4).

Proviamo con del semplice codice:
```Nim
proc main = 
  let mystr = stdin.readLine()

  case mystr
  of "hello":
    echo "Nice to meet you too!"
  of "bye":
    echo "Goodbye!"
    quit()
  else:
    discard

main()
```

E poi usiamo lo strumento `expandArc` su `main` eseguendo
`nim c --gc:arc --expandArc:main example.nim`
```Nim
var mystr
try:
  mystr = readLine(stdin)
  case mystr
  of "hello":
    echo ["Nice to meet you too!"]
  of "bye":
    echo ["Goodbye!"]
    quit(0)
  else:
    discard
finally:
  `=destroy`(mystr)
```
Quello che vediamo qui è molto interessante - il compilatore Nim ha "rinchiuso"
il corpo della funzione `main` in un blocco `try: finally`
(il codice nella parte `finally` viene sempre eseguito, anche quando viene
sollevata un'eccezione) ed ha inserito una chiamata a `=destroy` per `mystr`
(inizializzata in fase di esecuzione)
in modo da distruggerla quando non più necessaria
(cioè quando il suo lifetime finisce).

Questo mette in mostra una delle caratteristiche principali di ARC:
**gestione della memoria basato su "blocchi"**. Un "blocco" è una regione
separata di codice nel nostro programma.
Una gestione della memoria basata sui "blocchi" implica che il compilatore
inserirà automaticamente le chiamate al distruttore per ogni variabile che
necessita di esso quando si esce da un "blocco" di codice.
Molti construtti in Nim introducono un nuovo "blocco": `proc`, `func`,
convertitori, metodi, `block`, cicle `for` e `while`, ecc.

ARC possiede anche il concetto di **"agganci"** - procedure speciali che
possono essere definite per uno specifico tipo in modo da cambiare il
comportamento di default
quando la variabile viene distrutta/spostata/cambiata. Questo ritorna
particolarmente utile nel caso si voglia implementare dei costrutti
personalizzati per i propri tipi oppure per
occuparsi di operazioni di basso livello come utilizzare i puntatori od
anche per una FFI.

I vantaggi di ARC rispetto al GC `refc` predefinito sono
(inclusi quelli che ho appena specificato):
* **Gestione della memoria basata su blocchi** (i distruttori vengono
inseriti dopo il blocco di codice) - generalmente riduce l'utilizzo della RAM
dei programmi e aumenta le prestazioni.
* **Semantiche di movimento** - l'abilità del compilatore di analizzare
staticamente il programma
e convertire le copie di blocchi di memoria in spostamenti ove possibile.
* **Heap condiviso** - i diversi thread hanno accesso alla stessa sezione 
di memoria quindi non hai bisogno di copiare le variabili per passarle da un
thread ad un altro, puoi semplicemente effettuare uno spostamento;
vedi questo [RFC](https://github.com/nim-lang/RFCs/issues/244) riguardante
l'isolamento e l'invio di dati da un thread ad un altro.
* Adatto per [**Sistemi Real-Time**](https://it.wikipedia.org/wiki/Sistema_real-time).
* FFI più semplice da usare (`refc` richiede di settare il GC manualmente per
ogni thread esterno all'applicazione, cosa non richiesta con ARC).
Inoltre questo significa che ARC è una scelta migliore per scrivere delle
librerie in Nim che permettono di interfacciarsi con altri linguaggi
(.dll, .so, [Estensioni Python](https://github.com/yglukhov/nimpy), ecc.).
* Eliminazione di copie non necessarie - questa ottimizzazione, molto spesso, 
riduce delle copie in semplici alias.

Generalmente, ARC è un grosso passo avanti che rende i propri programmi più
performanti, permette un consumo più basso di memoria e sopratutto garantisce
un comportamento deterministico.

Per abilitare ARC nel tuo programma, devi semplicemente compilare con lo switch
`--gc:arc`,
oppure puoi aggiungerlo al file di configurazione del tuo progetto
(`.nims` o `.cfg`).

## Problema con le dipendenze cicliche

Ma aspetta! Non abbiamo dimenticato qualcosa? ARC è basato sul conteggio
di rifermienti - e come noi sappiamo - quest'ultimo non si occupa delle
dipendenze cicliche.
In poche parole, una dipendenza ciclica si verifica quando delle variabili
fanno riferimeno una all'altra, in modo da ricordare un ciclo perpetuo.
Facciamo un semplice esempio: abbiamo tre oggetti (A, B, C) e ciascuno di essi
fa riferimento ad un altro oggetto, un'illustrazione ci aiuterà a capire meglio:

![cicli](https://nim-lang.org/assets/news/images/yardanico-arc/cycle.svg)

Per trovare la dipendenza ciclica abbiamo bisogno di un collettore di cicli - 
una parte speciale del runtime che identifica e rimuove le dipendenze cicliche
non più necessarie nel nostro programma.

In Nim la collezione di cicli è stata effettuata nella fase di "marcamento e
collezione" del GC `refc`, ma è meglio utilizzare ARC come fondamenta di un
meccanismo migliore. Questo ci porta a:

## ORC - Il collettore di dipendenze cicliche di Nim

ORC è il nuovo collettore di dipendenze cicliche basato su ARC. Può essere
considerato un GC completo poiché include una fase di tracciamento locale
(al contrario degli altri GC che utilizzano un metodo di tracciamento globale).
ORC è ciò che dovresti usare se stai lavorando con `async` di Nim perché
contiene delle dipendenze cicliche che devono essere risolte.

ORC mantiene molti dei vantaggi di ARC eccetto **determinismo** e
**sistema real-time** (parzialmente) -
di default ORC imposta un threshold adattivo per collezionare e risolvere
le dipendenze cicliche che possono essere generate.

Per abilitare ORC nel tuo programma devi compilare usando lo switch `--gc:orc`
ma molto probabilmente ORC diventerà in futuro il GC predefinito di Nim.

## Non sto più nella pelle! Come posso provarli?

ARC è disponibile dai rilasci 1.2.X di Nim, ma per via di bug risolti è meglio
aspettare la versone 1.4 (dovrebbe uscire a momenti) la quale renderà
disponibili ARC ed ORC per testarli. Se però non riesci ad aspettare, esiste
un [pre-rilascio della versione 1.4](https://github.com/nim-lang/nightlies/
releases/tag/2020-10-07-version-1-4-3b901d1e361f49d48fb64d115e42c04a4a37100c).

Questo è tutto! Grazie per aver letto l'articolo - spero che sia stato di tuo
gradimento e che possa apprezzare le possibilità che ARC ed ORC forniscono
a Nim :)

### Fonti / Ulteriori informazioni

* [Introducing –gc:arc](https://forum.nim-lang.org/t/5734)
* [Update on –gc:arc](https://forum.nim-lang.org/t/6549)
* [New garbage collector –gc:orc is a joy to use.](https://forum.nim-lang.org/t/6483)
* [Nim destructors and move semantics](https://nim-lang.org/docs/destructors.html)
* [FOSDEM 2020 - Move semantics for Nim](https://www.youtube.com/watch?v=yA32Wxl59wo)
* [NimConf 2020 - Nim ARC/ORC](https://www.youtube.com/watch?v=aUJcYTnPWCg)
* [Nim community](https://nim-lang.org/community.html)
* [RFC: Unify Nim’s GC/memory management options](https://github.com/nim-lang/RFCs/issues/177)

---
*Scritto il Giovedì 15 Ottobre 2020*  
*L'articolo originale di Danil Yarantsev lo puoi trovare [qui](https://nim-lang.org/blog/2020/10/15/introduction-to-arc-orc-in-nim.html).*

[🏠 Ritorna alla pagina iniziale](https://rc-05.github.io/index-it)
