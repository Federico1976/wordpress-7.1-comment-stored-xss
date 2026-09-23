# WordPress 7.1 — Unauthenticated Stored XSS via Comment Rendering

Studio tecnico e Proof of Concept di una vulnerabilità **Stored Cross-Site Scripting (XSS) non autenticata** presente nel rendering dei commenti di **WordPress 7.1**.

La vulnerabilità è stata corretta in **WordPress 7.1.1**.

## Stato

- Versione vulnerabile verificata: **WordPress 7.1**
- Versione corretta verificata: **WordPress 7.1.1**
- Tema utilizzato nella riproduzione: **Twenty Twenty-Five 1.5**
- Attaccante: **non autenticato**
- Vittima: **Administrator**
- HackerOne report: **#4050384**
- Stato HackerOne: **Duplicate**
- Report originale associato da WordPress/HackerOne: **#4009053**

## Sintesi

Un visitatore non autenticato può inviare un commento contenente markup consentito dal normale percorso di sanitizzazione di WordPress.

Il problema nasce successivamente durante il rendering del contenuto.

Una particolare combinazione di sanitizzazione, `wpautop()`, trasformazioni successive e parsing finale del browser può modificare la struttura dell'HTML già sanitizzato e far reinterpretare testo controllato dall'attaccante come un vero attributo evento HTML.

Nel test originale Chromium materializzava un attributo `onmouseover` sul `<blockquote>`, permettendo l'esecuzione di JavaScript nell'origine e nella sessione dell'utente autenticato che visualizzava il commento.

## Root cause

La vulnerabilità non consiste in un semplice bypass diretto di KSES.

Il commento attraversa il normale percorso di sanitizzazione di WordPress e viene memorizzato. Il comportamento pericoloso emerge successivamente, durante le trasformazioni applicate al contenuto prima del rendering finale.

In WordPress 7.1, `wpautop()` utilizzava una gestione del tag `<blockquote>` che non trattava in modo atomico i valori degli attributi quotati.

Una newline inserita all'interno di un attributo `cite` consentito poteva provocare l'inserimento di un elemento `<p>` all'interno del valore dell'attributo.

Le trasformazioni successive potevano alterare ulteriormente la rappresentazione HTML e creare una differenza tra:

- contenuto inizialmente sanitizzato;
- HTML successivamente trasformato;
- DOM finale interpretato dal browser.

Un secondo elemento HTML consentito poteva quindi contribuire alla chiusura del contesto dell'attributo.

Il risultato finale osservato in Chromium era la materializzazione di un vero attributo evento `onmouseover` sul `<blockquote>`.

## Catena causale verificata

La catena riprodotta nel laboratorio è stata:

    Attaccante non autenticato
              |
              v
    POST /wp-comments-post.php
              |
              v
    Commento memorizzato
              |
              v
    Approvazione del commento
              |
              v
    Rendering del commento
              |
              v
           wpautop()
              |
              v
    Rottura del contesto dell'attributo
              |
              v
    Chromium materializza onmouseover
              |
              v
    JavaScript nella sessione Administrator

La Proof of Concept minima ha confermato l'esecuzione JavaScript impostando un marker nel contesto della pagina.

Durante la verifica dell'impatto è stata inoltre dimostrata una catena completa nella quale la XSS, eseguita nella sessione di un Administrator:

1. richiedeva `/wp-admin/user-new.php`;
2. recuperava automaticamente il nonce `_wpnonce_create-user`;
3. inviava la richiesta privilegiata di creazione utente;
4. creava un nuovo account WordPress con ruolo `administrator`.

Questa seconda parte dimostra l'impatto della Stored XSS; la vulnerabilità primaria documentata in questo repository rimane la Stored XSS nel rendering dei commenti.

## Condizioni della riproduzione

La riproduzione originale è stata effettuata con:

- WordPress 7.1
- Twenty Twenty-Five 1.5
- commenti abilitati
- attaccante non autenticato
- vittima autenticata come Administrator

Il commento veniva inviato attraverso il normale endpoint:

    POST /wp-comments-post.php

Non era necessario alcun account WordPress per l'attaccante.

## Correzione in WordPress 7.1.1

La stessa Proof of Concept è stata successivamente ritestata su WordPress 7.1.1.

La catena originale non è più riproducibile.

Il comportamento corretto impedisce alla sequenza utilizzata nella PoC di rompere il contesto dell'attributo e Chromium non materializza più l'attributo evento controllato dall'attaccante.

WordPress/HackerOne ha confermato che il report #4050384 corrispondeva allo stesso problema sottostante già segnalato nel report #4009053 e che la correzione distribuita in WordPress 7.1.1 impedisce il malformed attribute e l'esecuzione dello script.

## Responsible Disclosure

La vulnerabilità è stata segnalata attraverso il programma WordPress su HackerOne.

- Report: `#4050384`
- Stato: `Duplicate`
- Duplicate di: `#4009053`
- Versione corretta: `WordPress 7.1.1`

Questo repository documenta una vulnerabilità già corretta ed è pubblicato esclusivamente per finalità di ricerca, studio tecnico e responsible disclosure.

## Autore

**Federico Brasili**

GitHub: [Federico1976](https://github.com/Federico1976)
