# Lernenden Spamfilter auf Uberspace7 selbst betreiben 
Vorweg: **Ein Dank an Bernhard Ehlers!** Diese Anleitung basiert auf seinem Blogtext und seinen Config-Dateien. Er gab mir die Erlaubnis, sie als Grundlage zu verwenden.

# DIESES TUTORIAL BEFINDET SICH NOCH IM AUFBAU!

I   - Grundsätzliches und Vorbereitungen\
II  - Installation der Spamfilter-Software\
III - Die Lernfunktionen aktivieren\
IV  - Notwendige Ordner im Mailaccount anlegen\
V   - Aktivierung der automatischen Spamprüfung für einen Mailaccount\
VI  - Anleitung zur Nutzung und zum Anlernen\
VII - Sonstiges

# I - Grundsätzliches und Vorbereitungen

## 1. Hintergrund
Stand heute bietet der Hosting-Anbieter [uberspace](https://www.uberspace.de) auf seinem Produkt [Uberspace7](https://blog.uberspace.de/tag/uberspace7/) (abgekürzt U7) von Haus aus keinen *lernenden* bzw. *trainierbaren* Spamfilter an. Bisher existiert unter U7 vorimplementiert nur folgender, sehr rudimentärer, Spamschutz: Jede eingehende E-Mail wird automatisch mit einem Rspamd-Score versehen und bei einem Wert größer 15 sofort gelöscht (siehe [U7-Manual > 'Filtering mails'](https://manual.uberspace.de/mail-filter.html)). Grundsätzlich wäre es möglich, via qmail und mailfilter, auch für niedrigere Rspamd-Scores eigene Verarbeitungsschritte einzubauen und so die Schwelle für Spam/Ham den eigenen Bedürfnissen nach festzulegen. Um aber Spam wirklich effektiv ausfiltern zu können, benötigt man einen trainierbaren Spamfilter, der in vielen Software-Mailclients wie z.B. Thunderbird häufig fester Bestandteil ist. Wer über mehrere Geräte hinweg auf E-Mails zugreift wird aber nicht auf einen serverseitigen lernenden Spamfilter verzichten wollen. Und bis die Ubernauten einen solchen Filter auf U7 anbieten, hilft diese Anleitung dabei, sich selbst einen solchen Spamfilter einzurichten.

Noch der Hinweis, dass es auf dem Vorgängerprodukt U6 vorinstallierte Spamfilter-Software gibt (SpamAssassin (mit integriertem Regelwerk) und DSPAM (trainierbar)), die man selbst einrichten muss/kann. Auf U7 werden diese Spamfilter-Programme aber nicht mehr von Vornherein angeboten, da sie unmaintained sind. Diese Anleitung imitiert in etwa das von U6 gewohnte Verhalten von DSPAM, indem es CRM114 auf U7 nutzt. Allen, die das Spamfiltern und Anlernen von U6 kennen, sollten sich hier also schnell zurechtfinden.

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

>**Beispiel**:\
>Du entscheidest dich für den Mailbenutzer `spamfiltertest`. In der Anleitung steht: Bitte `uberspace mail user add [!USERNAME!]` eingeben. Die Eingabe muss dann lauten: `uberspace mail user add spamfiltertest`

## 4. nano zum Standardeditor machen

Sofern nicht eh schon geschehen, bitte jetzt nano als Standardeditor festlegen. So kann der Rest der Anleitung so durchgegangen werden, wie beschrieben. Wenn gewünscht, kann dieser Schritt nach Beenden dieses Tutorials wieder rückgängig gemacht werden.

Um nano zum Standardeditor zu machen, bitte folgende Befehle in der Shell ausführen:
```Shell
nano ~/.bash_profile
```
Durch den nano-Befehl öffnet sich die Datei .bash_profile

Dort drin eine weitere Zeile mit folgendem Text unten anfügen, sofern diese Zeile noch nicht existiert.
```
export VISUAL='nano'
```
Nun speichern und schließen.
Mit nano geht das immer so: Tastenkombination Strg+X drücken, dann ein `y` eintippen und mit Enter bestätigen.

Mit folgendem Shell-Befehl die Änderung schon für die aktuelle Session aktivieren:
```Shell
source ~/.bash_profile
```
Nano ist ab jetzt und bei jedem neuen SSH-Verbindungsaufbau der Standardeditor. 

# II  - Installation der Spamfilter-Software

## 5. CRM114 installieren

Die folgenden Befehle installieren und konfigurieren CRM114 in den Ordner *~/crm114*.\
Installiert werden: [CRM114](http://crm114.sourceforge.net) und [TRE](https://laurikari.net/tre/) ('The free and portable approximate regex matching library' von Laurikari) sowie Konfigurationsdateien und Scripte hier aus diesem Repository.

**Achtung**: Sofern der Ordner ~/crm114 bereits existiert wird er durch folgende Befehle ohne Rückfrage überschrieben.

### 5.1 Automatische Installation

Nur folgende Zeile in der Shell ausführen:
```Shell
curl -sSL https://raw.githubusercontent.com/flowdx/crm114_u7/master/install_crm114_u7.sh -o ~/install_crm114_u7.sh && chmod +x ~/install_crm114_u7.sh && ~/install_crm114_u7.sh && rm ~/install_crm114_u7.sh
```

### 5.2 ODER: Manuelle Installation

**Alternativ zur automatischen Installation** können die folgenden Zeilen manuell inder Shell ausgeführt werden. Diese entsprechen den Schritten in der install_crm114_u7.sh aus der Automatischen Installation:

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
## 6. CRM114 auf Funktionsfähigkeit testen

Ob die Installation erfolgreich war und CRM114 funktioniert, lässt sich in der Shell mit folgendem Befehl testen:

```Shell
~/crm114/learn_maildir -v
```

Hier wird das Script 'learn_maildir' im Ordner crm114 manuell ausgeführt, und zwar mit der -v Option (v steht hier für "verbose", also "geschwätzig"). Dieses Script prüft auf neue Ham- und Spammails, die ihm zum Anlernen übergeben werden. Zu diesem Zeitpunkt sollte es aber nur die vorhandenen Mailaccounts deines Uberspace auflisten und keine Spammails oder Hammails zum Anlernen finden. Diesen Shell-Befehl kannst du später immer wieder ausführen, wenn du sehen möchtest, was learn_maildir bei Ausführung tut.

# III - Die Lernfunktionen aktivieren

## 7. Anlernen und Aufräumen via Cronjob automatisieren

Das Script 'learn_maildir' (ab hier nicht mehr im Verbose-mode) geht bei Aufruf immer alle Mailaccounts des Uberspace durch und prüft, ob in dem Mailaccount die beiden Ordner "als Spam lernen" und "als Ham lernen" vorhanden sind. Sind diese Ordner vorhanden, so prüft das Script, ob in den Ordnern E-Mails vorhanden sind und zeigt diese CRM114 entweder als Spam oder als Ham. So lernt CRM114 mit der Zeit, E-Mails zuverlässig einzuordnen. Nach dem Anlernen werden die E-Mails aus den beiden Ordnern gelöscht. Dieses Script sollte regelmäßig automatisch ausgeführt werden. Deshalb wird im folgenden ein Cronjob angelegt, der das Script 'learn_maildir' alle 20 Minuten automatisch aufruft.

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
# IV  - Notwendige Ordner im Mailaccount anlegen
## 8. OPTIONAL: Test-E-Mail-Account erstellen

An dieser Stelle musst du eine Entscheidung treffen: Entweder du richtest den Spamfilter für einen bestehenden Mailaccount ein oder du erstellst zu Testzwecken erstmal einen neuen Test-Mailaccount. Sofern du dich mit .qmail- und .mailfilter-Dateien bisher noch nicht auskennst empfehle ich, vorerst einen Test-Mailaccount zu erstellen und deine produktiven Mailaccounts erst anzutasten, sofern du sicher bist, zu wissen, was du tust. Es ist problemlos möglich, die Spamfilterung dann später auf beliebig viele weitere Mailaccounts auf dem selben U7 auszuweiten.

**Hier ein weiteres mal der Verweis auf:** ***"3.2 Die Platzhalter-Variable [!USERNAME!]"***\
Ab hier könntest du `[!USERNAME!]` z.B. konsequent durch `spamfiltertest` ersetzen. 

Also bitte einen neuen Mailaccount auf dem U7 anlegen, dafür den Befehl `uberspace mail user add [!USERNAME!]` nutzen. (siehe [U7-Manual > 'Mailboxes' > 'Main mailbox'](https://manual.uberspace.de/mail-mailboxes.html))

## 9. Notwendige Ordnerstruktur im Mailaccount erstellen
In jedem Mailaccount, der Spamschutz von CRM114 erhalten soll, muss folgende Ordnerstruktur vorhanden sein:
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

### 9.1 Diese geforderte Ordnerstruktur einrichten
                 
Die geforderte Ordnerstruktur lässt sich mit folgenden Befehlen komfortabel einrichten.

Zuerst eine Umgebungsvariable mit dem Benutzernamen füllen, unter dem der Mailaccount angelegt wurde. Dazu folgendes, entsprechend angepasst, in die Shell eingeben (für dieses Tutorial z.B. den oben angelegten Test-Mailaccount):
```Shell
export MAILUSERNAME=[!USERNAME!]
```

Anschließend die folgenden vier Befehle in der Shell ausführen um die Ordner in dem Mailaccount anzulegen:
(Die Ordner werden nur angelegt, sofern sie noch nicht existieren.)
```Shell
test -d "$HOME/users/$MAILUSERNAME/.0 Spamfilter" || maildirmake "$HOME/users/$MAILUSERNAME/.0 Spamfilter"                
test -d "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Ham lernen" || maildirmake "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Ham lernen"
test -d "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Spam lernen" || maildirmake "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Spam lernen"
test -d "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Spam erkannt" || maildirmake "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Spam erkannt"
```

Prüfe nun in deinem Mailclient, ob die erstellten Ordner angezeigt werden!\
Sollte dies nicht der Fall sein, ist es nötig, die neu angelegten Ordner in den Einstellungen manuell sichtbar zu machen bzw. zu abonnieren, damit sie im Mailclient auftauchen!\
Im [RainLoop-Webmail-Client von Uberspace7](https://webmail.uberspace.de/) blendet man Ordner wie folgt ein: `Settings > Folders` aufrufen und über einen Klick auf die jeweiligen Augen-Symbole einblenden.

# V   - Aktivierung der Spam/Ham-Prüfung bei eingehenden Mails für einen Mailaccount

## 10. Spamerkennung für einen Mailaccount einrichten

**Wichtig: Die folgenden Schritte sollten so nur für den Test-Mailaccount ausgeführt werden. Oder anders: Nur ausführen, sofern die .qmail-Datei und die .mailfilter-Datei bisher noch nicht vorhanden sind.**

### 10.1 .mailfilter-Datei erstellen
Zuerst folgende Befehle, **immer jeweils angepasst**, in der Shell ausführen.
```Shell
touch ~/.mailfilter_[!USERNAME!]        # Erstellt die Datei, aber nur, sofern sie noch nicht existiert
chmod 600 ~/.mailfilter_[!USERNAME!]    # Ändert auf die von U7 benötigte Rechte-Einstellung
nano ~/.mailfilter_[!USERNAME!]         # Öffnet die Datei zum Bearbeiten
```
Durch den nano-Befehl öffnete sich die Texteingabe für die Datei.

Den folgenden Text in die nun geöffnete Datei eingeben:\
(Copy & Paste ist möglich. Auch hier das Anpassen in der ersten Zeile nicht vergessen!)\
**Wichtig:** Sofern bereits eine .mailfilter-Datei für den Mailaccount existiert, kann das folgende nicht einfach übernommen werden. Stattdessen muss es in die vorhandenden Regelungen ggf. angepasst eingebaut werden.
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

### 10.2 .qmail-Datei erstellen 

Folgende Befehle, **immer jeweils angepasst**, in der Shell ausführen.

```Shell
touch ~/.qmail-[!USERNAME!]       # Erstellt die Datei, aber nur, sofern sie noch nicht existiert
chmod 644 ~/.qmail-[!USERNAME!]   # Ändert auf die von U7 benötigte Rechte-Einstellung
nano ~/.qmail-[!USERNAME!]        # Öffnet die Datei zum Bearbeiten
```
Durch den nano-Befehl öffnete sich die Texteingabe für die Datei.

Den folgenden Text in den nun offenen nano-Editor eingeben:\
(Copy & Paste ist möglich. Auch hier das Anpassen der Zeile nicht vergessen!)\
**Wichtig:** Sofern bereits eine .qmail-Datei für den Mailaccount existiert, kann das folgende nicht einfach übernommen werden. Stattdessen muss es in die vorhandenden Regelungen ggf. angepasst eingebaut werden.
```
|maildrop $HOME/.mailfilter_[!USERNAME!]
```
Speichern und schließen, wie gewohnt mit: Strg+X, `y`, Enter.

## 11. OPTIONAL: Ankommende E-Mails des produktiven Mailaccounts in Kopie an den Test-Mailaccount weiterleiten

Sofern du dich für einen Test-Mailaccount entschieden hast ist es sinnvoll, ankommende E-Mails deiner bereits vorhandenen, produktiven E-Mail-Adresse wie bisher auf regulärem Weg (ohne Spamfilterung) zuzustellen und **zusätzlich** auf die Test-E-Mail-Adresse mit Spamfilterung zuzustellen. So kann das Verhalten des Spamfilters bei real ankommenden Mails getestet werden, ohne vorerst in die gewohnten Abläufe der produktiven E-Mail-Adresse einzugreifen.

Um dies zu erreichen die **vorhandene** .qmail-Datei der produktiven E-Mail-Adresse um die Zustellung an die .mailfilter-Datei **ergänzen**.

Die .qmail-Datei, z.B. `.qmail-meinehauptmailadresse`,  könnte dann so aussehen:
```
./users/meinehauptmailadresse/              # Wie bisher und unverändert. Die E-Mail wird regulär an den Mailaccount zugestellt
|maildrop $HOME/.mailfilter_[!USERNAME!]    # Zusätzlich wird die Mail via Mailfilter an den Test-Account zugestellt
```

Nun kannst du im Test-E-Mail-Account risikofrei testen und dort CRM114 trainieren. Dieses Training hilft dir auch bereits für die Zukunft, denn nach der Umstellung des Spamfilters auf weitere E-Mail-Accounts bleiben die antrainierten Regeln erhalten. CRM114 legt für die Spamerkennung eine globale Erkennungsdatenbank an, die für alle eingebundenen E-Mail-Adressen eines Uberspace gleichzeitig gültig ist. In diesem Fall ist das eine nützliche Sache, aber es mag auch Anwendungsfälle geben, in denen dieses Verhalten unerwünscht ist. Daher ist es sinnvoll, sich das bewusst zu machen.
# VI  - Anleitung zur Nutzung und zum Anlernen
## 12. CRM114 testen, nutzen und trainieren.
Jede eingehende E-Mail wird vom CRM114 einer Prüfung unterzogen und bekommt eine von drei möglichen Kennzeichnungen: Ham, Spam oder Unsure (also Unsicher).\
Je nach Kennzeichnung verfährt CRM114 mit der E-Mail unterschiedlich wie folgt:
- **Ham**: Die E-Mail wird im Posteingang abgelegt.
- **Spam**: Die E-Mail wird im Ordner `0 Spamfilter/als Spam erkannt` abgelegt.
- **Unsure**: Die E-Mail wird im Posteingang abgelegt und der Betreff wird um `[UNSURE]` ergänzt.

Den Ordner `0 Spamfilter/als Spam erkannt` musst du regelmäßig prüfen!\
Hier sammeln sich die E-Mails an, die CRM114 als Spam erkennt. E-Mails in diesem Ordner, die auch wirklich Spam sind, kannst du einfach löschen und musst sie CRM114 nicht erneut zum Anlernen vorlegen. Findest du in diesem Ordner allerdings eine E-Mail, die eigentlich kein Spam ist, dann ist Sorgfalt geboten! Die folgenden Erklärungen sind dann besonders wichtig, insbesondere Beispiel 3!

Zu Anfang wird CRM114 viele eingehende E-Mails mit einem [UNSURE] im Betreff versehen und nur wenige E-Mails im Spam-Erkannt-Ordner ablegen. CRM114 muss lernen, die E-Mails in deinem Sinne einzuschätzen. Dazu musst du CRM114 zeigen, ob eingehende E-Mails aus deiner Sicht Spam oder Ham darstellen. Es ist auch möglich (und sinvoll), CRM114 zu korrigieren, wenn es eine eingehende E-Mail falsch eingeschätzt hat.

Das Anlernen erfolgt über die beiden Ordner `0 Spamfilter/als Spam lernen` und `0 Spamfilter/als Ham lernen`, mit deren Hilfe du CRM114 E-Mails zum Lernen vorlegen kannst.

**GANZ GANZ WICHTIG: E-Mails, die du in die Lernordner speicherst, werden nach dem Anlernen durch CRM114 !! unwiederbringlich gelöscht !!. Es besteht also das Risiko, E-Mails unwiederbringlich zu verlieren, wenn nicht sorgfältig mit der Lernfunktion umgegangen wird. Aus diesem Grund müssen die folgenden Regeln verstanden und eisern beachtet werden:**
- **!!** Mails, die als **Ham** angelernt werden sollen, in den Ordner `0 Spamfilter/als Ham lernen` **immer KOPIEREN !!**
- Mails, die als **Spam** angelernt werden und danach gelöscht werden sollen, kann man in den Ordner `0 Spamfilter/als Spam lernen` verschieben.

>**Beispiel 1**:\
>Eine E-Mail landet in deinem Posteingang und wurde im Betreff mit [UNSURE] gekennzeichnet. Diese E-Mail ist Ham und du möchtest, dass solche E-Mails zukünftig von CRM114 auch als Ham erkannt werden (also kein [UNSURE] mehr bekommen). Diese E-Mail **kopierst** du in den Ordner `0 Spamfilter/als Ham lernen` und belässt die originale E-Mail in deinem Posteingang (oder sortierst sie nach deinem Bedarf in einen anderen Ordner). Würdest du die E-Mail in den Ham-Lernordner verschieben, so wäre sie nach dem Anlernen für immer gelöscht.

>**Beispiel 2**:\
>Eine E-Mail landet in deinem Posteingang (ganz gleich ob mit oder ohne [UNSURE]). Diese E-Mail ist aber Spam, und du möchtest, dass solche E-Mails zukünftig von CRM114 auch als Spam erkannt werden. Da du diese E-Mail nach dem Anlernen auch garnicht mehr gespeichert wissen willst, **verschiebst** du diese E-Mail in den Ordner `0 Spamfilter/als Spam lernen`. Nach dem Anlernen durch CRM114 ist diese E-Mail dann gelöscht.

>**Beispiel 3**:\
>Eine E-Mail wird von CRM114 beim Eingang als Spam erkannt und aus diesem Grund findest du sie nicht im Posteingang sondern im Ordner `0 Spamfilter/als Spam erkannt`. Du stellst fest, dass diese E-Mail fälschlicherweise als Spam erkannt wurde und sie für dich eigentlich Ham darstellt. Nun gehst du wie folgt vor: Zuerst **kopierst** du die E-Mail in den Posteingang, um sie dort zu behalten. Anschließend **verschiebst** du die E-Mail aus dem Spam-Erkannt-Ordner in den Ham-Lernordner, damit CRM114 lernt, solche E-Mails in Zukunft als Ham anstelle von Spam einzustufen. Die E-Mail wird nach dem Anlernen aus dem Ham-Lernordner gelöscht sein, aber im Posteingang ist sie noch vorhanden.


Dies zeigt dir an, dass CRM114 noch nicht weiß, wie es mit dieser E-Mail umgehen soll. Dazu eingehende E-Mails immer wie folgt markierenentweder als Spam (Spammails aus dem Posteingang in den Ordner "als Spam lernen" verschieben) oder Ham (Hammails aus dem Posteingang in den Ordner "als Ham lernen" kopieren) präsentieren.

-> Hier fehlt derzeit noch einiges an Text

## 13. Test beenden und Spamfilter für die produktive E-Mail-Adresse einrichten
Sofern du mit dem Testlauf zufrieden bist, kannst du später folgende Schritte durchführen, um CRM114 für deine eigentliche E-Mail-Adresse zu aktivieren:
- Die geforderte Ordnerstruktur in der produktiven E-Mail-Adresse erstellen (siehe 9.1).
- Eine eigene Mailfilter-Datei für die produktive E-Mail-Adresse erstellen und anpassen oder, wenn bereits vorhanden, die vorhandene Mailfilter-Datei entsprechend anpassen.
- Die vorhandene .qmail-Datei der produktiven E-Mail-Adresse so einstellen, dass Mails nicht mehr direkt in den Mailaccount zugestellt werden, sondern stattdessen über die zugehörige Mailfilter-Datei verarbeitet werden. Das könnte z.B. wie folgt aussehen, für die Datei `.qmail-meinehauptmailadresse`: 
```
#./users/meinehauptmailadresse/                     # Die Zeile auskommentieren, so dass keine E-Mails direkt ohne Spamfilter zugestellt werden
|maildrop $HOME/.mailfilter_meinehauptmailadresse   # Die Mail wird an die neue .mailfilter-Datei übergeben
```

# VII - Sonstiges
## 14. Tipps

-> noch nicht vorhanden

## 15. Einschränkungen & deren Lösung

**Derzeit ist keine Einschränkung bekannt, für die es keine implementierte Lösung gibt.**

### 15.1 CRM114 hat Schwierigkeiten beim Anlernen "großer" E-Mails (Workaround implementiert)
**Problembeschreibung:** Eine E-Mail, die zum Anlernen in einem Ordner liegt und die in Summe größer als 3,9Mb ist, bringt CRM114 auf Uberspache (und wohl generell) zu einem kompletten Abbruch der Ausführung des Anlernvorgangs. Jede Ausführung des Anlernens scheitert, so lange sich diese E-Mail in einem Anlernordner befindet. Nach dem Löschen dieser E-Mail aus dem Anlernordner funktioniert die Erkennung wieder reibungslos. Ursache ist ein cachebezogenes Problem der Komponenten von CRM114, für das ich keinen Fix gefunden haben. (Weitere Infos [hier](https://sourceforge.net/p/crm114/mailman/message/22532062/) und [hier](https://sourceforge.net/p/crm114/mailman/message/28066834/)). CRM114 erwartet wohl, dass ihm E-Mails immer frei von Anhängen bzw. gekürzt auf eine passable Größe übergeben werden. Aber auch wenn Spammer selten E-Mails mit großen Dateianhängen verschicken, so kann es doch gewünscht und sinnvoll sein, CRM114 auch E-Mails zum Anlernen vorzulegen, die große Dateianhänge haben, insbesondere bei Ham.\
\
**Implementierte Lösung: Zwei Varianten zur Fehlervermeidung sind implementiert:**\
Genutzt wird immer **entweder** Variante 1 **oder** Variante 2. Ein Wechsel zwischen beiden Varianten ist jederzeit möglich, auch im laufenden Betrieb.

- **Variante 1** (Als **Standard aktiv** und sehr sicher funktionstüchtig, aber mit Einschränkungen verbunden)\
Dies ist die radikale Variante, die aber garantiert immer funktioniert: E-Mails in Anlernordnern, die größer als 3,9Mb sind, werden vor dem Anlernen gelöscht und somit beim Anlernen nicht berücksichtigt. Nachteil: E-Mails, die größer als 3,9Mb sind, lassen sich nicht nutzen, um den Filter zu trainieren. Da man dies nicht mitbekommt verhält sich diese Variante also im Fall großer E-Mails nicht so, wie man den Anlernvorgang eigentlich erwartet.
- **Variante 2** (experimentell, muss manuell aktiviert werden)\
Die elegante Variante, von der aber nicht ganz klar ist, ob sich diese immer problemlos verhält: E-Mails in Anlernordnern, die größer als 3,9Mb sind, werden vor dem Anlernen auf 3,9Mb "gekürzt" (genutzt wird truncate) und werden dann regulär von CRM114 angelernt. In den ersten 3,9Mb einer E-Mail sollten in der Regel alle Informationen vorhanden sein, die CRM114 benötigt, um die E-Mail anzulernen. Inhalte von Dateianhängen sind für eine Spamerkennung eher irrelevant.

**Umschalten der genutzten Variante (ACHTUNG, DIES GESCHIEHT AUF EIGENE VERANTWORTUNG!)**\
In der Datei 'learn_maildir' die Variable CACHEWORKAROUND in Zeile 61 entweder mit 1 oder 2 füllen und die Datei anschließend speichern. Die Variable ist vorbelegt mit einer 1, Standard ist also die Nutzung der Variante 1. Ein Wechsel zwischen beiden Varianten ist jederzeit möglich.

**Ändern der Mailgröße, ab der die Varianten greifen sollen (ACHTUNG, DIES GESCHIEHT AUF EIGENE VERANTWORTUNG!)**\
Es ist möglich, das Limit von 3,9Mb abzuändern. Ich rate aber ausdrücklich davon ab, denn dies kann zu den oben benannten Fehlern beim Anlernvorgang führen. Ändern geht wie folgt: In der Datei 'learn_maildir' die Variable MAILSIZELIMITkb in Zeile 59 mit der gewünschten Kilobytemenge befüllen. Standard ist: 3915

### 15.2 Die Spamerkennung eingehender "großer" E-Mails verzögert die Zustellung (Workaround implementiert)
**Implementierte Lösung:** Eingehende E-Mails, die größer als 2MB sind, werden nicht an CRM114 zur Spamrüfung übergeben. Dies geschieht, um die Last für CRM114 gering zu halten und weil "echte" Spammails selten größer sind als 2MB. Auf eigene Verantwortung kann dieser Wert in der .mailfilter-Datei abgeändert werden. 


## 16. Credits

Externe Quellen, die für dieses Tutorial bzw. seine Durchführung herangezogen werden:
- CRM114 unter GPLv2-Lizenz
http://crm114.sourceforge.net/
- TRE (The free and portable approximate regex matching library) unter BSD-Lizenz von Ville Laurikari
https://laurikari.net/tre/ & https://github.com/laurikari/tre/
- Dieses Tutorial, Texte, Quellcode und Scripte basieren auf einem (inzwischen gelöschten) Blogbeitrag von Bernhard Ehlers. Er hat es mir gestattet, diese als Vorlagen für eine erneute Veröffentlichung zu verwenden.
