# Analisi tecnica della root cause

## Panoramica

La vulnerabilità interessa la pipeline di rendering dei commenti di WordPress 7.1.

Il punto fondamentale è che il payload non introduce direttamente un attributo evento vietato attraverso KSES.

Il contenuto viene inizialmente accettato e memorizzato, ma una trasformazione applicata successivamente da `wpautop()` modifica la struttura dell'HTML già sanitizzato.

La vulnerabilità nasce quindi da una discrepanza tra:

- rappresentazione validata;
- rappresentazione trasformata;
- rappresentazione interpretata dal browser.

## Codice rilevante in WordPress 7.1

La logica vulnerabile di `wpautop()` includeva una trasformazione del `<blockquote>` equivalente a:

    $text = preg_replace(
        '|<p><blockquote([^>]*)>|i',
        '<blockquote$1><p>',
        $text
    );

Il problema è il frammento:

    ([^>]*)

Questa espressione considera `>` come terminatore senza distinguere correttamente se il carattere si trovi:

- realmente alla fine del tag;
- oppure all'interno di un valore di attributo quotato.

In altre parole, il parser basato sulla regular expression non tratta il valore dell'attributo come una unità atomica.

## Input controllato dall'attaccante

Un commento poteva contenere markup consentito come:

    <blockquote cite="...">...</blockquote>

L'attributo `cite` è consentito.

Inserendo una newline all'interno del valore dell'attributo, il contenuto poteva attraversare il normale percorso di sanitizzazione e rimanere memorizzato.

## Trasformazione successiva

Durante il rendering, `wpautop()` elaborava nuovamente il contenuto.

La struttura poteva diventare concettualmente:

    <blockquote cite="VALORE
    <p>
    RESTO_DEL_VALORE">

A questo punto il markup risultante non corrispondeva più alla struttura che era stata originariamente validata.

## Secondo elemento consentito

Un secondo elemento HTML consentito, ad esempio un `<a>` con attributo `title`, poteva fornire testo controllato dall'attaccante in una posizione utile al successivo parsing HTML.

Il punto importante è che anche questo elemento poteva essere valido singolarmente.

La vulnerabilità emergeva dalla combinazione delle trasformazioni successive.

## Browser parsing

Il browser riceveva quindi HTML strutturalmente diverso da quello originariamente sanitizzato.

Nel test effettuato con Chromium, il valore controllato dall'attaccante veniva reinterpretato come un vero attributo evento sul `<blockquote>`.

Il DOM risultante conteneva un attributo:

    onmouseover="..."

che non era presente come attributo evento valido nel contenuto originariamente accettato da WordPress.

## Confine di sicurezza violato

La pipeline può essere rappresentata così:

    input attacker
          |
          v
       KSES
          |
          |  contenuto considerato sicuro
          v
       storage
          |
          v
       wpautop()
          |
          |  trasformazione strutturale
          v
    HTML differente
          |
          v
    browser parser
          |
          v
    event handler reale

La proprietà di sicurezza assunta era:

    contenuto sanitizzato == contenuto sicuro al rendering

La vulnerabilità dimostra invece:

    contenuto sanitizzato
        + trasformazione successiva
        != contenuto necessariamente sicuro

## Perché è una Stored XSS

L'input dell'attaccante viene memorizzato nel database.

L'esecuzione avviene successivamente quando un altro utente visualizza la pagina contenente il commento.

Nel caso verificato, il JavaScript veniva eseguito nell'origine WordPress e nella sessione autenticata dell'Administrator.

Si tratta quindi di una Stored Cross-Site Scripting vulnerability.

## Correzione

WordPress 7.1.1 modifica la gestione del parsing affinché i valori degli attributi quotati vengano trattati correttamente.

La sequenza usata nella PoC originale non riesce più a rompere il contesto dell'attributo.

Nel retest su WordPress 7.1.1:

- il `<blockquote>` rimane strutturalmente sicuro;
- non viene materializzato `onmouseover`;
- il JavaScript non viene eseguito.

## Lezione generale

Questa vulnerabilità mostra un pattern importante nelle pipeline HTML:

    sanitize -> transform -> render

Sanitizzare un contenuto prima di una trasformazione strutturale non garantisce necessariamente che il risultato finale rimanga sicuro.

Quando una trasformazione successiva può modificare:

- quote;
- delimitatori;
- attributi;
- tag;
- contesti HTML;

il risultato dovrebbe essere trattato come una nuova rappresentazione che richiede nuovamente garanzie di sicurezza.
