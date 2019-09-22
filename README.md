# Lernenden Spamfilter auf Uberspace7 selbst betreiben 
Vorweg: **Ein Dank an Bernhard Ehlers!** Diese Anleitung basiert auf seinem Blogtext und seinen Config-Dateien. Er gab mir die Erlaubnis, sie als Grundlage zu verwenden.
## Hintergrund
Stand heute bietet der Hosting-Anbieter [uberspace](https://www.uberspace.de) auf seinem Produkt [Uberspace7](https://blog.uberspace.de/tag/uberspace7/) (abgekürzt U7) von Haus aus keinen *lernenden* bzw. *trainierbaren* Spamfilter an. Bisher existiert unter U7 vorimplementiert nur folgender, sehr rudimentärer, Spamschutz: Jede eingehende E-Mail wird automatisch mit einem Rspamd-Score versehen und bei einem Wert größer 15 sofort gelöscht (siehe [U7-Manual: 'Filtering mails'](https://manual.uberspace.de/mail-filter.html)). Es ist möglich, via qmail und mailfilter, auch bei niedrigeren Rspamd-Scores eigene Verarbeitungsschritte einzubauen. Wer Spam effektiv ausfiltern möchte, der benötigt einen trainierbaren Spamfilter, der in viele Mailclients fest integriert ist. Wer über mehrere Geräte hinweg auf E-Mails zugreift, wird aber nicht auf einen serverseitigen lernenden Spamfilter verzichten wollen. Und bis die Ubernauten einen solchen Filter auf U7 anbieten, hilft diese Anleitung dabei, sich selbst einen solchen Spamfilter einzurichten.

An dieser Stelle noch der Hinweis, dass es auf dem Vorgängerprodukt U6 mit SpamAssassin (mit integriertem Regelwerk) und DSPAM (trainierbar) vorinstallierte Spamfilter gibt/gab. Diese funktionieren zwar, werden bei U7 aber nicht mehr integriert werden.

## Spamfiltersoftware der Wahl: CRM114

Mit folgendem Zitat von Bernhard ist alles gesagt:
> "Nach etwas Recherche habe ich mich für [CRM114](http://crm114.sourceforge.net) entschieden. Es wurde zwar sehr lange nicht mehr aktualisiert, ist aber weiterhin in vielen Linux-Distributionen präsent. Der Lernalgorithmus ist sehr schnell und effizient, das Programm ist sehr klein und performant."

## Voraussetzung: Mailaccount mit korrekter Ordnerstruktur

Es besteht ein Mailaccount, der mit 'uberspace mail user add [USERNAME]' erstellt wurde.
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
Die Ordnerstruktur erstellt man komfortabel mit folgenden Befehlen:

Zuerst folgenden Befehl in der Shell ausführen.
 **WICHTIG: [USERNAME] muss, einschließlich der [], durch den Benutzernamen des Mailaccount ersetzt werden!**
```Shell
export MAILUSERNAME=[USERNAME]
```

Anschließend die folgenden vier Befehle in der Shell ausführen:
```Shell
test -d "$HOME/users/$MAILUSERNAME/.0 Spamfilter" || maildirmake "$HOME/users/$MAILUSERNAME/.0 Spamfilter"                
test -d "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Ham lernen" || maildirmake "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Ham lernen"
test -d "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Spam lernen" || maildirmake "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Spam lernen"
test -d "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Spam erkannt" || maildirmake "$HOME/users/$MAILUSERNAME/.0 Spamfilter.als Spam erkannt"
```
## Weiter geht's

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
curl -sSL https://github.com/flowdx/crm114_u7/archive/master.zip -o master.zip
unzip master.zip
rm master.zip
mv ./crm114_u7-master/* ./
rm -r crm114_u7-master
chmod 755 cache_cleanup crm114 cssdiff cssmerge cssutil db_init learn_maildir
sh db_init
cd ..
```
