# Lernenden Spamfilter auf Uberspace7 selbst betreiben 
Vorweg: **Ein Dank an Bernhard Ehlers!** Diese Anleitung basiert auf seinem Blogtext und seinen Config-Dateien. Er gab mir die Erlaubnis, sie als Grundlage zu verwenden.
## 1. Hintergrund
Stand heute bietet der Hosting-Anbieter [uberspace](https://www.uberspace.de) auf seinem Produkt [Uberspace7](https://blog.uberspace.de/tag/uberspace7/) (abgekürzt U7) von Haus aus keinen *lernenden* bzw. *trainierbaren* Spamfilter an. Bisher existiert unter U7 vorimplementiert nur folgender, sehr rudimentärer, Spamschutz: Jede eingehende E-Mail wird automatisch mit einem Rspamd-Score versehen und bei einem Wert größer 15 sofort gelöscht (siehe [U7-Manual > 'Filtering mails'](https://manual.uberspace.de/mail-filter.html)). Es ist möglich, via qmail und mailfilter, auch bei niedrigeren Rspamd-Scores eigene Verarbeitungsschritte einzubauen. Wer aber Spam effektiv ausfiltern möchte, der benötigt einen trainierbaren Spamfilter, der in viele Mailclients fest integriert ist. Wer über mehrere Geräte hinweg auf E-Mails zugreift, wird aber nicht auf einen serverseitigen lernenden Spamfilter verzichten wollen. Und bis die Ubernauten einen solchen Filter auf U7 anbieten, hilft diese Anleitung dabei, sich selbst einen solchen Spamfilter einzurichten.

Noch der Hinweis, dass es auf dem Vorgängerprodukt U6 mit SpamAssassin (mit integriertem Regelwerk) und DSPAM (trainierbar) vorinstallierte Spamfilter gibt/gab. Diese funktionieren auf U6 auch nach wie vor, sind aber unmaintained. Deshalb werden sie auf U7 nicht mehr eingesetzt werden. Diese Anleitung imitiert in etwa das Verhalten von DSPAM auf U6, indem es CRM114 auf U7 nutzt. Allen, die das Spamfiltern und Anlernen von U6 kennen, sollten sich hier also schnell zurechtfinden.
## 2. Spamfiltersoftware der Wahl: CRM114

Mit folgendem Zitat von Bernhard ist alles gesagt:
> "Nach etwas Recherche habe ich mich für [CRM114](http://crm114.sourceforge.net) entschieden. Es wurde zwar sehr lange nicht mehr aktualisiert, ist aber weiterhin in vielen Linux-Distributionen präsent. Der Lernalgorithmus ist sehr schnell und effizient, das Programm ist sehr klein und performant."

## 3. Grundlagen

### 3.1 Die Shell benutzen ;)

Alle Befehle hier werden via SSH-Zugriff in der Shell eingegeben. Darüber hinaus werden Dateien und Crontab mit nano bearbeitet.

### 3.2 Die Platzhalter-Variable [!USERNAME!]

In der folgenden Anleitung wird an mehreren Stellen die Platzhalter-Variable [!USERNAME!] verwendet. Diese ist **IMMER** durch einen selbstgewählten Benutzernamen zu ersetzen, und zwar unter Wegfall der eckigen Klammern
und der Ausrufezeichen!

**Beispiel**:\
Du entscheidest dich für den Mailbenutzer `nureintestbenutzer`. In der Anleitung steht: Bitte `uberspace mail user add [!USERNAME!]` eingeben. Die Eingabe muss dann lauten: `uberspace mail user add nureintestbenutzer`

## 4. Voraussetzung #1: nano als Standard-Editor

Folgende Befehle ausführen
```Shell
cd
nano .bash_profile
```

Nun öffnet sich die Datei .bash_profile. Dort drin eine weitere Zeile mit folgendem Text unten anfügen:
```
export VISUAL='nano'
```
Nun Speichern und Schließen. Mit nano geht das immer so: Tastenkombination Strg+X drücken, dann ein `y` eintippen und mit Enter bestätigen.

Jetzt, ganz wichtig: SSH-Verbindung beenden, und anschließend neu verbinden. Nur so wird die Umstellung des Standardeditors auf nano aktiv.

## 5. Voraussetzung #2: Ein Mailaccount mit korrekter Ordnerstruktur

Zum Testen dieses Tutorials empfehle ich, eine bereits vorhandene E-Mail-Adresse nicht anzutasten. Stattdessen erstmal eine Test-E-Mail-Adresse einrichten. Es ist problemlos möglich, die
Spamfilterung bei Gefallen auch auf alle weiteren gewünschten E-Mail-Adressen auf dem selben U7 auszuweiten.

**Hier noch ein letztes mal der Verweis auf:** ***"3.2 Die Platzhalter-Variable [!USERNAME!]"***\
Du könntest ab hier z.B. `[!USERNAME!]` konsequent durch `spamfiltertest` ersetzen. 

### 5.1 Neuen Mailaccount anlegen
Also bitte einen neuen Mailaccount auf dem U7 anlegen, dafür den Befehl `uberspace mail user add [!USERNAME!]` nutzen.\
(siehe [U7-Manual > 'Mailboxes' > 'Main mailbox'](https://manual.uberspace.de/mail-mailboxes.html))

### 5.2 Benötigte Ordnerstruktur
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

### 5.3 Diese geforderte Ordnerstruktur einrichten
                 
Die geforderte Ordnerstruktur lässt sich mit folgenden Befehlen komfortabel einrichten.

Zuerst eine Umgebungsvariable mit dem Benutzernamen füllen, unter dem der Mailaccount angelegt wurde:

```Shell
export MAILUSERNAME=[!USERNAME!]
```

Anschließend die folgenden vier Befehle in der Shell ausführen um die Ordner in dem Mailaccount anzulegen:
```Shell
test -d "$HOME/users/$MAILUSERNAME/.0 Spamfilter" || maildirmake "$HOME/users/$MAILUSERNAME/.0 Spamfilter"                
test -d "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Ham lernen" || maildirmake "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Ham lernen"
test -d "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Spam lernen" || maildirmake "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Spam lernen"
test -d "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Spam erkannt" || maildirmake "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Spam erkannt"
```

Prüft nun in eurem Mailclient, ob die Ordner erstellt wurden!\
Häufig ist es nötig, neu angelegte Ordner in den Einstellungen erst noch manuell sichtbar zu machen bzw. zu abbonieren, bevor sie im Mailclient auftauchen!\
Im [RainLoop-Webmail-Client von Uberspace7](https://webmail.uberspace.de/) blendet man weitere Ordner wie folgt ein: `Settings > Folders` aufrufen und über die jeweiligen Augen-Icons einblenden.

## 6. Installation & Einrichtung CRM114

Die folgenden Befehle installieren und konfigurieren CRM114 in den Ordner *~/crm114*.\
(Dazu wird u.a. auch TRE (The free and portable approximate regex matching library) von Laurikari installiert sowie Konfigurationsdateien und Scripte hier aus diesem Repository gezogen.)

**Achtung**: Sofern der Ordner ~/crm114 bereits existiert wird er durch folgende Befehle ohne Rückfrage überschrieben.

Der folgende Codeblock enthält keine [!USERNAME!]-Variable. Demnach wäre es möglich, den folgenden Block komplett via Copy&Paste auszuführen.

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
cd ~ # Zurück in den $USER-Ordner

```

## 7. Anlernen und Aufräumen via Cronjob automatisieren

CRM114 lernt, indem dem Programm, durch den Benutzer (also dich!), eingegangene E-Mails entweder als Spam (im Ordner "als Spam lernen") oder Ham ("als Ham lernen") präsentiert werden.

Das Script 'learn_maildir' prüft alle vorhandenen Mailaccounts auf diese beiden Ordner. Wird ein Ordner gefunden und sind darin Mails vorhanden, dann werden diese durch CRM114 verarbeitet und anschließend gelöscht. Im Cronjob wird eine Zeile eingefügt, die das Script learn_maildir alle 20 Minuten automatisch aufruft.

Das Script 'cache_cleanup' sorgt dafür, dass die Cache-Dateien von CRM114 regelmäßig entschlackt werden. Auch dafür wird im folgenden ein Cronjob angelegt, der regelmäßig das Script cache_cleanup aufruft.

Zuerst Crontab aufrufen mit dem Bearbeitungsprogramm nano:
```Shell
nano crontab -e
```
und dort die folgende zwei Zeilen am Ende der Datei ergänzen:
```
*/20 * * * * sleep $((RANDOM \% 40 + 10)); crm114/learn_maildir
32 4 * * 0,3 crm114/cache_cleanup
```
Nun Speichern und Schließen. Mit nano geht das immer so: Tastenkombination Strg+X drücken, dann ein `y` eintippen und mit Enter bestätigen.

## 8. Spamerkennung für einen Mailaccount einrichten

**Wichtig: Die folgenden Schritte sollten so nur ausgeführt werden, wenn die .qmail-Datei und die .mailfilter-Datei bisher noch nicht vorhanden sind.**

### 8.1 .mailfilter-Datei erstellen
Zuerst folgende Befehle, **immer jeweils angepasst**, ausführen. Durch den nano-Befehl öffnet sich die Texteingabe für die Datei.
```Shell
cd ~
touch .mailfilter_[!USERNAME!]        # Erstellt die Datei, aber nur, sofern sie noch nicht existiert
chmod 600 .mailfilter_[!USERNAME!]    # Ändert auf die von U7 benötigte Rechte-Einstellung
nano .mailfilter_[!USERNAME!]         # Öffnet die Datei zum Bearbeiten
```
Durch den nano-Befehl öffnete sich die Texteingabe für die Datei.

Den folgenden Text in den nun offenen nano-Editor eingeben:\
(Copy&Paste ist möglich. Auch hier das Anpassen in der ersten Zeile nicht vergessen!)
```
MAILUSERNAME=[!USERNAME!]             # <--- Benutzernamen anpassen nicht vergessen!
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
Nun speichern und schließen wie gewohnt mit: Strg+X, `y`, Enter.

### 8.2 .qmail-Datei erstellen 

Folgende Befehle, **immer jeweils angepasst**, ausführen.

```Shell
cd ~
touch .qmail-[!USERNAME!]       # Erstellt die Datei, aber nur, sofern sie noch nicht existiert
chmod 644 .qmail-[!USERNAME!]   # Ändert auf die von U7 benötigte Rechte-Einstellung
nano .qmail-[!USERNAME!]        # Öffnet die Datei zum Bearbeiten
```
Durch den nano-Befehl öffnete sich die Texteingabe für die Datei.

Den folgenden Text, ist nur eine Zeile, in den nun offenen nano-Editor eingeben:\
(Copy&Paste ist möglich. Auch hier das Anpassen in der ersten Zeile nicht vergessen!)
```Shell
|maildrop $HOME/.mailfilter_[!USERNAME!]
```
Nun speichern und schließen wie gewohnt mit: Strg+X, `y`, Enter.

## 9. OPTIONAL: Produktive E-Mail-Adresse in den Test einbeziehen

Es ist möglich, E-Mails der produktiven E-Mail-Adresse auf regulärem Weg (ohne Spamfilterung) zuzustellen und **zusätzlich** eine Weiterleitung auf die Test-E-Mail-Adresse mit Spamfilterung
zuzustellen. So kann das Verhalten des Spamfilters bei real ankommenden Mails getestet werden, aber ohne in die produktive E-Mail-Adresse einzugreifen.

Dafür die **vorhandene** .qmail-Datei der produktiven E-Mail-Adresse um den Verweise auf maildrop mit dem mailfilter **ergänzen**:

Die Datei, z.B. `.qmail-meinehauptmailadresse`,  könnte dann so aussehen:
```
./users/meinehauptmailadresse/              # Wie bisher/unverändert. Die E-Mail wird regulär an den Mailaccount zugestellt
|maildrop $HOME/.mailfilter_[!USERNAME!]    # Zusätzlich wird die Mail via Mailfilter an den Test-Account zugestellt
```

## 10. Test beenden und Spamfilter für die produktive E-Mail-Adresse einrichten
Sofern man mit dem Test zufrieden ist, führt man später folgende Schritte durch:
- Die geforderte Ordnerstruktur in der produktiven E-Mail-Adresse erstellen
- Eine Mailfilter-Datei für die produktive E-Mail-Adresse erstellen, z.B. `.mailfilter_meinehauptmailadresse`
- Die vorhandene .qmail-Datei der produktiven E-Mail-Adresse (z.B. `.qmail-meinehauptmailadresse`) wie folgt ändern
```
#./users/meinehauptmailadresse/                     # Die Zeile auskommentieren, so dass keine E-Mails direkt ohne Spamfilter zugestellt werden
|maildrop $HOME/.mailfilter_meinehauptmailadresse   # Die Mail wird an die neue .mailfilter-Datei übergeben
```
