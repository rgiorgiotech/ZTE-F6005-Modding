# ZTE F6005 GPON ONT Modding

### Introduzione

Questa repository documenta la **mia ricerca indipendente** e i progressi nel modding dell’ONT GPON ZTE F6005. L’obiettivo è sbloccare funzionalità avanzate, personalizzare le impostazioni ed esplorare il firmware e l’hardware di questo tipo di dispositivi GPON.

Per il mio studio è stato fondamentale confrontare tre ONT differenti:
- Nokia G-010G-T
- ZTE F6005 (versione distribuita da Open Fiber)
- ZTE F6005 (versione distribuita da TIM)

Questi tre dispositivi condividono lo stesso hardware e derivano da un “padre” comune, il **CIG G-97CP**; confrontando le board si nota una quasi completa analogia, a meno delle SPI NOR Flash che tuttavia rispettano le medesime specifiche e quindi le differenze di produttore non comportano incompatibilità scambiando i firmware.
- ZTE F6005 TIM: Macronix MX25L12833F
- ZTE F6005 Open Fiber: XMC XM25QH128C
- Nokia G-010G-T: Winbond W25Q128JV

*(Nota: queste sono le SPI NOR Flash installate nei miei ONT, ma potrebbero variare tra diversi esemplari)*

Aprendo il cage di metallo che protegge la parte ottica, però, emerge una differenza cruciale: il Nokia utilizza un laser driver diverso rispetto a quello dei due ZTE. Questa discrepanza è confermata dal fatto che, facendo un dump completo della SPI flash del Nokia e flashandolo sullo ZTE, tutto funziona – il kernel viene caricato e l’OS funziona senza problemi – ma la parte ottica non opera correttamente (LED “LOS” acceso fisso e nessuna connessione ottica).
- Laser driver del Nokia G-010G-T: SEMTECH 25L95
- Laser driver dello ZTE F6005: 02099G-15

Ogni kernel analizzato carica esclusivamente i driver (kmodules) per cui è stato progettato, rendendo incompatibili le parti ottiche tra dispositivi con laser driver differenti. Poiché il firmware del Nokia ha già tutto sbloccato (Telnet e SSH a livello OS, porta seriale sia nel bootloader sia nel kernel), l’idea iniziale era quella di provare a flashare un dump completo del Nokia sullo ZTE. Tuttavia, come già scritto, a causa del laser driver diverso, lo ZTE non rispondeva al segnale ottico.

Il passaggio successivo è stato quello di analizzare attentamente i dump e applicare un ragionamento in stile “reverse engineering” per provare a capire il funzionamento specifico di ciascun firmware.

Ho quindi proceduto ad utilizzare diversi tool da riga di comando, tra cui:
- *binwalk* e *firmware-mod-kit*, per analizzare il contenuto di dump e firmware estratti;
- *flashrom*, per flashare le SPI NOR Flash;
- *cramfs-tools*, per lavorare con il filesystem CramFS utilizzato dal firmware;
- *jefferson*, per lavorare con il filesystem JFFS2 utilizzato da alcuni settori di configurazione.

*(In questo elenco sono esclusi i comandi base come `dd` e `file`)*

---

### Risultati iniziali con Binwalk

I primi interessanti risultati ottenuti con binwalk sul dump completo permettono di capire la struttura, più o meno comune a tutti gli ONT analizzati, della NAND:
- **0x404**: Copyright text “Copyright 2008, Cambridge Industry Group(CIG), All Rights Reserved.”
- **0x16410 (ZTE)** o **0x16470 (Nokia)**: uImage firmware image, di circa 93KB, che contiene il bootloader (U-Boot) compresso in lzma e compilato per architettura MIPS32, load address ed entry point 0x81C00000, nome “U-Boot 2011.12.NA-svn145404 for ]”.
- **0x80000**: Settori di configurazione, filesystem JFFS2, big endian, dimensione di circa 1.6MB.
- **0x200000**: Prima immagine (“imagea” nel bootloader) del firmware, filesystem CramFS, big endian, dimensione di circa 6.5MB e controllo CRC errato (ci torneremo più avanti).
- **0x900000**: Seconda immagine (“imageb” nel bootloader) del firmware, con le stesse specifiche della prima immagine.

Dopo un primo momento in cui pensavo che il **kernel** fosse compreso nella uImage a 0x164X0 (di solito esse sono impiegate proprio per contenere il kernel Linux e altre parti fondamentali), ho poi capito invece che esso corrisponde al file “uImage” contenuto nei due CramFS, visto che i log del bootloader da porta seriale suggeriscono il caricamento dell’immagine CramFS prima che il kernel venga avviato.

Concludiamo quindi che il settore a 0x164X0 comprende *solo* il bootloader.

---

### Kernel Modules e Bootloader

A questo punto è stata fondamentale l’analisi dei due CramFS – quasi uguali in tutta la loro struttura – per capire come fosse strutturato il firmware e di quali file il kernel si servisse per far funzionare correttamente il dispositivo.

La prima interessante scoperta è relativa ai file “.ko” (es. `gpon.ko`) contenuti in alcune sottocartelle della directory `lib/modules`: avevano l’aspetto di driver, e questa ipotesi è stata presto confermata dato che si tratta di Kernel Modules (non sono esattamente driver, ma in questo caso il concetto è simile).

Perché quindi non sostituire i file `.ko` del Nokia con quelli dello ZTE per permettere il funzionamento della parte ottica anche con il firmware Nokia? L’ho fatto, e la risposta è: **non funzionerà mai**. Come scritto all’inizio, i kernel sono progettati per lavorare “in sinergia” con i vari file di cui si servono, tra cui i kmodules, e dare in pasto al kernel ZTE i kmodules Nokia non è una buona idea: il risultato è un **kernel panic** e quindi un **bootloop**.

Perché quindi non provare a sostituire anche il kernel Nokia con quello ZTE per permettere ai kmodules di essere riconosciuti dal loro kernel originale? Anche in questo caso, l’ho fatto, e la risposta è: **non funzionerà nemmeno così**. L’avvio del kernel non si serve solo dei kmodules ma anche di altri componenti, e il bootloader ricopre in qualche modo un ruolo particolare nell’avvio del kernel. Il bootloader Nokia (che, ricordiamo, ha la porta seriale sbloccata ed era quindi il mio preferito per gli esperimenti) non sembrava andare perfettamente d’accordo con il kernel ZTE e viceversa. In questi casi, non ottenevo un kernel panic ma il sistema si bloccava su **“Starting kernel …”**. Inizialmente pensavo fosse un problema legato alla porta seriale bloccata lato kernel ZTE, ma anche aspettando 2-3 minuti i LED LOS e LAN non si accendevano, segno che il firmware non veniva caricato.

Ho sperimentato altre combinazioni, ad esempio il bootloader Nokia con tutto il resto ZTE, ma non ho mai ottenuto un risultato positivo. Questo conferma che il bootloader – sebbene le versioni fossero apparentemente uguali tra Nokia e ZTE – è evidentemente adattato dal produttore specifico. Possibilmente, con il bootloader del CIG avrei potuto ottenere più successo, ma non essendo in possesso dell’ONT “puro” non ho potuto approfondire in tal senso.

Una nota a margine riguarda i settori di configurazione, di cui non mi sono dimenticato: anche con uno swap di questi settori tra Nokia e ZTE, per cercare di “allinearli” con il kernel corrispondente, non ho ottenuto buoni risultati.

---

### Studio del Firmware e Modifica di setup.sh

Il prossimo step riguarda lo studio del firmware ZTE per cercare di sbloccare Telnet, SSH e porta seriale “a monte”, senza cambiare bootloader o creare mix tra i due firmware.

Conoscendo per sommi capi la struttura di un firmware Linux e lo scopo dei relativi file e directory, non ho impiegato troppo tempo per capire che un ruolo fondamentale, in quanto a regole, è svolto dal file `setup.sh`, che si trova nella cartella `sbin`.

In particolar modo, mi sono saltate all’occhio tre righe situate verso la fine del file:

```sh
#/bin/Console &
#/bin/telnetd
#/bin/dropbear 1>/dev/null 2>&1
```
Teoricamente:
- La prima riga dovrebbe abilitare la porta seriale;
- La seconda dovrebbe abilitare il Telnet;
- La terza dovrebbe abilitare l’SSH.

Ho subito notato che erano tutte e tre commentate con #, quindi ho proceduto a decommentarle per abilitarle. Oltretutto, il file setup.sh del Nokia ha tutte e tre le righe non commentate, e questo mi ha dato un’ulteriore conferma che **ciò che avevo trovato avrebbe potuto essere la soluzione**.

---

### CRAMFS, CRC e Dimensioni

Ho quindi apportato questa piccola modifica e poi ricompilato i due CramFS, accertandomi che la proprietà del **big endian** fosse rispettata (con macOS non c’era modo di ricreare i CramFS in big endian, ho dovuto usare Linux con il comando `mkfs.cramfs` e il giusto argomento per forzare il big endian). Dopodiché ho inserito i CramFS modificati nel dump ZTE originale.  
Non funzionava niente. Inserendo un attimo il bootloader Nokia (grazie a cui potevo quanto meno capire se ci fossero log particolari tramite la porta seriale), mi sono accorto che U-Boot esegue un check CRC sui CramFS:

```plaintext
Root Filesystem crc check error! srcCrc = ffffffff calcCrc = e52ae918

### CRAMFS LOAD ERROR<ffffffff> for uImage!
```
Il CRC è un metodo di verifica dell’integrità dei dati e si basa su un calcolo matematico che genera un checksum derivato dal contenuto stesso; questo valore viene confrontato con quello atteso e determina, nel nostro caso, il corretto caricamento dei CramFS da parte del bootloader.
Avendo trovato documentazioni discrepanti in merito al CRC (in alcuni casi è situato nei primi 64 byte di un file, in altri proprio alla fine, ma in ogni caso è fatto da 4 byte), sono andato a tentativi fin quando U-Boot non ha gradito il file.

Per comodità i tentativi non sono stati fatti flashando ogni volta il dump completo, ma ho utilizzato il comando `upgdimage` di U-Boot, che serve ad aggiornare i CramFS direttamente dal bootloader tramite protocollo TFTP (il bootloader si aspetta in particolar modo un file `cramfs.img.crc`) ed esegue anche in questo caso il check. Non avrei potuto usare il comando per far funzionare tutto, poiché il bootloader ZTE, come sappiamo, ha la porta seriale bloccata e con il bootloader Nokia non avrei potuto far girare il firmware ZTE, ma è stato molto utile e più pratico.

Dopo diverse prove sono arrivato alla conclusione che U-Boot non guarda il CRC posizionato nei primi 64 byte (che è invece quello visualizzato, ad esempio, dal comando `file nomedump.bin` e che eventualmente determina il “checksum error” di binwalk), **ma vuole un CRC specifico** – che per fortuna viene scritto, come si evince dal log sopra – alla fine del file, negli ultimi 4 byte.

Ma non è tutto: prima dell’errore sul CRC, il comando `upgdimage` **controlla la dimensione del CramFS** e in tutte le prove che facevo si aspettava sempre 4 byte in più:
```plaintext
RootFS CRAMFS size [0x651004] length [0x651000]
Cramfs image length is error 651004 3 651000
```
Probabilmente questo è dovuto alla mancata gestione corretta dell’allineamento o dal CRC mal posizionato. In realtà credo che la causa corretta sia la seconda, poiché questi 4 byte in più dovranno essere riempiti proprio dal CRC che il bootloader vuole.

Risolti i due problemi, ho reinserito i due CramFS (si sta sempre parlando di quelli con il setup.sh modificato) tramite il solito comando `dd`, già utilizzato svariate altre volte per i “taglia e cuci” che ho fatto con tutti i dump, sia in 0x200000 sia in 0x900000.

Flashato tutto e acceso l’ONT… **Funziona!** Le spie LOS e LAN si accendono. Provo subito a collegarmi via Telnet (`telnet 192.168.1.1`) e… **funziona anche questo!** Le credenziali però non sono admin/admin come nel caso della web GUI, ma root/admin: anche in questo caso sono andato a tentativi.

Purtroppo, non funziona né SSH né la porta seriale. Per quanto riguarda il primo, credo che la causa sia da ricercare nelle regole di iptables oppure nel daemon dropbear non configurato o non presente nel firmware ZTE; sulla seconda, credo che il problema risieda nel kernel, che blocca la porta a monte ancor prima di eventuali regole dell’OS.

---

### Considerazioni Finali

È interessante notare che, alla fine, se spiegassi a qualcuno come agire senza raccontare tutto ciò che ho fatto prima, risulterebbe una procedura tutto sommato semplice (a patto di avere le giuste conoscenze e competenze preliminari) e breve da applicare. Tuttavia, “l’essenza” sta piuttosto in tutto ciò che ho fatto per arrivare alla conclusione.

Ritengo corretto specificare che l’unica funzione sbloccata risulterà il Telnet e quindi, come già scritto, SSH e porta seriale rimarranno bloccate. Inoltre, il bootloader non avrà intenzione di dialogare tramite porta seriale perché anch’esso la blocca.  
Nonostante ciò, il Telnet è tutto ciò di cui si potrebbe aver bisogno per far autenticare l’ONT, poiché da lì è possibile impartire qualsiasi comando che si impartirebbe via SSH o via porta seriale (a OS caricato).

Un bootloader sbloccato permetterebbe di fare parecchie cose interessanti, tra cui:
- Un upgrade di CramFS/kernel/bootloader senza flashare il dump completo sulla SPI flash;
- Uno swap di immagini di avvio tra `0x200000` e `0x900000` (che però non è necessario visto che ho flashato lo stesso CramFS in entrambi i settori);
- Altre operazioni avanzate.

Tuttavia, per chi vuole semplicemente modificare i parametri per la propria connessione, il Telnet è più che sufficiente.

In futuro proverò a lavorare sul kernel dello ZTE e sul bootloader, per trovare eventuali regole che attualmente bloccano la porta seriale. Proverò a capire inoltre come abilitare la porta SSH lato OS.
