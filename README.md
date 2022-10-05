# Amtliche Hauskoordinaten der Stadt Essen
## Einleitung
Der Fachbereich 62 (Amt für Geoinformation, Vermessung und Kataster) veröffentlicht die amtlichen Hauskoordinaten der Stadt Essen über einen Web Feature Service (WFS) unter einer offenen Lizenz.

### Allgemeines
- OpenData-Portal: https://opendata.essen.de/dataset/amtliche-hauskoordinaten
- Ressource: https://geo.essen.de/arcgis/services/public/Amtliche_Hauskoordinaten_wfs/MapServer/WFSServer?request=GetCapabilities&service=WFS
- Metadaten-Bezeichner: 7cb8c7c0-4a7a-4059-9960-e0097d1e19e1
- Lizenz: Datenlizenz Deutschland – Namensnennung – Version 2.0" (dl-de/by-2-0), www.govdata.de/dl-de/by-2-0

### Aktuelle Problematik
In der Kommentarsektion des OpenData-Portals der Stadt Essen wird deutlich, dass viele Nutzer*innen mit dem derzeiten Angebot nur wenig anfangen können. Zudem scheint der Datensatz nicht einwandfrei zu sein. Im Folgenden sollen die technischen Eigenheiten des Datensatz und der Bereitstellung sowie eine mögliche lösung erarbeitet werden.

## Kritische Analyse des WFS und den ausgelieferten Daten
Auch wenn zur Analyse einfach die GDAL-Tools eingesetzt werden könnten, wird zunächst "LowLevel" vorgegangen. Es werden also mit den unbearbeiteten Rohdaten gearbeitet, die vom Server ausgeliefert werden.

### Eingesetzte Server-Software
Anhand der Ressource der URL wird deutlich, dass der WFS durch einen ArcGIS Server bereitgestellt wird. Im Konkreten handelt es sich um die Version 10.61. Es ist also mit Eigenheiten zu rechnen.

### Abfrage der Capabilities
``` sh
URL='https://geo.essen.de/arcgis/services/public/Amtliche_Hauskoordinaten_wfs/MapServer/WFSServer'

curl \
    --location \
    --remote-time \
    --get \
    --url "$URL" \
    --data 'SERVICE=WFS' \
    --data 'VERSION=2.0.0' \
    --data 'TYPENAME=Amtliche_Hauskoordinaten:Hauskoordinaten' \
    --data 'REQUEST=GetCapabilities' \
    --output 'capabilities.xml'
```

#### Verfügbare Koordinatensysteme
Das einzig verfügbare Koordinaten ist `urn:ogc:def:crs:EPSG::102329`. Es handelt es sich hierbei um eine ESRI-typische URN von EPSG:4647. Bei dieser Projektion erhält der Rechtswert (Easting) die UTM-Zonennummer "32". Die Koordinaten haben somit acht Stellen vor dem Dezimaltrennzeichen. Im Vergleich: Bei EPSG:25832 stehen nur sechs Ziffern vor dem Dezimaltrennzeichen.

#### Verfügbare Ausgabeformate
- GML32
- GML32+ZIP
- application/gml+xml; version=3.2
- GML3
- GML3+ZIP
- text/xml; subtype=gml/3.1.1
- GML2
- GML2+ZIP
- text/xml; subtype=gml/2.1.2
- GEOJSON
- GEOJSON+ZIP
- ESRIGEOJSON
- ESRIGEOJSON+ZIP
- KML
- application/vnd.google-earth.kml xml
- application/vnd.google-earth.kml+xml
- KMZ
- application/vnd.google-earth.kmz
- SHAPE+ZIP
- CSV
- CSV+ZIP

Der WFS liefert standardmäßig GML32 aus.

#### Paging
Der WFS liefert keine Paging-Informationen aus. Hierbei handelt es sich entweder um einen Konfigurationsfehler oder Absicht.

### Analyse der Metadaten des Datensatzes
``` sh
curl \
    --location \
    --remote-time \
    --get \
    --url "$URL" \
    --data 'SERVICE=WFS' \
    --data 'VERSION=2.0.0' \
    --data 'TYPENAME=Amtliche_Hauskoordinaten:Hauskoordinaten' \
    --data 'REQUEST=DescribeFeatureType' \
    --output 'default.gfs'
```

| Spaltenname | Spaltentyp | Spaltenlänge |
|-------------|------------|--------------|
|OBJECTID|integer|
|STRSCHL|double|
|HSNR|double|
|HSNRZU|string|20
|STRNAME|string|255
|ADRESSE|string|255
|STRNAME|string|255
|Y|double|
|X|double|

Bereits an der Wahl der Spaltentypen wird deutlich, dass hier mit Problemen zu rechnen ist:
- bei Straßenschlüssel _STRSCHL_ handelt es sich um eine Ganzzahl,
- die Hausnummer _HSNR_ kann theoretisch Bruchzahlen, etc. beinhalten, auch wenn dies in der Stadt Essen derzeit nicht der Fall ist,
- bei Rechts- _X_ und Hochwert _Y_ den Typ "double" zu verwenden, führt aufgrund von IEEE-754 unweigerlich zu Problemen; üblicherweise werden derartige Informationen als String gespeichert. Auf diese Weise wird ein Präzisionsverlust vermieten und es lässt sich problemlos in bessere Typen, wie z.B. PostgreSQL's _numeric_, casten.

### Anzahl der Items
``` sh
curl \
    --location \
    --remote-time \
    --get \
    --url "$URL" \
    --data 'SERVICE=WFS' \
    --data 'VERSION=2.0.0' \
    --data 'TYPENAME=Amtliche_Hauskoordinaten:Hauskoordinaten' \
    --data 'REQUEST=GetFeature' \
    --data 'RESULTTYPE=hits'
```

Zum Abrufzeitpunkt (2022-10-05) wird ein Wert von 102.981 Objekten angegeben (`numberMatched="102981"`).

### Datenabruf
Wie bereits abgemerkt, wird in den Capabilities keine Angabe zum Paging gemacht. Ausprobieren hat ergeben, dass der WFS nur 1.000 Objekte pro Abfrage liefert. Es müssen also 102 Einzelabfragen (`floor(102981 / 1000)`) stattfinden:

``` sh
[ -d download/ ] || mkdir download/
for i in {0..102}; do
    printf -v fn '%03d' "$i"
    [ -f download/"$fn".gml ] && continue;

    curl \
        --silent \
        --remote-time \
        --location \
        --get \
        --url "$URL" \
        --data 'SERVICE=WFS' \
        --data 'VERSION=2.0.0' \
        --data 'TYPENAME=Amtliche_Hauskoordinaten:Hauskoordinaten' \
        --data 'REQUEST=GetFeature' \
        --data "STARTINDEX=$((i*1000))" \
        --data "COUNT=$(((i+1)*1000))" \
        --output download/"$fn".gml;
    
    # optional
    # cp default.gfs download/"$fn".gfs;
done;
```

### Beispieldaten
1. Es ist zu bezweifeln, ob die _Aachner Straße_ wirklich von amtlicher Seite als _Aachener Str._ gewidmet wurde.
2. Rechts- (Easting) und Hochwert (Northing) wurden vertauscht
3. Effekte durch IEEE-754 in der GML-Koordinate.

``` xml
  <wfs:member>
    <Amtliche_Hauskoordinaten:Hauskoordinaten gml:id="Hauskoordinaten.9">
      <Amtliche_Hauskoordinaten:OBJECTID>9</Amtliche_Hauskoordinaten:OBJECTID>
      <Amtliche_Hauskoordinaten:STRSCHL>1</Amtliche_Hauskoordinaten:STRSCHL>
      <Amtliche_Hauskoordinaten:HSNR>4</Amtliche_Hauskoordinaten:HSNR>
      <Amtliche_Hauskoordinaten:STRNAME>Aachener Str.</Amtliche_Hauskoordinaten:STRNAME>
      <Amtliche_Hauskoordinaten:ADRESSE>Aachener Str. 4</Amtliche_Hauskoordinaten:ADRESSE>
      <Amtliche_Hauskoordinaten:Y>32359252.073</Amtliche_Hauskoordinaten:Y>
      <Amtliche_Hauskoordinaten:X>5701482.515</Amtliche_Hauskoordinaten:X>
      <Amtliche_Hauskoordinaten:SHAPE><gml:Point gml:id="Hauskoordinaten.9.pn.0" srsName="urn:ogc:def:crs:EPSG::102329"><gml:pos>5701482.515000001 32359252.073</gml:pos></gml:Point></Amtliche_Hauskoordinaten:SHAPE>
    </Amtliche_Hauskoordinaten:Hauskoordinaten>
```

1. Rechts- (Easting) und Hochwert (Northing) wurden vertauscht
2. Effekte durch IEEE-754 im Hochwert, da nicht als String gespeichert wurde.
``` xml
  <wfs:member>
    <Amtliche_Hauskoordinaten:Hauskoordinaten gml:id="Hauskoordinaten.136">
      <Amtliche_Hauskoordinaten:OBJECTID>136</Amtliche_Hauskoordinaten:OBJECTID>
      <Amtliche_Hauskoordinaten:STRSCHL>4</Amtliche_Hauskoordinaten:STRSCHL>
      <Amtliche_Hauskoordinaten:HSNR>14</Amtliche_Hauskoordinaten:HSNR>
      <Amtliche_Hauskoordinaten:STRNAME>Achtermbergbredde</Amtliche_Hauskoordinaten:STRNAME>
      <Amtliche_Hauskoordinaten:ADRESSE>Achtermbergbredde 14</Amtliche_Hauskoordinaten:ADRESSE>
      <Amtliche_Hauskoordinaten:Y>32366644.871</Amtliche_Hauskoordinaten:Y>
      <Amtliche_Hauskoordinaten:X>5705174.060000001</Amtliche_Hauskoordinaten:X>
      <Amtliche_Hauskoordinaten:SHAPE><gml:Point gml:id="Hauskoordinaten.136.pn.0" srsName="urn:ogc:def:crs:EPSG::102329"><gml:pos>5705174.060000001 32366644.871</gml:pos></gml:Point></Amtliche_Hauskoordinaten:SHAPE>
    </Amtliche_Hauskoordinaten:Hauskoordinaten>
  </wfs:member>
```

## Lösung
Nachdem nun alle Probleme bekannt sind, wird zur Lösung nun _ogr2ogr_ verwendet. Der Datensatz wird zunächst als GeoJSON in EPSG:4647 gespeichert.

``` sh
ogr2ogr \
    --config OGR_WFS_PAGE_SIZE 1000 \
    -preserve_fid \
    -unsetFieldWidth \
    -ct '+proj=axisswap +order=2,1' \
    -fieldTypeToString All \
    -sql 'SELECT objectid, strschl, hsnr, hsnrzu, adresse, strname, y AS x, x AS y FROM Hauskoordinaten' \
    -a_srs EPSG:4647 \
    hauskoordinaten_essen_2022-10-05_4647.geojson \
    WFS:"$URL" \
    -lco COORDINATE_PRECISION=4 \
    -lco SIGNIFICANT_FIGURES=12
```

Und nun die Transformation zu EPSG:4326 (WGS84)
``` sh
ogr2ogr \
   -t_srs EPSG:4326 \
   hauskoordinaten_essen_2022-10-05_4326.geojson \
   hauskoordinaten_essen_2022-10-05_4647.geojson

```
