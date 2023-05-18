--- 
layout: post
title: R-Vektoren-und-Datumsoperationen
date: 2023-05-18
author: Christian Glahn
tags: 
- funktionale Programmierung
- R
- Map-Reduce
- Style
- No Loops
---

Bei [R-Bloggers](https://www.r-bloggers.com/) bin ich heute auf den [Beitrag von Steven Sanderson II](https://www.spsanderson.com/steveondata/posts/rtip-2023-05-17/index.html) gestossen, welcher die Funktion `strftime()` erklärt. Ich habe den Artikel eigentlich nur gelesen, um festzustellen, was die R-Community gerade so beschäftigt. Weil der gezeigte Code auf mich geradezu antiquiert wirkte, musste ich das Datum des Beitrags kontrollieren und habe festgestellt, dass der Beitrag erst einen Tag alt war 😱.

Hier ist der anstössige Code aus dem Originalbeitrag: 

```R
RightNow <- Sys.time()

# all of the modifiers
for (formatter in sort(c(letters, LETTERS))) {
  modifier <- paste0("%", formatter)
  print(
    paste0(
      modifier, 
      " used on: ",
      RightNow,
      " will give: ",
      strftime(RightNow, modifier)
    )
  )
}
```

Das verwendete Programmierparadigma ist **imperativ**, was nicht mehr ganz zum modernen R passt. Ich habe mich gefragt, wie aufwändig eine Lösung im **funktionalen** Programmierparadigma ausfallen würde, um den sprachlichen Eigenschaften von modernen R-Versionen gerechtzuwerden. Weil der Code im Originalbeitrag ohne zusätzliche Bibliotheken auskommt, musste die funktionale Lösung ebenfalls in Base R umgesetzt werden.

Hier ist meine Lösung: 

```R
c(letters, LETTERS) |> 
    sort() |>
    (\(formatter) paste0("%", formatter))() |> 
    (\(modifier, RightNow) paste0(
        modifier, 
        " used on: ", 
        RightNow, 
        " will give: ", 
        strftime(RightNow, modifier)
    ))(Sys.time())
```

Diese Lösung kommt ohne globale Variablen und Schleifen aus und ist ausserdem kompakter als die ursprüngliche Variante. 

Meine Lösung verwendet den *native Pipe*-Operator (`|>`) sowie die Kurzform für anonyme Funktionsdeklarationen (`\()`). Beide Operatoren sind seit zwei Jahren bzw. seit [Version 4.1.0 Teil von Base R](https://cran.r-project.org/bin/windows/base/old/4.1.0/NEWS.R-4.1.0.html) und gehören somit zum sprachlichen Kern von R.

Der Anfang des Codes ist bis zum Aufruf von `sort()` identisch mit der Logik des ursprünglichen Codes: Das Ergebnis ist ein Vektor mit der sortierten Kombination der beiden Basis-Vektoren `letters` und `LETTERS`. Ab diesem Punkt spare ich mir die `for`-Schleife und nutze aus, dass R-Funktionen immer für alle Elemente eines Vektors als implizite [Map-Funktion](https://de.wikipedia.org/wiki/MapReduce) ausgeführt werden.

Die Code-Zeile `(\(formatter) paste0("%", formatter))()` erstellt eine anonyme Hilfsfunktion, welche vor jeden übergebenen Parameterwert ein Prozentzeichen einfügt, und führt diese Funktion sofort aus. Diese Hilfsfunktion wird nur deshalb benötigt, weil der Pipe-Operator die Werte ausschliesslich als ersten Parameter an die nachfolgende Funktion übergeben kann. Die Funktion `paste0()` kann aber die übergebenen Zeichenketten nur in dieser Reihenfolge aneinanderfügen. Die Hilfsfunktion löst dieses Problem, indem die Funktion den Datenfluss der Pipe aufnimmt und dann innerhalb der Funktion korrekt arrangiert. 

Die beiden Hilfsfunktionen haben einen positiven Nebeneffekt: Die Variablennamen der ursprünglichen Lösung bleiben für die Lesbarkeit erhalten.

Damit die Hilfsfunktion sofort ausgeführt werden kann, muss die Funktionsdeklaration geklammert werden. Dieser Klammer folgen dann die Funktionsparameter. In diesem Fall steht für die Parameterliste nur `()`, weil der einzige Parameter von der Pipe eine Zeile darüber weitergereicht und deshalb nicht angegeben wird.

Das letzte Fragment erstellt die Ausgabezeichenkette und ist ebenfalls als anonyme Hilfsfunktion umgesetzt. Diesmal hat die Funktion zwei Parameter, weil für die Ausgabe zwei Werte von Bedeutung sind. Der erste Parameter (`modifier`) wird von der Pipe befüllt und ist der Ergebnisvektor der anderen Hilfsfunktion. Der zweite Parameter `RightNow` nimmt das Ergebnis der Funktion `Sys.time()` auf. Weil diese Parameterübergabe nur ***eine*** Ausführung dieser Funktion erfordert, entfällt die globale Variable `RightNow` des ursprünglichen Codes und die damit verbundenen potenziellen Seiteneffekte. 

Weil die Lesbarkeit durch die Kurzform der Funktionsdeklaration und die vielen Zeilenumbrüche für die Ausgabe leidet, hier noch eine Variante in der funktionalen Langform: 

```R
c(letters, LETTERS) |> 
    sort() |>
    (function (formatter) paste0("%", formatter))() |> 
    (function (modifier, RightNow) paste0(
        modifier, " used on: ", RightNow, " will give: ", strftime(RightNow, modifier)
    ))(Sys.time())
```

Das sieht doch gleich viel besser aus. 😎

Für die Funktionsweise dieser Logik empfielt sich die Lektüre des [Beitrags von Steven Sanderson II](https://www.spsanderson.com/steveondata/posts/rtip-2023-05-17/index.html).
