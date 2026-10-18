1. Potenzial: Oracle-Kalibrierung als Obergrenze

Nimm die OOT-Predictions aus deinem Backtest und kalibriere sie absichtlich auf genau diesen Daten. Das ist geschummelt, aber es liefert dir die Obergrenze: Ist der Oracle-Gewinn klein, kannst du dir alles Weitere sparen.

Globale Variante: Gneiting & Resin zerlegen den mittleren Score in Miscalibration, Discrimination und Uncertainty: S̄ = MCB − DSC + UNC. Die Schätzung läuft über isotone Regression. MCB ist genau der Teil des Scores, den du durch eine Rekalibrierung auf ŷ maximal zurückholen könntest. Fertig implementiert ist das im Package model-diagnostics (pip). Es bietet die Score-Zerlegung und Bias-Plots für bedingte Kalibrierung über compute_bias mit einem Feature-Argument. Damit kannst du dir direkt den Bias nach Wochentag inklusive Standardfehler anschauen.

Wichtig: Die isotone MCB misst nur die Fehlkalibrierung als Funktion von ŷ. Deinen Bias nach Gruppe × Samstag sieht sie nicht. Dafür rechnest du zusätzlich das Oracle-k_{g,d} auf dem Eval-Fenster und vergleichst den Score vorher und nachher. Das ist dann deine Slice-MCB.

Nebeneffekt für MSE vs. Tweedie: DSC sagt dir, wie gut ein Modell nach perfekter Kalibrierung wäre. Wenn Tweedie mehr MCB hat (es unterschätzt ja), aber auch mehr DSC, dann gewinnt Tweedie mit Kalibrierung. Ein Vergleich nur auf dem Roh-Score würde das verdecken.

Nicht auf p-Werte schauen. Bei Retail-Datenmengen ist jeder Bias signifikant. Entscheidend ist die Effektgröße, also der Anteil der MCB am Score.

2. Übertragbarkeit: Ist der Bias persistent?

Ein großer Oracle-Gewinn bringt nichts, wenn der Faktor von Woche zu Woche springt. Rechne deshalb k_{g,d} pro Origin aus den OOT-Predictions:

python
k = (oot.groupby(["origin", "group", "dow"])[["y", "yhat"]].sum()
        .eval("y / yhat").unstack("origin"))
sat = k.xs(5, level="dow")  # Samstag: Zeilen = Gruppen, Spalten = Origins
lag_corr = [sat[a].corr(sat[b]) for a, b in zip(sat.columns, sat.columns[1:])]
signal, noise = sat.mean(axis=1).var(), sat.var(axis=1).mean()

Prüf das auf zwei Ebenen:

Gemeinsamer Samstags-Bias: Ist sat.mean() über die Origins stabil, z.B. immer etwa 1,12? Dann reicht ein gepoolter Faktor pro Wochentag. Das ist robust und braucht kaum Daten.
Gruppenspezifische Abweichungen: Wie stark ist lag_corr, und wie groß ist signal im Verhältnis zu noise? Nur wenn beides passt, lohnen sich Faktoren pro Gruppe × Wochentag. Sonst fittest du Rauschen.

Achtung: Eine niedrige Korrelation über die Gruppen heißt nicht, dass Kalibrierung nichts bringt. Wenn alle Gruppen denselben Bias haben, misst die Korrelation nur die Abweichungen davon.

3. Realer Gewinn: Walk-forward

Wie besprochen: Faktoren nur aus früheren Origins schätzen, dann auf den aktuellen Origin anwenden. Daraus ergibt sich die Capture Rate = realer Gewinn / Oracle-Gewinn. Eine hohe Capture Rate bedeutet, dass der Bias stabil und gut schätzbar ist. Eine niedrige heißt, dass die Schätzunsicherheit oder Drift den Gewinn auffressen. Zusätzlich wie vorher prüfen, ob es auf Item-Ebene und an den anderen Wochentagen nichts verschlechtert.

Entscheidungsregel: Post-Processing lohnt sich, wenn

der Oracle-Gewinn relevant ist (im Vergleich zu dem, was dir sonst ein Modell-Release bringt),
die Faktoren über die Origins stabil sind, und
der Walk-forward einen ordentlichen Teil des Oracle-Gewinns holt, ohne anderswo zu schaden.
