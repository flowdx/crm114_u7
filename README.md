# Lernenden Spamfilter auf Uberspace7 selbst betreiben 
Vorweg: **Ein Dank an Bernhard Ehlers!** Diese Anleitung basiert auf seinem Blogtext und seinen Config-Dateien. Er gab mir die Erlaubnis, sie als Grundlage zu verwenden.

# DIESES TUTORIAL BEFINDET SICH NOCH IM AUFBAU!

I   - Grundsätzliches und Vorbereitungen\
II  - Installation der Spamfilter-Software\
III - Die Lernfunktionen aktivieren\
IV  - Notwendige Ordner im Mailaccount anlegen\
V   - Aktivierung der automatischen Spamprüfung für einen Mailaccount\
VI  - Anleitung zur Nutzung und zum Anlernen

# I - Grundsätzliches und Vorbereitungen
## 1. Hintergrund
Stand heute bietet der Hosting-Anbieter [uberspace](https://www.uberspace.de) auf seinem Produkt [Uberspace7](https://blog.uberspace.de/tag/uberspace7/) (abgekürzt U7) von Haus aus keinen *lernenden* bzw. *trainierbaren* Spamfilter an. Bisher existiert unter U7 vorimplementiert nur folgender, sehr rudimentärer, Spamschutz: Jede eingehende E-Mail wird automatisch mit einem Rspamd-Score versehen und bei einem Wert größer 15 sofort gelöscht (siehe [U7-Manual > 'Filtering mails'](https://manual.uberspace.de/mail-filter.html)). Grundsätzlich wäre es möglich, via qmail und mailfilter, auch für niedrigere Rspamd-Scores eigene Verarbeitungsschritte einzubauen und so die Schwelle für Spam/Ham den eigenen Bedürfnissen nach festzulegen. Um aber Spam wirklich effektiv ausfiltern zu können, benötigt man einen trainierbaren Spamfilter, der in vielen Software-Mailclients wie z.B. Thunderbird häufig fester Bestandteil ist. Wer über mehrere Geräte hinweg auf E-Mails zugreift wird aber nicht auf einen serverseitigen lernenden Spamfilter verzichten wollen. Und bis die Ubernauten einen solchen Filter auf U7 anbieten, hilft diese Anleitung dabei, sich selbst einen solchen Spamfilter einzurichten.

Noch der Hinweis, dass es auf dem Vorgängerprodukt U6 vorinstallierte Spamfilter-Software gibt (SpamAssassin (mit integriertem Regelwerk) und DSPAM (trainierbar)), die man selbst einrichten muss/kann. Auf U7 werden diese Spamfilter-Programme aber nicht mehr von Vornherein angeboten, da sie unmaintained sind. Diese Anleitung imitiert in etwa das, von U6 gewohnte Verhalten von DSPAM, indem es CRM114 auf U7 nutzt. Allen, die das Spamfiltern und Anlernen von U6 kennen, sollten sich hier also schnell zurechtfinden.
## 2. Spamfiltersoftware der Wahl: CRM114

Mit folgendem Zitat von Bernhard ist alles gesagt:
> "Nach etwas Recherche habe ich mich für [CRM114](http://crm114.sourceforge.net) entschieden. Es wurde zwar sehr lange nicht mehr aktualisiert, ist aber weiterhin in vielen Linux-Distributionen präsent. Der Lernalgorithmus ist sehr schnell und effizient, das Programm ist sehr klein und performant."

Der Rspamd-Score, den die Ubernauten für jede E-Mail automatisch bereitstellen, wird im Rahmen dieses Tutorials nicht verwendet. CRM114 verrichtet bei mir seine Arbeit so gut, dass mir dies nicht notwendig erscheint. Eine Kombination aus CRM114 und Rspamd ist natürlich möglich, wird hier aber nicht weiter beschrieben.

## 3. Grundlagen

### 3.1 Die Shell benutzen ;)

Alle Befehle hier werden via SSH-Zugriff in der Shell eingegeben. Darüber hinaus werden Dateien und Crontab mit nano bearbeitet.

### 3.2 Die Platzhalter-Variable [!USERNAME!]

In der folgenden Anleitung wird an mehreren Stellen die Platzhalter-Variable [!USERNAME!] verwendet. Diese ist **IMMER** durch einen selbstgewählten Benutzernamen zu ersetzen, und zwar unter Wegfall der eckigen Klammern
und der Ausrufezeichen!

**Beispiel**:\
Du entscheidest dich für den Mailbenutzer `nureintestbenutzer`. In der Anleitung steht: Bitte `uberspace mail user add [!USERNAME!]` eingeben. Die Eingabe muss dann lauten: `uberspace mail user add nureintestbenutzer`

## 4. Einschränkungen

- Es können nur E-Mails bis zu einer Gesamtgröße von ca. 3,9MB angelernt werden. Größere E-Mails werden ohne Anlernen verworfen.
Grund: CRM114 bricht den Anlernvorgang mit einem Fehler ab, sobald es eine E-Mail mit zu großer Gesamtgröße in einem der Anlernordner findet. Die E-Mail verbleibt dann in dem Ordner und blockiert weiteres Anlernen. Es handelt sich dabei um ein cachebezogenes Problem der Komponenten von CRM114, für das ich keinen Fix gefunden haben. Aus diesem Grund der Workaround, E-Mails mit mehr als 3,9MB beim Anlernvorgang aus den Anlernordnern zu löschen, ohne sie CRM114 zu präsentieren. Auf eigene Verantwortung kann dieses Limit in der Datei 'learn_maildir' angepasst werden. 

- Eingehende E-Mails, die größer als 2MB sind, werden nicht an CRM114 zur Spamrüfung übergeben. Dies geschieht, um die Last für CRM114 gering zu halten und weil "echte" Spammails selten größer sind als 2MB. Auf eigene Verantwortung kann dieser Wert in der .mailfilter-Datei abgeändert werden. 
## 5. nano zum Standardeditor machen

Sofern nicht eh schon geschehen, bitte jetzt nano als Standardeditor festlegen. So kann der Rest der Anleitung so durchgegangen werden, wie beschrieben. Wenn gewünscht, kann dieser Schritt nach Beenden dieses Tutorials wieder rückgängig gemacht werden.

Um nano zum Standardeditor zu machen, bitte folgende Befehle in der Shell ausführen:
```Shell
cd
nano .bash_profile
```
Durch den nano-Befehl öffnet sich die Datei .bash_profile

Dort drin eine weitere Zeile mit folgendem Text unten anfügen, sofern diese Zeile noch nicht existiert.
```
export VISUAL='nano'
```
Nun speichern und schließen.
Mit nano geht das immer so: Tastenkombination Strg+X drücken, dann ein `y` eintippen und mit Enter bestätigen.

**Jetzt, ganz wichtig: SSH-Verbindung trennen und anschließend neu verbinden!**\
Nur so wird die Umstellung des Standardeditors auf nano aktiv. Danach weiter mit Punkt 5.

## 6. Voraussetzung: Ein Mailaccount mit korrekter Ordnerstruktur

Zum Testen dieses Tutorials empfehle ich, eine bereits vorhandene E-Mail-Adresse nicht anzutasten. Stattdessen erstmal eine Test-E-Mail-Adresse einrichten. Es ist problemlos möglich, die
Spamfilterung bei Gefallen auch auf alle weiteren gewünschten E-Mail-Adressen auf dem selben U7 auszuweiten.

**Hier ein weiteres mal der Verweis auf:** ***"3.2 Die Platzhalter-Variable [!USERNAME!]"***\
Ab hier könntest du `[!USERNAME!]` z.B. konsequent durch `spamfiltertest` ersetzen. 

### 6.1 Neuen Mailaccount anlegen
Also bitte einen neuen Mailaccount auf dem U7 anlegen, dafür den Befehl `uberspace mail user add [!USERNAME!]` nutzen. (siehe [U7-Manual > 'Mailboxes' > 'Main mailbox'](https://manual.uberspace.de/mail-mailboxes.html))

### 6.2 Benötigte Ordnerstruktur
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
Weitere individuelle Ordner in der INBOX sind natürlich problemlos möglich. Die 0 zu Beginn des Ordners Spamfilter sorgt dafür, dass der Ordner in den Ordnerlisten oben steht.

### 6.3 Diese geforderte Ordnerstruktur einrichten
                 
Die geforderte Ordnerstruktur lässt sich mit folgenden Befehlen komfortabel einrichten.

Zuerst eine Umgebungsvariable mit dem Benutzernamen füllen, unter dem der Mailaccount angelegt wurde. Dazu folgendes, entsprechend angepasst, in die Shell eingeben:
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

Prüfe nun in deinem Mailclient, ob die erstellten Ordner angezeigt werden!\
Sollte dies nicht der Fall sein, ist es nötig, die neu angelegten Ordner in den Einstellungen manuell sichtbar zu machen bzw. zu abonnieren, damit sie im Mailclient auftauchen!\
Im [RainLoop-Webmail-Client von Uberspace7](https://webmail.uberspace.de/) blendet man Ordner wie folgt ein: `Settings > Folders` aufrufen und über einen Klick auf die jeweiligen Augen-Symbole einblenden.

## 7. Installation & Einrichtung CRM114

Die folgenden Befehle installieren und konfigurieren CRM114 in den Ordner *~/crm114*.\
Installiert werden: [CRM114](http://crm114.sourceforge.net) und [TRE](https://laurikari.net/tre/) ('The free and portable approximate regex matching library' von Laurikari) sowie Konfigurationsdateien und Scripte hier aus diesem Repository.

**Achtung**: Sofern der Ordner ~/crm114 bereits existiert wird er durch folgende Befehle ohne Rückfrage überschrieben.

Der folgende Codeblock enthält keine [!USERNAME!]-Variable. Demnach ist es möglich, die folgenden Shell-Befehle komplett in einem Block via Copy & Paste auszuführen, sofern dein SSH-Client das unterstützt (putty kann das).

```Shell
mkdir -p ~/crm114       # Erzeuge Ordner 'crm114'
cd ~/crm114             # Wechsle in den Ordner 'crm114'
curl -sSL http://crm114.sourceforge.net/tarballs/crm114-20100106-BlameMichelson.src.tar.gz | tar xz # Kopiere und entpacke $ CRM114
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

## 8. Anlernen und Aufräumen via Cronjob automatisieren

Das Script 'learn_maildir' geht bei Aufruf immer alle Mailaccounts des Uberspace durch und prüft, ob in dem Mailaccount die beiden Ordner "als Spam lernen" und "als Ham lernen" vorhanden sind. Sind diese Ordner vorhanden, so prüft das Script, ob in den Ordnern Mails vorhanden sind und zeigt diese CRM114 entweder als Spam oder als Ham. So lernt CRM114 mit der Zeit, deine Mails zuverlässig einzuordnen. Nach dem Anlernen werden die E-Mails aus den beiden Ordnern gelöscht. Dieses Script sollte regelmäßig automatisch ausgeführt werden. Deshalb wird im folgenden ein Cronjob angelegt, der das Script 'learn_maildir' alle 20 Minuten automatisch aufruft.

Das Script 'cache_cleanup' sorgt dafür, dass die Cache-Dateien von CRM114 regelmäßig entschlackt werden. Auch dafür wird im folgenden ein Cronjob angelegt, der regelmäßig das Script cache_cleanup aufruft.

Zuerst die Crontab zum Bearbeiten aufrufen mit folgendem Shell-Befehl:
```Shell
crontab -e
```
und dort die folgende zwei Zeilen am Ende der Datei ergänzen:
```
*/20 * * * * sleep $((RANDOM \% 40 + 10)); crm114/learn_maildir
32 4 * * 0,3 crm114/cache_cleanup
```
Nun speichern und schließen, wie gewohnt mit: Strg+X, `y`, Enter.

## 9. Spamerkennung für einen Mailaccount einrichten

**Wichtig: Die folgenden Schritte sollten so nur ausgeführt werden, wenn die .qmail-Datei und die .mailfilter-Datei bisher noch nicht vorhanden sind.**

### 9.1 .mailfilter-Datei erstellen
Zuerst folgende Befehle, **immer jeweils angepasst**, in der Shell ausführen.
```Shell
cd ~
touch .mailfilter_[!USERNAME!]        # Erstellt die Datei, aber nur, sofern sie noch nicht existiert
chmod 600 .mailfilter_[!USERNAME!]    # Ändert auf die von U7 benötigte Rechte-Einstellung
nano .mailfilter_[!USERNAME!]         # Öffnet die Datei zum Bearbeiten
```
Durch den nano-Befehl öffnete sich die Texteingabe für die Datei.

Den folgenden Text in die nun geöffnete Datei eingeben:\
(Copy & Paste ist möglich. Auch hier das Anpassen in der ersten Zeile nicht vergessen!)
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
Nun speichern und schließen, wie gewohnt mit: Strg+X, `y`, Enter.

### 9.2 .qmail-Datei erstellen 

Folgende Befehle, **immer jeweils angepasst**, in der Shell ausführen.

```Shell
cd ~
touch .qmail-[!USERNAME!]       # Erstellt die Datei, aber nur, sofern sie noch nicht existiert
chmod 644 .qmail-[!USERNAME!]   # Ändert auf die von U7 benötigte Rechte-Einstellung
nano .qmail-[!USERNAME!]        # Öffnet die Datei zum Bearbeiten
```
Durch den nano-Befehl öffnete sich die Texteingabe für die Datei.

Den folgenden Text in den nun offenen nano-Editor eingeben:\
(Copy & Paste ist möglich. Auch hier das Anpassen der Zeile nicht vergessen!)
```
|maildrop $HOME/.mailfilter_[!USERNAME!]
```
Speichern und schließen, wie gewohnt mit: Strg+X, `y`, Enter.

## 10. OPTIONAL: Produktive E-Mail-Adresse in den Test einbeziehen

Es ist möglich, E-Mails der produktiven E-Mail-Adresse auf regulärem Weg (ohne Spamfilterung) zuzustellen und **zusätzlich** auf die Test-E-Mail-Adresse mit Spamfilterung zuzustellen. So kann das Verhalten des Spamfilters bei real ankommenden Mails getestet werden ohne in Abläufe der produktiven E-Mail-Adresse einzugreifen.

Um dies zu erreichen die **vorhandene** .qmail-Datei der produktiven E-Mail-Adresse um die Zustellung an die .mailfilter-Datei **ergänzen**.

Die .qmail-Datei, z.B. `.qmail-meinehauptmailadresse`,  könnte dann so aussehen:
```
./users/meinehauptmailadresse/              # Wie bisher und unverändert. Die E-Mail wird regulär an den Mailaccount zugestellt
|maildrop $HOME/.mailfilter_[!USERNAME!]    # Zusätzlich wird die Mail via Mailfilter an den Test-Account zugestellt
```

Nun kannst du im Test-E-Mail-Account risikofrei testen und dort CRM114 trainieren. Dieses Training hilft dir auch bereits für die Zukunft, denn nach der Umstellung des Spamfilters auf produktive E-Mail-Adressen bleiben die antrainierten Regeln erhalten. CRM114 legt für die Spamerkennung eine globale Erkennungsdatenbank an, die für alle einbundenen E-Mail-Adressen eines Uberspace gleichzeitig gültig ist. In diesem Fall ist das eine nützliche Sache, aber es mag auch Anwendungsfälle geben, in denen dieses Verhalten unerwünscht ist. Daher ist es sinnvoll, sich das bewusst zu machen.

## 11. Test beenden und Spamfilter für die produktive E-Mail-Adresse einrichten
Sofern du mit dem Testlauf zufrieden bist, kannst du später folgende Schritte durchführen, um CRM114 für deine eigentliche E-Mail-Adresse zu aktivieren:
- Die geforderte Ordnerstruktur in der produktiven E-Mail-Adresse erstellen.
- Eine eigene Mailfilter-Datei für die produktive E-Mail-Adresse erstellen und anpassen oder, wenn bereits vorhanden, die vorhandene Mailfilter-Datei entsprechend anpassen.
- Die vorhandene .qmail-Datei der produktiven E-Mail-Adresse so einstellen, dass Mails nicht mehr direkt in den Mailaccount zugestellt werden, sondern stattdessen über die zugehörige Mailfilter-Datei verarbeitet werden. Das könnte z.B. wie folgt aussehen, für die Datei `.qmail-meinehauptmailadresse`: 
```
#./users/meinehauptmailadresse/                     # Die Zeile auskommentieren, so dass keine E-Mails direkt ohne Spamfilter zugestellt werden
|maildrop $HOME/.mailfilter_meinehauptmailadresse   # Die Mail wird an die neue .mailfilter-Datei übergeben
```
## 12. CRM114 trainieren, Spam und Ham zu erkennen
CRM114 lernt, Spam und Ham zu erkennen, indem du ihm zu Anfang für eingehende Mails zeigst, was Spam und Ham für dich ist. Dazu eingehende E-Mails immer wie folgt markierenentweder als Spam (Spammails aus dem Posteingang in den Ordner "als Spam lernen" verschieben) oder Ham (Hammails aus dem Posteingang in den Ordner "als Ham lernen" kopieren) präsentieren.

 -- To be continued

## 13. Credits

Externe Quellen, die für dieses Tutorial bzw. seine Durchführung herangezogen werden:
- CRM114 unter GPLv2-Lizenz
http://crm114.sourceforge.net/
- TRE (The free and portable approximate regex matching library) unter BSD-Lizenz von Ville Laurikari
https://laurikari.net/tre/ & https://github.com/laurikari/tre/
- Dieses Tutorial, Texte, Quellcode und Scripte basieren auf einem (inzwischen gelöschten) Blogbeitrag von Bernhard Ehlers. Er hat es mir gestattet, diese als Vorlagen für eine erneute Veröffentlichung zu verwenden.
