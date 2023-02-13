---
marp: true
theme: default
size: 16:9
paginate: true
style: |
    img[alt~="center"] {
        display: block;
        margin: 0 auto;
    }
    section::after {
        content: attr(data-marpit-pagination) ' / ' attr(data-marpit-pagination-total);
    }
---

# Sicurezza Hardware in Ambienti Virtualizzati Tramite un Passthrough TEE tra QEMU e Linux

#
#

## Marco Cutecchia
## Relatore: Danilo Bruschi

---

# Trusted Execution Environment (TEE)

- Ambienti isolati all'interno di un dispositivo dove eseguire codice critico
- Garantiscono l'integrità dei dati e del codice, prevedendo anche meccanismi di verifica da remoto (Remote Attestation)
- Garantiscono la riservatezza dei dati, sia quando sono salvati su disco sia quando sono in memoria
    - Esempio: Memoria RAM crittografata
- Utilizzano dei meccanismi hardware per garantire queste proprietà
    - ARM TrustZone, Intel SGX, AMD SEV

---


# Perchè usare un TEE?

- Mobile: proteggere i dati sensibili, anche nel caso un malware riuscisse a prendere il controllo del dispositivo
- IoT: costruire sensori sicuri, che non possano essere facilmente compromessi anche se l'attaccante ha accesso fisico al dispositivo
- Cloud: permettere lo sviluppo di servizi alla quale gli utenti  possono mandare dati sensibili, avendo la garanzia che questi non possano essere letti da nessuno, neanche il gestore del servizio
    * Realizzare il **Confidential Computing**

---

# Lo stato attuale del Confidential Computing

- Alcuni provider offrono servizi di Confidential Computing, ma solo per alcuni tipi di macchine, e spesso con lock-in
- Gli sforzi al momento sono per permettere ai clienti di eseguire il proprio codice in un TEE di loro completa responsabilità

#

![center width:1000px](./immagini/png/common-cloud-tee-model.png)

---

# Proposta alternativa

- Il provider offre un TEE con dei servizi utili pre-installati e utilizzabili dagli utenti

- Interfaccia esterna GlobalPlatform, dunque compatibile con la maggior parte dei TEE esistenti

- Può coesistere con l'altro modello

![center width:700px](./immagini/png/our-cloud-tee-model.png)

---

# Come implementare questo modello

- È necessario implementare un canale di comunicazione tra le macchine virtuali e il TEE, passando attraverso l'hypervisor
- Questo canale può essere diviso in tre componenti:
    * Un device virtuale, dentro l'hypervisor, per comunicare con il TEE
    * Un driver per questo device, da installare nel guest
    * Un layer di traduzione tra le chiamate del driver e le chiamate del TEE, implementato nell'hypervisor

---

# Il device virtuale e il driver

- Interfaccia basata su MMIO e un semplice protocollo di comunicazione a messaggi
- Driver integrato nel sottosistema TEE di Linux, il guest può accedere al TEE come se fosse un device normale

---

# Sequenza di invio di un messaggio

1. Il driver riceve le richieste dalle applicazioni, le traduce in messaggi corrispondenti e segnala al device virtuale la posizione del messaggio in memoria
 
2. QEMU riceve le richieste del driver, le fa passare attraverso un layer di traduzione, e le inoltra al TEE

3. Il TEE risponde all'hypervisor, che fa passare la risposta ancora attraverso il layer di traduzione, e la scrive nella memoria del guest

4. Il driver legge la risposta dal device virtuale e la restituisce all'applicazione

---

# Layer di traduzione

- Ogni messaggio che TEE e macchine virtuali scambiano possono contenere valori che hanno un significato diverso per le due parti
    * Esempio: Indirizzi di memoria validi in una macchina virtuale, non lo sono nell'hypervisor
- Questo layer ispeziona i messaggi inoltrati e traduce puntatori, identificatori di sessioni, ecc. in modo che siano validi per entrambe le parti

---

# Verificare la correttezza dell'implementazione

- `xtest` è una suite di oltre 40000 test case utilizzati per verificare la correttezza di OP-TEE, un TEE open source per piattaforme ARM TrustZone
- Abbiamo anche effettuato dei primitivi test delle prestazioni, misurando il tempo impiegato per eseguire l'intera suite di test
    * **Overhead di circa l'8%** rispetto a un sistema con accesso diretto al TEE
    * Abbiamo identificato alcune aree di miglioramento che possono ridurlo a meno del 2%
---

# Conclusioni

- Riteniamo che il nostro prototipo dimostri la fattibilità e sensatezza di questo modello operativo
- È un punto di partenza promettente, ma rimangono ancora delle sfide da affrontare:
    * Migrazione dei dati sensibili tra TEE
    * Migliorare le prestazioni
    * Implementare su un hypervisor di tipo 1
