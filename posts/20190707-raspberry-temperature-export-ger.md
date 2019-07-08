# Messung und Export der Temperatur mit einem Raspberry Pi

Bei sommerlichen Temperaturen kann die Arbeit am PC schonmal anstrengend werden. Daher entschied ich mich die Temperatur in Büro und Wohnung in irgendeiner Form auszuwerten.

#### Raspberry Pi

Glücklicherweise habe ich noch ein altes Raspberry Pi 2 in der Ecke liegen gehabt. Bastelanleitungen für Temperaturmessschaltungen gibt es zudem genug im Internet.

- https://buyzero.de/blogs/news/tutorial-dht22-dht11-und-am2302-temperatursensor-feuchtigkeitsensor-am-raspberry-pi-anschliessen-und-ansteuern
- https://www.einplatinencomputer.com/raspberry-pi-temperatur-und-luftfeuchtigkeitssensor-dht22/
- https://tutorials-raspberrypi.de/raspberry-pi-luftfeuchtigkeit-temperatur-messen-dht11-dht22/

Ich entschied mich bei meiner Lösung für einen AM2302 Sensor der bereits mit einem Pullup-Widerstand ausgestattet ist. Dadurch konnte ich den Sensor direkt mit den GPIO Pins des Raspberry Pi verbinden. Kleinere Anpassungen am Gehäuse des Pi waren dafür leider unumgänglich.

![Rasperrry Pi mit AM2302 Sensor](https://raw.githubusercontent.com/rotfuchs/rotfuchs.dev/master/images/20190707-raspberry-pi.jpg)

Der nächste Schritt war das Auslesen der Daten, ein Testlauf. Natürlich gab es auch in diesem Fall bereits viele Softwarelösungen zu finden, der überwiegende Teil davon in Python geschrieben. Ich entschied mich aufgrund zahlreicher Empfehlungen für [Adafruit](https://github.com/adafruit/Adafruit_Python_DHT).

Tatsächlich funktionierte der Testlauf auf Anhieb. Der Temperatursensor lieferte über Adafruit sofort die verlangten Werte. Die Reaktionszeit zwischen dem Ausführen der Abfrage und der Ergebnislieferung könnte kürzer sein, andererseits war die neuste Version von Raspbian allgemein etwas langsam auf meinem in die Tage gekommenen Raspberry Pi.

Die Grundvoraussetzungen für einen Datenexport wurden geschaffen.

#### Der Datenexport

Mein nächster Schritt war ein CSV Export der Temperaturinformationen. Ich bin mir absolut sicher dass es hierzu fertige Lösung in Python gibt, in diesem Fall ging ich aber einen anderen Weg und konzentriere mich auf eine Eigenentwicklung in go.
Die Sprache go ist auf betagter Hardware sehr schnell, außerdem bereitet mir die Arbeit mit dieser viel Spaß, und die Syntax ist eine willkommene Abwechselung zur [OOP-Welt](https://de.wikipedia.org/wiki/Objektorientierte_Programmierung).

Die Umsetzung in go gestaltete sich sehr einfach, auch da go bereits alles Nötige für [die Verarbeitung von CSV Dateien](https://golang.org/pkg/encoding/csv/) implementiert hat.

```go
var temperatureData fileWriter.Temperature
var value string
var humidityString string

csvWriter := fileWriter.NewCsvWriter("BueroTemperatur.csv")

for {
    temperature, humidity, err := temperatureReader.read()
    
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Printf("Temperature %v*C, Humidity %v%%", temperature, humidity)

    value = fmt.Sprintf("%.2f", temperature)
    humidityString = fmt.Sprintf("%.2f", humidity)

    temperatureData = fileWriter.NewTemperature(
        time.Now(),
        value,
        humidityString,
    )
    csvWriter.AppendTemperatureData(temperatureData)

    time.Sleep(5 * time.Minute)
}
```

Das go Skript ist darauf ausgelegt solange zu laufen bis es von dem Benutzer abgebrochen wird. Dabei wird alle 5 Minuten die Temperatur gemessen und in eine CSV Datei geschrieben. Existiert die CSV Datei bereits werden die Werte natürlich nur angehägt. 

Bei einer Messung die alle 5 Minuten stattfindet, habe ich ziemlich genau 288 Werte pro Tag, auf ein Jahr gerechnet entspricht das einer Menge von 105120 Werten. Da es sich außerdem nur um 1-2 Datenwerte handelt (Temperatur, Luftfeuchtigkeit), kann ich die statig ansteigende Datenmenge in der CSV Datei für die nächsten Jahre erst einmal vernachlässigen. Beruflich hatte ich es schon mit CSV-Dateien zu tun die mehrere Millionen Datensätze täglich beinhalteten. Dagegen sind diese Werte sehr gering.

#### Die Auswertung

Jetzt nachdem ich mir eine CSV exportiert habe, fehlt nur noch die Auswertung. Im Idealfall über ein hübsches Diagramm. Hierfür habe ich mich bei der Softwarelösung meines [Arbeitgebers](https://www.scoreworx.de) bedient. Als Entwickler kenne ich die erforderlichen Schnittstellen und kann direkt loslegen. Da CSV Dateien unterstützt werden, und diese direkt über einen POST an den Server überragen / in einer SQL-Datenbank importiert werden könnnen, habe ich hierzu einen Cronjob auf meinem Raspberry Pi eingerichtet. 

```bash
curl -F 'csv[]=@/home/pi/weather-exporter/BueroTemperatur.csv' https://...
```

Die Erstellung des Diagramms war danach eine Kleinigkeit. Insgesamt habe ich zwei Diagramme erstellt. Im ersten Dia wird der Temperaturverlauf der lettzen 48 Stunden anzeigt.

![48 Stunden Temperaturwerte](https://github.com/rotfuchs/rotfuchs.dev/blob/master/images/20190707-bueo_temperatur.png?raw=true)

Im zweiten Diagramm habe ich die Ausgabe nach Tag gruppiert und einen Durchschnittswert ermittelt.

![8 Tage Durchschnitt](https://github.com/rotfuchs/rotfuchs.dev/blob/master/images/20190707-buerotemperatur_%5B7tage_durchschnitt%5D.png?raw=true)

Beide Diagramme haben als Basis die gleiche SQL-Abfrage, allein die Gruppierung macht den Unterschied bei der Auswertung der Ergebnisse. Das erleichtert mir die Pflege wenn ich später weitere Messwerte haben sollte.
