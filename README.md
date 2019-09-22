# Lernenden Spamfilter auf Uberspace7 selbst betreiben 
Vorweg: **Ein Dank an Bernhard Ehlers!** Diese Anleitung basiert auf seinem Blogtext und seinen Config-Dateien. Er gab mir die Erlaubnis, sie als Grundlage zu verwenden.
## Hintergrund
Stand heute bietet der Hosting-Anbieter [uberspace](https://www.uberspace.de) auf seinem Produkt [Uberspace7](https://blog.uberspace.de/tag/uberspace7/) (abgekürzt U7) von Haus aus keinen *lernenden* bzw. *trainierbaren* Spamfilter an. Bisher existiert unter U7 vorimplementiert nur folgender, sehr rudimentärer, Spamschutz: Jede eingehende E-Mail wird automatisch mit einem Rspamd-Score versehen und bei einem Wert größer 15 sofort gelöscht (siehe [U7-Manual > 'Filtering mails'](https://manual.uberspace.de/mail-filter.html)). Es ist möglich, via qmail und mailfilter, auch bei niedrigeren Rspamd-Scores eigene Verarbeitungsschritte einzubauen. Wer Spam effektiv ausfiltern möchte, der benötigt einen trainierbaren Spamfilter, der in viele Mailclients fest integriert ist. Wer über mehrere Geräte hinweg auf E-Mails zugreift, wird aber nicht auf einen serverseitigen lernenden Spamfilter verzichten wollen. Und bis die Ubernauten einen solchen Filter auf U7 anbieten, hilft diese Anleitung dabei, sich selbst einen solchen Spamfilter einzurichten.

An dieser Stelle noch der Hinweis, dass es auf dem Vorgängerprodukt U6 mit SpamAssassin (mit integriertem Regelwerk) und DSPAM (trainierbar) vorinstallierte Spamfilter gibt/gab. Diese funktionieren zwar, werden bei U7 aber nicht mehr integriert werden.

## Spamfiltersoftware der Wahl: CRM114

Mit folgendem Zitat von Bernhard ist alles gesagt:
> "Nach etwas Recherche habe ich mich für [CRM114](http://crm114.sourceforge.net) entschieden. Es wurde zwar sehr lange nicht mehr aktualisiert, ist aber weiterhin in vielen Linux-Distributionen präsent. Der Lernalgorithmus ist sehr schnell und effizient, das Programm ist sehr klein und performant."

## Voraussetzung: Mailaccount mit korrekter Ordnerstruktur

Es besteht ein Mailaccount, der mit 'uberspace mail user add [USERNAME]' erstellt wurde.\
(siehe [U7-Manual > 'Mailboxes' > 'Main mailbox'](https://manual.uberspace.de/mail-mailboxes.html))

In diesem Mailaccount muss folgende Ordnerstruktur vorhanden sein:
```
  INBOX
   \
    0 Spamfilter
     \
      \___ als Ham lernen
      |
      |___ als Spam erkannt
      |
      |___ als Spam lernen
```                 
Diese Ordnerstruktur erstellt man komfortabel mit folgenden Befehlen:

Zuerst folgenden Befehl **ABGEWANDELT!** in der Shell ausführen.\
**WICHTIG: [#USERNAME#] muss, einschließlich Rauten und der eckigen Klammern, durch den Benutzernamen des Mailaccount ersetzt werden!**
```Shell
export MAILUSERNAME=[#USERNAME#]  # [#USERNAME#] durch den richtigen Mailaccountbenutzer ersetzen!
```

Anschließend die folgenden vier Befehle in der Shell ausführen:
```Shell
test -d "$HOME/users/$MAILUSERNAME/.0 Spamfilter" || maildirmake "$HOME/users/$MAILUSERNAME/.0 Spamfilter"                
test -d "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Ham lernen" || maildirmake "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Ham lernen"
test -d "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Spam lernen" || maildirmake "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Spam lernen"
test -d "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Spam erkannt" || maildirmake "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Spam erkannt"
```

Prüft nun in eurem Mailclient, ob die Ordner erstellt wurden!\
Häufig ist es nötig, neu angelegte Ordner in den Einstellungen erst noch manuell sichtbar zu machen / zu abbonieren bevor sie im Mailclient auftauchen!

## Installation & Einrichtung CRM114

Die folgenden Befehle installieren und konfigurieren CRM114 in den Ordner crm114.\
**Wichtig**: Sofern dieser Ordner bereits existiert wird er ohne Rückfrage überschrieben.

```Shell
mkdir -p ~/crm114       # Erzeuge Ordner 'crm114' (Wenn vorhanden wird ohne Rückfrage überschrieben)
cd ~/crm114             # Wechsle in den Ordner 'crm114'
curl -sSL http://crm114.sourceforge.net/tarballs/crm114-20100106-BlameMichelson.src.tar.gz | tar xz # Kopiere und entpacke CRM114
cd crm114-20*           # Wechsle in den CRM114-Programmordner
curl -sSL https://laurikari.net/tre/tre-0.8.0.tar.bz2 | tar xj
cd tre*
./configure --prefix "`cd ..; pwd`/tre" --enable-static
make install
cd ..
sed -i 's/^LDFLAGS/#LDFLAGS/' Makefile
sed -i 's|^\([^#].*\)-ltre|\1tre/lib/libtre.a|' Makefile
CFLAGS="-std=gnu89 -Wno-unused-but-set-variable -I tre/include" LDFLAGS="$CFLAGS" make
strip -s crm114 cssdiff cssmerge cssutil osbf-util
cp -p crm114 cssdiff cssmerge cssutil osbf-util ..
cp -p mailfilter.crm maillib.crm mailreaver.crm mailtrainer.crm rewriteutil.crm shuffle.crm ..
chmod 755 ../*.crm
for i in mailfilter.cf blacklist.mfp priolist.mfp rewrites.mfp; do cp -p $i ../$i.example; done
cp -p whitelist.mfp.example ..
cd ..
rm -r crm114-20*
curl -sSL https://github.com/flowdx/crm114_u7/archive/master.zip -o master.zip # Die Konfigurationsdateien aus diesem Github-Repository herunterladen
unzip master.zip # Das gezippte Repository entpacken
rm master.zip    # Das gezippte Repository löschen
mv ./crm114_u7-master/* ./ # Die Dateien aus dem Entpackordner in den Hauptordner verschieben
rm -r crm114_u7-master     # Den Entpackordner löschen
chmod 755 cache_cleanup crm114 cssdiff cssmerge cssutil db_init learn_maildir # Die korrekte Berechtigung für die ausführbaren Dateien vergeben
sh db_init # Die Einrichtung der Datenbank durchführen
cd ..

```

## Anlernen und Aufräumen automatisieren

### Regelmäßiges Anlernen via crontab aktivieren

Das Script learn_maildir nimmt die Mails aus den Ordnern "als Ham lernen" und "als Spam lernen" und lernt sich über diese an. Dazu müssen die Mails aus dem Posteingang in den jeweils richtigen Ordner **kopiert** werden. Nach
dem Anlernen löscht das Script die E-Mails nämlich, die in diesen Ordnern abgelegt wurden. Mit folgenden Cronjob wird das Script learn_maildir alle 20 Minuten aufgerufen.

Die Crontab aufrufen mit:
```Shell
crontab -e
```
und dort folgende Zeile ergänzen:
```
*/20 * * * *	sleep $((RANDOM \% 40 + 10)); crm114/learn_maildir
```

### Regelmäßig automatisches Aufräumen des Caches via crontab aktivieren

cache_cleanup sorgt dafür, dass die Cache-Dateien regelmäßig entschlackt werden.

Die Crontab aufrufen mit:
```Shell
crontab -e
```
und dort folgende Zeile ergänzen:
```
32 4 * * 0,3	crm114/cache_cleanup
```


## Spamerkennung für einen Mailaccount einrichten

**Auch hier gilt: Jedes [#USERNAME#] bitte durch den Benutzernamen der E-Mail-Adresse ersetzen!**\
**Und: Die folgenden Schritte sind so nur gültig, wenn die jeweiligen qmail- und die mailfilter-Dateien bisher noch nicht vorhanden sind.**

### .Mailfilter-Datei erstellen
Zuerst folgende Befehle, **immer jeweils angepasst**, ausführen. Durch den nano-Befehl öffnet sich die Texteingabe für die Datei.
```Shell
cd
touch .mailfilter_[#USERNAME#] # CHANGE!     # Erstellt die Datei, aber nur, sofern sie noch nicht existiert
chmod 600 .mailfilter_[#USERNAME#] # CHANGE! # Ändert auf die von U7 benötigte Rechte-Einstellung
nano .mailfilter_[#USERNAME#] # CHANGE!      # Öffnet die Datei zum Bearbeiten
```
In die Datei folgendes eintragen, Copy & Paste ist möglich. Auch hier das Anpassen in der ersten Zeile nicht vergessen! Nach dem Ändern mit Strg+X schließen und das Speichern bestätigen.
```
MAILUSERNAME=[#USERNAME#] # CHANGE!
MAILDIR="$HOME/users/$MAILUSERNAME"
MAILDIRSPAM="$MAILDIR/.0 Spamfilter.als Spam erkannt"

# Show mail to Bayes-Spamfilter (CRM114) if it's not larger than 2MB (=2000000 Bytes)
if ($SIZE < 2000000)
{
    xfilter "'$HOME/crm114/crm114' -u '$HOME/crm114' mailreaver.crm --"
    
    # if it's classified as spam
    if (/^X-CRM114-Status: *SPAM/:h)
    {
        # save it to spamfolder
        to "$MAILDIRSPAM"
    }
}
to "$MAILDIR"       
```
### .qmail-Datei erstellen 

Folgende Befehle, **immer jeweils angepasst**, ausführen. Durch den nano-Befehl öffnet sich die Texteingabe für die Datei.

```Shell
cd
touch .qmail-[#USERNAME#] # CHANGE!
chmod 644 .qmail-[#USERNAME#] # CHANGE!
nano .qmail-[#USERNAME#] # CHANGE!
```
In die Datei folgendes eintragen, auch hier das [#USERNAME#] ersetzen. Dann mit Strg+X und Bestätigung speichern und schließen.
```Shell
|maildrop $HOME/.mailfilter_[#USERNAME#] # CHANGE!
```

