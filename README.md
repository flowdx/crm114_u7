# Lernenden Spamfilter auf Uberspace7 selbst betreiben 
Vorweg: **Ein Dank an Bernhard Ehlers!** Diese Anleitung basiert auf seinem Blogtext und seinen Config-Dateien. Er gab mir die Erlaubnis, sie als Grundlage zu verwenden.

I   - Grundsätzliches und Vorbereitungen\
II  - Installation der Spamfilter-Software\
III - Die Lernfunktionen aktivieren\
IV  - Notwendige Ordner im Mailaccount anlegen\
V   - Aktivierung der Spam/Ham-Prüfung bei eingehenden Mails für einen Mailaccount\
VI  - Anleitung zur Nutzung und zum Anlernen\
VII - Sonstiges

# I - Grundsätzliches und Vorbereitungen

## 1. Hintergrund
Stand heute bietet der Hosting-Anbieter [uberspace](https://www.uberspace.de) auf seinem Produkt [Uberspace7](https://blog.uberspace.de/tag/uberspace7/) (abgekürzt U7) von Haus aus keinen *lernenden* bzw. *trainierbaren* Spamfilter an. Von Haus aus bieten die Ubernauten auf U7 derzeit "nur" folgenden basalen Spamschutz an: Jede eingehende E-Mail wird automatisch mit einem Rspamd-Score versehen und bei einem Wert größer 15 sofort gelöscht (siehe [U7-Manual > 'Filtering mails'](https://manual.uberspace.de/mail-filter.html)). Grundsätzlich wäre es möglich, via qmail und mailfilter, auch für niedrigere Rspamd-Scores eigene Verarbeitungsschritte einzubauen und so die Schwelle für Spam/Ham den eigenen Bedürfnissen nach festzulegen. Um aber Spam wirklich effektiv ausfiltern zu können benötigt man einen trainierbaren Spamfilter, der in vielen Software-Mailclients wie z.B. Thunderbird, häufig fester Bestandteil ist (also clientseitig). Alle, die über mehrere Geräte hinweg auf einen E-Mail-Account zugreifen, werden aber nicht auf einen zentralen (serverseitigen) lernenden Spamfilter verzichten wollen. Und bis die Ubernauten einen solchen Filter auf U7 anbieten, hilft diese Anleitung dabei, sich selbst einen solchen lernenden serverseitigen Spamfilter einzurichten.

Noch der Hinweis, dass es auf dem Vorgängerprodukt U6, welches inzwischen nicht mehr "buchbar" ist, eine vorinstallierte Spamfilter-Software gab. Dabei handelte es sich aber nicht um die im folgenden verwendete, sondern um SpamAssassin (mit integriertem Regelwerk) und DSPAM (trainierbar). Auf U7 werden diese Spamfilter-Programme aber nicht mehr von den Ubernauten angeboten, da sie unmaintained sind. Diese Anleitung imitiert in etwa das von U6 gewohnte Verhalten von DSPAM, indem es CRM114 auf U7 nutzt. Allen, die das Spamfiltern und Anlernen von U6 kennen, sollten sich hier also schnell zurechtfinden.

## 2. Spamfiltersoftware der Wahl: CRM114

Mit folgendem Zitat von Bernhard ist alles gesagt:
> "Nach etwas Recherche habe ich mich für [CRM114](http://crm114.sourceforge.net) entschieden. Es wurde zwar sehr lange nicht mehr aktualisiert, ist aber weiterhin in vielen Linux-Distributionen präsent. Der Lernalgorithmus ist sehr schnell und effizient, das Programm ist sehr klein und performant."

Der Rspamd-Score, den die Ubernauten für jede E-Mail automatisch bereitstellen (siehe [U7-Manual > 'Filtering mails'](https://manual.uberspace.de/mail-filter.html)), wird im Rahmen dieses Tutorials nicht verwendet. CRM114 verrichtet seine Arbeit so gut, dass dies nicht notwendig erscheint. Eine Kombination aus CRM114 und Verwendung von Rspamd-Score wäre natürlich auch möglich, wird hier aber nicht weiter beschrieben.

## 3. Grundlagen

### 3.1 Die Shell benutzen ;)

Alle Befehle hier werden via SSH-Zugriff in der Shell eingegeben. Darüber hinaus werden Dateien und Crontab mit nano bearbeitet.

### 3.2 Die Platzhalter-Variable [!USERNAME!]

In der folgenden Anleitung wird an mehreren Stellen die Platzhalter-Variable [!USERNAME!] verwendet. Diese ist **IMMER** durch einen selbstgewählten Benutzernamen zu ersetzen, und zwar unter Wegfall der eckigen Klammern und der Ausrufezeichen!

>**Beispiel**:\
>Du entscheidest dich für den Mailbenutzer `spamfiltertest`. In der Anleitung steht: Bitte `uberspace mail user add [!USERNAME!]` eingeben. Die Eingabe muss dann lauten: `uberspace mail user add spamfiltertest`

## 4. nano zum Standardeditor machen

Sofern nicht eh schon geschehen, bitte jetzt nano als Standardeditor festlegen. So kann der Rest der Anleitung exakt so durchgeführt werden wie beschrieben. Wenn gewünscht, kann dieser Schritt nach Beenden dieses Tutorials wieder rückgängig gemacht werden.

Um nano zum Standardeditor zu machen, bitte folgende Befehl in der Shell ausführen:
```Shell
nano ~/.bash_profile
```
Durch diesen Befehl öffnet sich die Datei .bash_profile im nano-Editor.

In die Datei eine weitere Zeile mit folgendem Text unten anfügen, sofern diese Zeile noch nicht existiert.
```
export VISUAL='nano'
```
Nun speichern und schließen.
Mit nano geht das immer so: Tastenkombination Strg+X drücken, dann ein `y` eintippen und mit Enter bestätigen.

Damit nano auch in der aktuell verbundenen SSH-Session bereits als Standard-Editor genutzt wird, den folgenden Shell-Befehl ausführen:
```Shell
source ~/.bash_profile
```
Nano ist ab jetzt, und bei jedem neuen SSH-Verbindungsaufbau, der Standardeditor. 

# II  - Installation der Spamfilter-Software

## 5. CRM114 installieren

Die folgenden Befehle installieren und konfigurieren CRM114 in den Ordner *~/crm114*.\
Installiert werden: [CRM114](http://crm114.sourceforge.net) und [TRE](https://laurikari.net/tre/) ('The free and portable approximate regex matching library' von Laurikari) sowie Konfigurationsdateien und Scripte hier aus diesem Repository.

**Achtung**: Sofern der Ordner ~/crm114 bereits existiert wird er durch folgende Befehle ohne Rückfrage überschrieben.

Es gibt zwei Optionen für die Installation, von der nur eine genutzt werden sollte, entweder 5.1 oder 5.2 !

### 5.1 Automatische Installation

Nur folgende Zeile in der Shell ausführen:
```Shell
curl -sSL https://raw.githubusercontent.com/flowdx/crm114_u7/master/install_crm114_u7.sh -o ~/install_crm114_u7.sh && chmod +x ~/install_crm114_u7.sh && ~/install_crm114_u7.sh && rm ~/install_crm114_u7.sh
```

### 5.2 ODER: Manuelle Installation

**Alternativ zur automatischen Installation** können die folgenden Zeilen manuell in der Shell ausgeführt werden. Diese entsprechen exakt den Schritten in der install_crm114_u7.sh, die bei der Automatischen Installation ausgeführt werden.

Der folgende Codeblock enthält keine [!USERNAME!]-Variable. Demnach ist es möglich, die folgenden Shell-Befehle komplett in einem Block via Copy & Paste auszuführen, sofern dein SSH-Client das unterstützt. (Der SSH-Client putty ist dazu in der Lage.)

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

Hier wird das Script 'learn_maildir' im Ordner crm114 manuell ausgeführt, und zwar mit der Option -v (Das v steht hier für "verbose", also "geschwätzig"). Bei diesem Script handelt es sich um jenes, das später immer die Lernordner auf neue E-Mails prüft, die zum Anlernen als Spam oder Ham übergeben werden. Zum jetzigen Zeitpunkt sollte der Aufruf des Scripts aber nur die vorhandenen Mailaccounts deines Uberspace auflisten und keine Spammails oder Hammails zum Anlernen finden, denn die beiden geforderten Lernordner existieren ja derzeit noch in keinem der Mailaccounts. Diesen Shell-Befehl kannst du auch später immer wieder ausführen, wenn du sehen möchtest, was learn_maildir bei Ausführung tut.

# III - Die Lernfunktionen aktivieren

## 7. Anlernen und Aufräumen via Cronjob automatisieren

Für das Anlernen und das Aufräumen des Caches von CRM114 gibt es zwei Scripte, die regelmäßig automatisch aufgerufen werden sollten. Automatische Vorgänge lassen sich auf einem uberspace immer gut mit der crontab-Datei realisieren.

Das Script 'learn_maildir' (ab hier nicht mehr im Verbose-mode) geht bei Aufruf immer alle Mailaccounts des Uberspace durch und prüft, ob in dem Mailaccount die beiden Ordner "0 Spamfilter/als Spam lernen" und "0 Spamfilter/als Ham lernen" vorhanden sind. Sind diese Ordner vorhanden, so prüft das Script, ob in den Ordnern E-Mails vorhanden sind und zeigt diese CRM114 entweder als Spam oder als Ham. So lernt CRM114 mit der Zeit, neu ankommende E-Mails zuverlässig als Spam oder Ham einzuordnen. Nach jedem Anlernvorgang werden die E-Mails aus den beiden Lernordnern gelöscht. Dieses Script sollte regelmäßig automatisch ausgeführt werden, deshalb wird im folgenden ein Cronjob angelegt, der das Script 'learn_maildir' alle 20 Minuten automatisch aufruft.

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
## 8. Den "richtigen" Mailaccount finden

Für den weiteren Verlauf des Tutorials wird angenommen, dass eine E-Mail-Adresse mit Spamschutz ausgestattet werden soll, die zuvor mit dem Befehl `uberspace mail user add [!USERNAME!]` angelegt wurde. (siehe [U7-Manual > 'Mailboxes' > 'Main mailbox'](https://manual.uberspace.de/mail-mailboxes.html))

**Hier ein weiteres mal der Verweis auf:** ***"3.2 Die Platzhalter-Variable [!USERNAME!]"***\
Ab hier muss die Platzhalter-Variable`[!USERNAME!]` konsequent durch den Benutzernamen ersetzt werden, der im uberspace-mail-user-add-Befehl (s.o.) verwendet wurde! 

(Alle, die zum jetzigen Zeitpunkt nicht direkt eine produktive E-Mail-Adresse mit Spamschutz ausstatten möchten könnten jetzt einen neuen Mailaccount zu Testzwecken einrichten und diesen verwenden.)

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
Weitere individuelle Ordner in der INBOX sind natürlich problemlos möglich. Die 0 zu Beginn des Ordners '0 Spamfilter' sorgt dafür, dass der Ordner in den Ordnerlisten immer oben steht.

### 9.1 Diese geforderte Ordnerstruktur einrichten
                 
Die geforderte Ordnerstruktur lässt sich mit folgenden Befehlen komfortabel einrichten:

Zuerst eine Umgebungsvariable (mit dem Namen MAILUSERNAME) mit dem Benutzernamen füllen, unter dem der Mailaccount angelegt wurde. Dazu folgenden Befehl, entsprechend angepasst, in die Shell eingeben:
```Shell
export MAILUSERNAME=[!USERNAME!]
```

Anschließend die folgenden vier Befehle in der Shell ausführen, um die Ordner in dem Mailaccount anzulegen:
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

Das Anlegen der benötigten Ordner kann auf diese Weise für mehrere Mailaccounts auf dem selben uberspace geschehen. Dazu diesen Abschnitt mehrfach durchführen: Immer zuerst die Umgebungsvariable entsprechend anpassen und anschließend die vier Befehle ausführen.

# V   - Aktivierung der Spam/Ham-Prüfung bei eingehenden Mails für einen Mailaccount

## 10. Spamerkennung für einen Mailaccount einrichten

**Wichtige Hinweise zu netqmail-Dateien (im weiteren Verlauf des Tutorials nur noch gemäß dem Dateinamen als .qmail-Dateien bezeichnet) und maildrop-Dateien (im weiteren Verlauf des Tutorials nur noch gemäß dem Dateinamen als .mailfilter-Dateien bezeichnet) !:** Mit den folgenden zwei Schritten 10.1 und 10.2 wird die Art, in der Mails in Empfang genommen werden, modifiziert. CRM114 wird in dem Empfangsprozess jeder ankommenden E-Mail des Mailaccounts eingebaut und kann so auf Ham und Spam prüfen. Anschließend wird die E-Mail, dem Ergebnis der Prüfung entsprechend, in den Spamerkannt-Ordner oder in den Posteingangs-Ordner gespeichert. Das Einbauen von CRM114 in den Mailempfang erfolgt für jeden Mailaccount einzeln, indem jeweils eine entsprechende .mailfilter-Datei und eine entsprechende .qmail-Datei angelegt werden und anschließend, mit entsprechenden Befehlen gefüllt, verwendet werden. Legt man auf einem uberspace einen neuen Mailaccount über den Befehl `uberspace mail user add [!USERNAME!]` an, so werden im Rahmen dieses Vorgangs erstmal *keine* .mailfilter-Datei und *keine* .qmail-Datei zu diesem Mailaccount angelegt, da dies für die direkte Zustellung einer ankommenden E-Mail an einen speziellen Mailaccount mit dem entsprechenden Username nicht notwendig ist. Will man nun CRM114 in diesen Mailempfangsweg einbauen, so werden die entsprechende .qmail- und .mailfilter-Datei mit den richtigen Inhalten definitiv benötigt. Sofern für den Mailaccount bisher noch keine .mailfilter-Datei und keine .qmail-Datei angelegt bzw. verwendet wird, kann die Anleitung wie beschrieben weiter befolgt werden. In den folgenden Schritten werden diese Dateien angelegt und entsprechend mit Programmcode gefüllt. Existieren jedoch bereits Zustellregelungen für diesen Mailaccount über eine .qmail-Datei und .mailfilter-Datei(en), so dürfen die folgenden Schritte nicht 1:1 übernommen werden! Stattdessen müssen die vorhandenen Dateien entsprechend angepasst werden. Wer bereits eigene .qmail- und .mailfilter-Dateien angelegt hat wird die Erfahrung damit haben und wissen, was zu tun ist. Wer sich unsicher ist sollte prüfen, ob und welche .qmail- und .mailfilter-Dateien mit der Benennung .qmail_[!USERNAME!] und .mailfilter-[!USERNAME!] für den jeweilige Mailaccount im Homeordner existieren. Die Prüfung auf Vorhandensein der Dateien kann mit der Auflistung aller Dateien des Homeordner geschehen. Dafür folgenden Befehl in der Shell ausführen: `ls -la ~`


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
**Wichtig:** Sofern bereits eine maildrop/.mailfilter-Datei für den Mailaccount existiert, kann das folgende nicht einfach übernommen werden. Stattdessen muss es in die vorhandenden Regelungen ggf. angepasst eingebaut werden.
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
**Wichtig:** Sofern bereits eine .qmail-Datei für den Mailaccount (.qmail_[!USERNAME!]) existiert, kann das folgende nicht einfach übernommen werden. Stattdessen muss es in die vorhandenden Regelungen ggf. angepasst eingebaut werden.
```
|maildrop $HOME/.mailfilter_[!USERNAME!]
```
Speichern und schließen, wie gewohnt mit: Strg+X, `y`, Enter.

Erklärung: Eingehende Mails für den Alias [!USERNAME!] werden, sofern eine .qmail-Datei für den [!USERNAME!]-Alias mit dem Dateinamen .qmail_[!USERNAME!] existiert, gemäß den Inhalten der Datei verarbeitet. Hier wurde diese .qmail_[!USERNAME!]-Datei angelegt und durch die in ihr gespeicherte Anweisung wird folgendes erreicht: Die ankommende E-Mail wird dem Programm 'maildrop' mit dem Pfad zur Datei mit den Filterregeln (.mailfilter_[!USERNAME!]) übergeben. Die Datei mit den Filterregeln wurde in Schritt 10.1 angelegt.


# VI  - Anleitung zur Nutzung und zum Anlernen
## 11. CRM114 testen, nutzen und trainieren.
Jede eingehende E-Mail wird vom CRM114 einer Prüfung unterzogen und bekommt eine von drei möglichen Kennzeichnungen: Ham, Spam oder Unsure (also Unsicher).\
Je nach Kennzeichnung verfährt die eingerichtete Zustellung über die .qmail- und die .mailfilter-Dateien mit der E-Mail unterschiedlich wie folgt:
- **Ham**: Die E-Mail wird im Posteingang abgelegt.
- **Spam**: Die E-Mail wird im Ordner `0 Spamfilter/als Spam erkannt` abgelegt.
- **Unsure**: Die E-Mail wird im Posteingang abgelegt und der Betreff wird um `[UNSURE]` ergänzt.

Den Ordner `0 Spamfilter/als Spam erkannt` musst du regelmäßig prüfen!\
Hier sammeln sich die E-Mails an, die CRM114 als Spam erkennt. E-Mails in diesem Ordner, die auch wirklich Spam sind, kannst du einfach löschen und musst sie CRM114 nicht erneut zum Anlernen vorlegen. Findest du in diesem Ordner allerdings eine E-Mail, die eigentlich kein Spam ist, dann ist Sorgfalt geboten! Die folgenden Erklärungen sind dann besonders wichtig, insbesondere Beispiel 3!

Zu Anfang wird CRM114 viele eingehende E-Mails mit einem [UNSURE] im Betreff versehen, und nur wenige E-Mails werden im Spam-Erkannt-Ordner abgelegt werden. CRM114 muss lernen, die E-Mails in deinem Sinne einzuschätzen. Dazu musst du CRM114 zeigen, ob eingehende E-Mails aus deiner Sicht Spam oder Ham darstellen. Es ist auch möglich (und sinvoll), CRM114 zu korrigieren, wenn es eine eingehende E-Mail falsch eingeschätzt hat.

Das Anlernen erfolgt über die beiden Ordner `0 Spamfilter/als Spam lernen` und `0 Spamfilter/als Ham lernen`, mit deren Hilfe du CRM114 E-Mails zum Lernen vorlegen kannst.

**GANZ GANZ WICHTIG, denn es droht DATENVERLUST:**\
**E-Mails, die du in die Lernordner speicherst, werden nach dem Anlernen durch CRM114 !! unwiederbringlich gelöscht !!. Es besteht also das Risiko, E-Mails unwiederbringlich zu verlieren, wenn nicht sorgfältig mit der Lernfunktion umgegangen wird. Aus diesem Grund müssen die folgenden Regeln verstanden und eisern beachtet werden:**
- **!!** Mails, die als **Ham** angelernt werden sollen, in den Ordner `0 Spamfilter/als Ham lernen` **immer KOPIEREN !!**
- Mails, die als **Spam** angelernt werden und danach gelöscht werden sollen, kann man in den Ordner `0 Spamfilter/als Spam lernen` verschieben.

>**Beispiel 1**:\
>Eine E-Mail landet in deinem Posteingang und wurde im Betreff mit [UNSURE] gekennzeichnet. Diese E-Mail ist Ham und du möchtest, dass solche E-Mails zukünftig von CRM114 auch als Ham erkannt werden (also kein [UNSURE] mehr bekommen). Diese E-Mail **kopierst** du in den Ordner `0 Spamfilter/als Ham lernen` und belässt die originale E-Mail in deinem Posteingang (oder sortierst sie nach deinem Bedarf in einen anderen Ordner). Würdest du die E-Mail in den Ham-Lernordner verschieben, so wäre sie nach dem Anlernen für immer gelöscht.

>**Beispiel 2**:\
>Eine E-Mail landet in deinem Posteingang (ganz gleich ob mit oder ohne [UNSURE]). Diese E-Mail ist aber Spam, und du möchtest, dass solche E-Mails zukünftig von CRM114 auch als Spam erkannt werden. Da du diese E-Mail nach dem Anlernen auch garnicht mehr gespeichert wissen willst, **verschiebst** du diese E-Mail in den Ordner `0 Spamfilter/als Spam lernen`. Nach dem Anlernen durch CRM114 ist diese E-Mail dann gelöscht.

>**Beispiel 3**:\
>Eine E-Mail wird von CRM114 beim Eingang als Spam erkannt und aus diesem Grund findest du sie nicht im Posteingang sondern im Ordner `0 Spamfilter/als Spam erkannt`. Du stellst fest, dass diese E-Mail fälschlicherweise als Spam erkannt wurde und sie für dich eigentlich Ham darstellt. Nun gehst du wie folgt vor: Zuerst **verschiebst** du die E-Mail in den Posteingang, denn dort gehört sie dauerhaft hin. Anschließend **kopierst** du die E-Mail aus Posteingang in den Ham-Lernordner, damit CRM114 lernt, solche E-Mails in Zukunft als Ham anstelle von Spam einzustufen. Die E-Mail wird nach dem Anlernen aus dem Ham-Lernordner gelöscht sein, aber im Posteingang ist sie noch vorhanden.

Umlernen ist also problemlos möglich: Eingehende E-Mails, die Spam sind, aber im Posteingang landen, gehören zum Anlernen in den Spam-Lernordner **verschoben**. Eingehende E-Mails, die Ham sind, aber im Spam-Erkannt-Ordner landen, gehören zur Aufbewahrung in den Posteingang **verschoben** und anschließend in den Spam-Lernordner **kopiert** (damit sie zwar gelernt werden aber nicht verloren gehen).

Zu Anfang ist es notwendig, CRM114 viele ankommende E-Mail via Lernordner zu zeigen, und zwar sowohl Spam als auch Ham! Nach einiger Zeit wird CRM114 gut darin, ankommende E-Mails korrekt einzuschätzen. Dann ist es nur noch sporadisch notwendig, eingehende E-Mails zum Anlernen zu übergeben, insbesondere die, die mit [UNSURE] gekennzeichnet sind. Mails im Spam-Erkannt-Ordner sollten regelmäßig durchgesehen werden: Korrekt erkannter Spam kann einfach gelöscht werden, fehlerhaft erkannter Spam muss in den Posteingang verschoben und anschließend als Ham zum Umlernen gegenüber CRM114 präsentiert werden.

Ich rate an dieser Stelle davon ab, CRM114 nach der Installation mit einem Haufen alter E-Mails zu trainieren! Das ist auch garnicht nötig, denn CRM114 lernt anhand neuer E-Mails sehr schnell.

# VII - Sonstiges
## 12. Tipps

### 12.1 Eigene Ordnerstruktur verwenden
Eine eigene Ordnerstruktur in Abweichung zu Punkt 9 zu verwenden ist problemlos möglich. Wichtig ist, dass immer drei Ordner existieren müssen, damit es wie beschrieben funktioniert: Jeweils ein Ordner für erkannten Spam, Mails zum Anlernen als Spam und Mails zum Anlernen als Ham.

Erst die Ordner mit gewünschtem Namen und Verschachtelung im Mailaccount anlegen, dann folgende Dateien ändern:

**Anlernordner für Ham und Spam**
In der Datei 'learn_maildir' die folgenden zwei Zeilen finden und abändern:
```
HAMDIR=".0 Spamfilter.als Ham lernen"
SPAMDIR=".0 Spamfilter.als Spam lernen"
```

**Ordner, in dem erkannter Spam abgelegt wird**
In der Datei .mailfilter-Datei (siehe Punkt 10.1) folgende Zeile finden und abändern:
```
MAILDIRSPAM="$MAILDIR/.0 Spamfilter.als Spam erkannt"
```

## 13. Einschränkungen & deren Lösung

**Für alle im folgenden beschriebenen Einschränkungen von CRM114 gibt es eine bereits implementierte Lösung!**

### 13.1 CRM114 hat Schwierigkeiten beim Anlernen "großer" E-Mails (Workaround implementiert)
**Problembeschreibung:** Eine E-Mail, die zum Anlernen in einem Ordner liegt und die in Summe größer als 3,9 MB ist, bringt CRM114 auf Uberspace (und wohl generell) zu einem kompletten Abbruch der Ausführung des Anlernvorgangs. Jede Ausführung des Anlernens scheitert, so lange sich diese E-Mail in einem Anlernordner befindet. Nach dem Löschen dieser E-Mail aus dem Anlernordner funktioniert die Erkennung wieder reibungslos. Ursache ist ein cachebezogenes Problem der Komponenten von CRM114, für das ich keinen Fix gefunden haben. (Weitere Infos [hier](https://sourceforge.net/p/crm114/mailman/message/22532062/) und [hier](https://sourceforge.net/p/crm114/mailman/message/28066834/)). CRM114 erwartet wohl, dass ihm E-Mails immer frei von Anhängen bzw. gekürzt auf eine passable Größe übergeben werden. Aber auch wenn Spammer selten E-Mails mit großen Dateianhängen verschicken, so kann es doch gewünscht und sinnvoll sein, CRM114 auch E-Mails zum Anlernen vorzulegen, die große Dateianhänge haben, insbesondere bei Ham.\
\
**Implementierte Lösung: Zwei Varianten zur Fehlervermeidung sind implementiert:**\
Genutzt wird immer **entweder** Variante 1 **oder** Variante 2. Ein Wechsel zwischen beiden Varianten ist jederzeit möglich, auch im laufenden Betrieb.

- **Variante 1** (Als **Standard aktiv** und sehr sicher funktionstüchtig, aber mit Einschränkungen verbunden)\
Dies ist die radikale Variante, die aber garantiert immer funktioniert: E-Mails in Anlernordnern, die größer als 3,9 MB sind, werden vor dem Anlernen gelöscht und somit beim Anlernen nicht berücksichtigt. Nachteil: E-Mails, die größer als 3,9 MB sind, lassen sich nicht nutzen, um den Filter zu trainieren. Da man dies nicht mitbekommt verhält sich diese Variante also im Fall großer E-Mails nicht so, wie man den Anlernvorgang eigentlich erwartet.
- **Variante 2** (experimentell, muss manuell aktiviert werden)\
Die elegante Variante, von der aber nicht ganz klar ist, ob sich diese immer problemlos verhält: E-Mails in Anlernordnern, die größer als 3,9 MB sind, werden vor dem Anlernen auf 3,9 MB "gekürzt" (genutzt wird truncate) und werden dann regulär von CRM114 angelernt. In den ersten 3,9 MB einer E-Mail sollten in der Regel alle Informationen vorhanden sein, die CRM114 benötigt, um die E-Mail anzulernen. Inhalte von Dateianhängen sind für eine Spamerkennung eher irrelevant.

**Umschalten der genutzten Variante (ACHTUNG, DIES GESCHIEHT AUF EIGENE VERANTWORTUNG!)**\
In der Datei 'learn_maildir' die Variable CACHEWORKAROUND in Zeile 61 entweder mit 1 oder 2 füllen und die Datei anschließend speichern. Die Variable ist vorbelegt mit einer 1, Standard ist also die Nutzung der Variante 1. Ein Wechsel zwischen beiden Varianten ist jederzeit möglich.

**Ändern der Mailgröße, ab der die Varianten greifen sollen (ACHTUNG, DIES GESCHIEHT AUF EIGENE VERANTWORTUNG!)**\
Es ist möglich, das Limit von 3,9 MB abzuändern. Ich rate aber ausdrücklich davon ab, denn dies kann zu den oben benannten Fehlern beim Anlernvorgang führen. Ändern geht wie folgt: In der Datei 'learn_maildir' die Variable MAILSIZELIMITkb in Zeile 59 mit der gewünschten Kilobytemenge befüllen. Standard ist: 3915

### 13.2 Die Spamerkennung eingehender "großer" E-Mails verzögert die Zustellung (Workaround implementiert)
**Implementierte Lösung:** Eingehende E-Mails, die größer als 2 MB sind, werden nicht an CRM114 zur Spamrüfung übergeben. Dies geschieht, um die Last für CRM114 gering zu halten und weil "echte" Spammails selten größer sind als 2 MB. Auf eigene Verantwortung kann dieser Wert in der .mailfilter-Datei abgeändert werden. 


## 14. Credits

Externe Quellen, die für dieses Tutorial bzw. seine Durchführung herangezogen werden:
- CRM114 unter GPLv2-Lizenz
http://crm114.sourceforge.net/
- TRE (The free and portable approximate regex matching library) unter BSD-Lizenz von Ville Laurikari
https://laurikari.net/tre/ & https://github.com/laurikari/tre/
- Dieses Tutorial, Texte, Quellcode und Scripte basieren auf einem (inzwischen gelöschten) Blogbeitrag von Bernhard Ehlers. Er hat es mir gestattet, diese als Vorlagen für eine erneute Veröffentlichung zu verwenden.

## 15. Kontakt
-- folgt --
