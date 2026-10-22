---
author: ["Simeone Vilardo"]
title: "Come NON implementare un promo code"
date: "2024-03-17"
description: "Come un sistema di promo code può essere facilmente reverse-engineered e sfruttato"
summary: "Un promo code è un codice utilizzabile per partecipare ad una certa promozione. Facciamo un esempio molto pratico. Immaginiamo di essere una catena che venda prodotti, vogliamo permettere ai nostri clienti di partecipare ad un sondaggio sulla qualità del nostro servizio. In cambio, potranno ricevere un premio (ad esempio uno sconto o un prodotto gratuito). Vogliamo però che questo sondaggio possa essere compilato solo da chi ha effettuato realmente un acquisto. Per fare ciò, possiamo stampare un codice univoco sullo scontrino, che il cliente potrà utilizzare per partecipare al sondaggio. Questo codice è il promo code."
tags: ["hacking", "programming"]
series: ["Reverse Engineering"]
---

## Introduzione
Cosa si intende, in questo articolo, per promo code?
Un promo code è un codice utilizzabile per partecipare ad una certa promozione.
Facciamo un esempio molto pratico. Immaginiamo di essere una catena che venda prodotti, vogliamo permettere ai nostri clienti di partecipare ad un sondaggio sulla qualità del nostro servizio. In cambio, potranno ricevere un premio (ad esempio uno sconto o un prodotto gratuito). Vogliamo però che questo sondaggio possa essere compilato solo da chi ha effettuato realmente un acquisto. Per fare ciò, possiamo stampare un codice univoco sullo scontrino, che il cliente potrà utilizzare per partecipare al sondaggio. Questo codice è il promo code.

## Attenti al front-end
Ho trovato personalmente un esempio estremamente simile a questo. Si tratta di diversi anni fa (per cui non posso garantire che sia ancora così) e non dirò di quale società si tratti. La società in questione permetteva la compilazione di una survey attraverso un sito web.
Il promo code era una stringa alfanumerica di 13 caratteri, questo è un esempio: **`OCFBMEGDFXEEE`**
C'è però un grosso problema: la logica di verifica del promo code era inserita all'interno del front-end. Chiunque ispezionasse il sorgente della pagina poteva facilmente risalire ad una funzione che decriptava il promo code in alcuni dati fondamentali e ne verificava la validità tramite un checksum.
Questo leak è molto grave, perché permette, con estrema facilità, di invertire la funzione di decriptazione e generare promo code validi a piacimento. È sufficiente "selezionare" i valori che sono encodati all'interno di un promo code e calcolare il checksum.
Io capisco la bellezza di avere un promo code che sia "auto-contenuto" e che quindi possa ritornare le informazioni del suo scontrino senza bisogno di alcuna sorgente esterna (come un database), ma fare ciò richiede molta attenzione e competenza. Inoltre, è sempre meglio effettuare queste operazioni sul back-end, anche nei casi in cui il database non sia richiesto.

## Implementazione
### Funzione di decrypt
Di seguito, una versione in pseudocodice dell'algoritmo di decodifica.
```js
function decryptPromoCode(code){
	var codeWithoutChecksum = removeChecksumFromCode(code);
	var decimalCode = convertBase28ToDecimal(codeWithoutChecksum);
	decimalCode = decimalCode.substring(1);
	decimalCode = reverseString(decimalCode);
	var siteId = Number(decimalCode.substring(0, 4));
	var posId = Number(decimalCode.substring(4, 6));
	var days = Number(decimalCode.substring(6, 10))
	var transId = Number(decimalCode.substring(10));
	return {days: days, siteId: siteId, posId: posId, transId: transId};
}
```
La funzione risulta molto semplice e mostra come il promo code sia solamente la concatenazione di:
- `siteId` (Numero identificativo del negozio)
- `posId` (Numero identificativo del terminale che effettua la vendita)
- `transId` (Numero identificativo della transazione)
- `days` (Numero di giorni trascorsi dal 31-12-2014)
Le componenti del promo code,  `siteId`, `posId`, `transId` e `days` sono tutte stringhe paddate con 0 a sinistra. Per fare un esempio, se il `siteId` fosse il numero `42`, all'interno del promo code decimale sarebbe espresso come `0042`.

La decodifica da base 28 a base 10 è effettuata dalla funzione `convertBase28ToDecimal`. È una normale funzione di cambiamento di base, i caratteri che formano la base 28 sono: `ABCDEFGHIJKLMNOPRSTUWXZ45679`.

Il checksum non è un elemento rilevante ai fini della nostra analisi. Per completezza, vi dico che è formato da 3 caratteri che sono inseriti all'interno del promo code, rispettivamente alle posizioni `1`, `7` e `11`. In fase di decodifica il checksum è semplicemente ignorato. Questo ha senso, perché può essere verificato in un altro momento (prima o dopo la decodifica del payload).

La cosa forse più interessante di questa funzione però è la riga:
```js
decimalCode = decimalCode.substring(1);
```
Per quale motivo, in fase di decodifica, si rimuove il primo carattere? È inutile? Da dove viene?
Lavoriamo per approssimazioni successive e usiamo i dati empirici per andare avanti. Decodificando diversi codici promo trovati sugli scontrini, posso verificare che questo carattere scartato è sempre `1`. Teniamo quindi bene in mente questa stranezza e proviamo a scrivere una funzione di codifica.


### Funzione di codifica
Eseguendo gli step del codice precedente a ritroso, otteniamo qualcosa del genere:
```js
function generatePromoCode(days, siteId, posId, transId){
	var decimalCode = siteId + posId + days + transId + "1";
	var codeNoChecksum = convertToBase28(reverse(decimalCode));
	var checksum = generateChecksum(codeNoChecksum);
	var code = `${codeNoChecksum.charAt(0)}${checksum[2]}${codeNoChecksum.substring(1, 6)}${checksum[1]}${codeNoChecksum.substring(6, 9)}${checksum[0]}${codeNoChecksum.charAt(9)}`;
	return code;
}
```
Notiamo che, come discusso poco fa, concateno `1` agli altri elementi. Il resto è ovvio e poco interessante.

## Quanto è grave?
Adesso che possediamo entrambe le funzioni, possiamo metterle alla prova.
Se provo a decodificare l'esempio che avevo fornito in precedenza, cioè **`OCFBMEGDFXEEE`**, ottengo il seguente risultato:
```js
{
	days: 1476,
	siteId: 37,
	posId: 6,
	transId: 5
}
```
Se codifico queste informazioni utilizzando la funzione inversa, ottengo proprio **`OCFBMEGDFXEEE`**.
Quanto è grave questo? Abbastanza. Chiunque è potenzialmente in grado di prevedere e generare qualsiasi promo code. Certo, bisogna conoscere `siteId`, `posId` e `transId`, potreste obiettare. Ma come è facile immaginare e come ho verificato sperimentalmente, `posId` e `transId` sono semplicemente numeri incrementali che partono da 1. Ogni punto vendita ha un certo numero di casse che equivalgono ai `posId` e partono dal numero `1`.
Allo stesso modo, `transId`, il numero della transazione, è un incrementale univoco per un certo pos in una certa data. Il primo scontrino del pos numero N sarà `1` e così via.
L'unico dato più "complesso" da trovare potrebbe essere l'id del punto vendita. Certo, parliamo di un numero a 4 cifre e di una catena che ha punti vendita ovunque nella mia nazione, quindi quasi qualsiasi numero potrebbe risultare valido. Ma se volessimo creare un promo code di uno specifico punto vendita e non semplicemente di uno a caso? Si dà il caso che l'app della società abbia (o avesse) una sezione "trova i punti vendita" dove c'era una lista completa di ogni store con relativo indirizzo. E, pensate un po', guardando il JSON di tale lista si scopre che anche se non viene mostrato all'utente, lì dentro è presente l'ID del punto vendita relativo a ogni indirizzo.

## Può essere ancora più grave? Sì
Arrivo alla parte più succosa della nostra analisi. A breve vi spiegherò il mistero dell'`1` che viene rimosso in fase di decodifica.
Per fare ciò però prima immaginiamo uno use case per questo sistema di promo code (vi assicuro che questo use case è andato in produzione per un lungo periodo dove vivo io).
> Ogni utente che acquista l'oggetto A può ricevere 1 punto inserendo il promo code relativo a quell'acquisto nella nostra web app. Si possono effettuare al massimo 5 inserimenti al giorno. Quando si ottengono 10 punti, si può riscattare gratuitamente l'oggetto B.
> NOTA: Se viene effettuato l'inserimento di un promo code il cui scontrino non comprende l'acquisto dell'oggetto A, tale inserimento viene comunque conteggiato per il limite dei 5 inserimenti giornalieri.

Un attaccante malevolo potrebbe generare promo code validi, sperando che alcuni di quelli siano collegati ad uno scontrino che contiene effettivamente l'oggetto A tra gli acquisti. Non esiste però nessun modo per sapere, prima dell'inserimento all'interno del sistema, se questo è vero. Inoltre, il limite dei 5 inserimenti giornalieri rende estremamente svantaggioso tentare un bruteforce. Nella maggior parte dei casi, anche creando più account, ogni account sarebbe subito bloccato per quel giorno consumando tutti e 5 gli inserimenti a vuoto.

Se solo fosse possibile creare delle "collisioni", cioè generare due o più promo code i cui caratteri alfanumerici siano diversi ma le cui informazioni codificate all'interno siano le stesse, in questo caso si potrebbe fare un bruteforce! Sarebbe sufficiente provare un gran numero di codici promo utilizzando un gran numero di account. I promo code contenenti l'articolo A darebbero 1 punto a quell'account, che sarebbe comunque bruciato (perché non sarebbe possibile caricare altri punti per tutto il giorno). Si otterrebbe però una lista di promo code (bruciati, perché già inseriti nel sistema) che comprendono effettivamente l'oggetto A. Creando delle collisioni di tali codici si potrebbero inserire queste collisioni in un nuovo account, che potrebbe riscattare 5 punti in un solo giorno e ottenere l'oggetto B in omaggio in due giorni nel 100% dei casi (non ci sarebbero mai casi di promo code da zero punti).
Ma come si possono generare delle collisioni? Ormai lo avrete intuito, è grazie al nostro `1`.

## Collisioni: come e perché
Vediamo che succede durante la generazione di un codice promo con le seguenti informazioni da codificare:
```js
{
	days: 666,
	siteId: 90,
	posId: 69,
	transId: 420
}
```
Se noi concatenassimo queste informazioni (con relativi padding di 0) otterremmo la stringa `00906906660420`.
Invertendo questa stringa, si ottiene `02406660960900`.
Questo numero convertito in base 28 diventa `GKKF49BLA`.
Qui arriva il bello, se convertito in base 10, questo numero diventa `2406660960900`.
Abbiamo appena perso uno `0`. Perdere uno `0` ovviamente rovina tutto l'algoritmo di decodifica, che si basa sul fatto che ogni variabile abbia una lunghezza fissa di caratteri.
Ecco perché viene aggiunto l'`1` in fase di codifica! È fatto per gestire il caso in cui l'ID di una transazione termini con uno `0`, valore che andrebbe perso in fase di conversione in int, perché ovviamente gli zeri a sinistra non sono cifre significative.
Ma questo significa... che abbiamo appena scoperto un modo per provocare collisioni: niente obbliga quel primo carattere a essere `1`.
Possiamo infatti modificare la funzione di codifica nel seguente modo:
```js
function generatePromoCode(days, siteId, posId, transId, collisionChar){
	var decimalCode = siteId + posId + days + transId + collisionChar;
	var codeNoChecksum = convertToBase28(reverse(decimalCode));
	var checksum = generateChecksum(codeNoChecksum);
	var code = `${codeNoChecksum.charAt(0)}${checksum[2]}${codeNoChecksum.substring(1, 6)}${checksum[1]}${codeNoChecksum.substring(6, 9)}${checksum[0]}${codeNoChecksum.charAt(9)}`;
	return code;
}
```
E invocare questa funzione due volte:
```js
var promoCode1 = generatePromoCode(1476, 37, 6, 5, "1");
var promoCode2 = generatePromoCode(1476, 37, 6, 5, "2");
console.log(promoCode1); // "OCFBMEGDFXEEE"
console.log(promoCode2); // "4MSWWHGTHXIPW"
```
I due promo code generati, cioè **`OCFBMEGDFXEEE`** e **`4MSWWHGTHXIPW`**, sono entrambi validi, vengono correttamente decodificati dalla funzione di decodifica ufficiale e contengono le stesse informazioni!
E con lo `0`? Non funziona, per lo stesso motivo visto poco fa: lo `0` iniziale si perde nella conversione, quindi la funzione di decodifica rimuove l'ultima cifra dell'ID transazione invece del nostro carattere di padding. Il codice generato con lo `0`, **`EHWKEBGSDW9HR`**, ha un checksum valido, ma viene decodificato con `transId` uguale a `0` invece che `5`.
È possibile proseguire e generare ulteriori collisioni fino all'utilizzo del char `9`? Purtroppo (o per fortuna) non è possibile. Ogni tentativo di generare promo code con `collisionChar` maggiori di 2 provocherà la corruzione dei dati codificati. Per quale motivo? Semplice, perché il `collisionChar` finirà per forza di cose al primo posto a sinistra del numero decimale. Dato che tale numero decimale andrà convertito in base 28 e dovrà avere come risultato una stringa lunga 10 caratteri, deve essere necessariamente uguale o inferiore a `296196766695423` (28^10 - 1).
Avendo il 3 come primo valore numerico, si ottiene per forza di cose un numero che convertito in base 28 è lungo 11 caratteri.
Per lo stesso motivo, nemmeno il `2` funziona in tutti i casi: se l'ID transazione termina con `79`, `89` o `99` (e in alcuni casi con `69`), il numero supera quel limite.

## Qual è la morale?
Quando si creano questi sistemi, bisogna assicurarsi che i codici generati:
- NON siano prevedibili e riproducibili
- NON sia possibile generare collisioni

Attenzione però: il vero problema non è solo il front-end. Il checksum non usa nessuna chiave segreta, quindi spostare lo stesso algoritmo sul back-end renderebbe il reverse engineering più difficile, non impossibile. Quanto alle collisioni, sarebbe bastato rifiutare i codici il cui carattere di padding non è `1` o, meglio ancora, tenere traccia dello scontrino decodificato (punto vendita, pos, giorno, transazione) invece che della stringa del codice.

La mia opinione è che sarebbe bello poter generare dei codici auto-contenuti, ed è possibile. I token JWT sono un ottimo esempio, grazie al loro meccanismo di firma digitale ci si può fidare del loro contenuto e della loro provenienza, il problema è che bisogna inserire la firma all'interno del token. Se tale token deve essere utilizzato manualmente dall'utente, deve essere il più corto e facile possibile, quindi inserire una firma completa nel token non è una strada percorribile. Ma una firma completa non serve: bastano pochi caratteri di un MAC (ad esempio un HMAC-SHA256 troncato), calcolato con una chiave segreta che conosce solo il back-end, al posto del checksum. Probabilmente è anche per questo che la società ha scelto codici auto-contenuti: la cassa può stamparli senza contattare nessun server.
Altrimenti, non andiamo a risparmio. Usiamo le tabelle e usiamo le chiamate API. Una semplice stringa alfanumerica casuale di 8 caratteri permette `2821109907456` combinazioni. Sono quasi tre trilioni. Ed essendo casuale, buona fortuna a riprodurre un codice o generare una collisione (basta ricordarsi un vincolo di unicità sulla tabella: anche i codici casuali possono collidere per caso).