--- 
layout: post
title: Abenteuer mit Hunspell Wörterbüchern
date: 2020-04-16
author: Christian Glahn
tags: 
- textanalyse
- methoden
- tools
- R
---

Stemmen und Lemmatisieren sind zwei Grundtechniken zur Datenaufbereitung von unstrukturierten Texten. Diese Techniken unterstützen uns beim bestimmen des Wortstamms von Wörtern, wobei der Wortstamm sich auf die jeweilige Grundform eines Wortes bezieht. Damit das funktioniert, werden Wörterbücher (*dictionaries*) benötigt.

In R nimmt uns die Arbeit für beide Techniken die [`hunspell`-Bibliothek](https://cran.r-project.org/package=hunspell) ab. Speziell die Funktion `hunspell_stem` ist dabei der ideale Begleiter für die Textanalyse. Dabei kann dem folgenden Muster gefolgt werden: 

```R
library(tidyverse)
library(hunspell)

wörterbuch = "de_CH"

wortliste %>%
    mutate(stamm = hunspell_stem(wort, dict = dictionary(wörterbuch))) %>%
    mutate(stamm = lapply(stamm,nth,-1)) %>%
    unnest(stamm) %>%
    mutate(stamm = ifelse(is.na(stamm), wort, stamm))
```

Die `hunspell`-Bibliothek setzt auf der [`hunspell`-Plattform](http://hunspell.github.io/) auf, die sehr modular aufgebaut ist.

Bei der Arbeit mit den [deutschen Wörterbüchern](https://www.j3e.de/ispell/igerman98/dict/) ist mir aber aufgefallen, dass die Informationen für das Lemmatisieren komplett fehlt und einige Regeln für das richtige Stemmen nicht bzw. nicht vollständig umgesetzt sind. Deshalb muss für bestimmte Analysen selbst Hand angelegt werden und das Wörterbuch an die eigenen Bedürfnisse angepasst werden. 

Ein `hunspell`-Wörterbuch besteht aus einer Wortliste und Genratorregeln, dem sogenannten affix. Diese beiden Teile sind in zwei Dateien abgelegt. Die Wortliste endet auf `.dic` und das Affix endet auf `.aff`, wobei der Name des Wörterbuchs für beide Dateien identisch sein muss. Ein Wörterbuch ist nur dann vollständig, wenn beide Dateien im gleichen Verzeichnis gespeichert sind. 

Die Regeln des Wörterbuchs müssen im Affix in einer [eigenen Syntax](https://www.systutorials.com/docs/linux/man/4-hunspell/) ausgedrückt werden.

### Das deutsche Perfekt und die Krux mit dem Stemmen

In den offiziellen Wörterbüchern fehlen mit derzeit die Stemmregeln für das Perfekt mit der Vorsilbe `ge-`. Das liesse sich wahrscheinlich mit der folgenden Prefix-Regel in der `.aff`-Datei des Wörterbuchs realisieren: 

```
PFX G Y 1
PFX G ge 0
```

Taucht ein Wort mit dem `ge-` Prefix in der Wortliste auf, dann wird es als eigenständiges Wort behandelt.

Für das Verb `kaufen` müsste dann der Eintrag in der Wortliste wie folgt ergänzt werden: 

```
kaufen/BDGIVXYW
```

Dabei ist zu beachten, dass die Buchstaben nach dem Schrägstich jeweils auf die Stemmregeln im Affix für dieses Wort verweisen.

### Unregelmässige Verben

Bei unregelmässigen Verben muss der Wortstamm explizit festgelegt werden. Diese Information muss in der Wortliste, d.h. in der `.dic`-Datei, des Wörterbuchs festgehalten werden. Falls ein Wort in der gleichen Schreibweise mehrere Bedeutungen hat, müssen die Wortstämme auseinandergehalten werden, wie das folgende Beispiel zeigt: 

```
bin st:sein
bist st:sein
gewesen st:sein
ist st:sein
sein 
seid st:sein
sind st:sein
war st:sein
waren
waren st:sein
warst st:sein
wart st:sein
```

Zusätzlich könnten mit den `is:` und `po:` Annotationen zusätzliche Hinweise zur Grammatik hinterlegt werden, die bei der Textanalyse genutzt werden können. Diese Annotationen wären dann mit der  `hunspell_analyze`-Funktion zugänglich. 

Hunspell erlaubt Hinweise über den Kontext einer Schreibweise, so dass die o.g. Information wie folgt aufgebaut sein könnte.

```
ich bin st:sein
du bist st:sein
gewesen st:sein
er ist st:sein
sie ist st:sein
es ist st:sein
sein 
ihr seid st:sein
wir sind st:sein
sie sind st:sein
ich war st:sein
er war st:sein
sie war st:sein
```

Obwohl das ein guter Hinweis wäre, finden sich diese Information in Texten nicht genau in dieser Form. Daher machen diese Hinweise keinen Sinn für deutsche Wörterbücher.

### Verben mit erweiterter Grundform

Bei zusammengesetzen Verben wie z.B. "verkaufen", "einkaufen" oder "abzahlen" liesse sich das Wörterbuch weiter vereinfachen. Allerdings würden diese Regeln ebenfalls in das Stemmen eingreifen. Das ist ggf. problematisch, wenn sich die Wortbedeutung durch das Prefix ändert und der daraus generierte Wortstamm die Analyse in die Irre führen würde. Die folgenden Regeln sind daher nur als Randnotiz zu verstehen.  

Die `affix`-Regeln wären aber wahrscheinlich wie folgt:

```
PFX V Y 5
PFX V 0 ver .
PFX V 0 ab .
PFX V 0 be .
PFX V 0 ein . 
PFX V 0 über .
```

Alternativ liesse sich die `affix` Regel so formulieren, um den Fall von "eingekauft" richtig zu erkennen.

```
PFX V Y 7
PFX V 0 ver .
PFX V 0 ab .
PFX V 0 ein .
PFX V 0 über .
PFX V 0 einge .
PFX V 0 abge .
```

Besser wäre allerdings eine Lösung mit einer Regel für schwache Verben, die das eingebette `-ge-` richtig abbildet. 
