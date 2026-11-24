# Att göra inför nästa version

Skrivet 2026-11-09, efter att `export-import` mergats till `main`. Version 1.0.1 är den senast släppta. Det som är gjort men inte släppt står under *Unreleased* i [CHANGELOG.md](../../CHANGELOG.md). Mats väntar med nästa version tills det kommit feedback på den första, så listan kan växa.

Reglerna i [CLAUDE.md](../../CLAUDE.md) gäller för varje punkt: källogg, metodbeskrivningarna hålls i takt med koden, och posten i `CHANGELOG.md` och i `species_data/metadata.txt` skrivs i samma incheckning som versionshöjningen.

## 1. Markera alla / avmarkera alla i artlistan på fliken Lokaler — klart (2026-11-24)

Två knappar, **Check all** och **Uncheck all** (`species_data/locality_tab.py`: `_check_all`), vid artlistan. De blockerar signalerna, ändrar alla kryssrutor och anropar `_refresh` en gång, inte en gång per art. Är ingen art ikryssad visar `_refresh` redan "Tick at least one species in the list." och Starta-knappen är avstängd, som tidigare. Test i `tests/test_plugin_load.py` (körs i CI mot QGIS). Texterna är i `species_data/i18n/species_data_sv.ts`/`.qm`, handledningen och `CHANGELOG.md` är uppdaterade.

## 2. Sök inom en polygon eller en ruta som användaren ritar på kartan

**Problem.** Fliken Geografi på Hämta har bara *Aktuell kartutsträckning*, *Län* och *Hela Sverige*. Man kan inte avgränsa sökningen till ett eget område, till exempel ett reservat eller ett avrinningsområde.

**Önskemål.** Användaren ritar en polygon på kartan, eller drar upp en ruta, och sökningen görs inom det området.

**Det som redan finns.** `Query` kan redan ta `bbox_wgs84` och `polygon_wgs84` och göra om dem till WFS-filter (`_bbox` och `_intersects` i `species_data/query.py`). `_intersects` tar en enkel polygon med hål och avvisar andra geometrier. Det som saknas är verktyget på kartan och en rad på fliken Geografi (`_geography_tab` i `species_data/dockwidget.py`, rad ca 302). `_canvas_bbox` visar hur en kartutsträckning görs om till WGS 84.

**Att bygga**
- Ett kartverktyg: `QgsMapToolExtent` (eller en egen rektangel) för rutan och ett polygonverktyg med `QgsRubberBand` för polygonen. Avsluta med högerklick, avbryt med Esc och återställ verktyget som var aktivt tidigare.
- En ny rad på fliken Geografi, *Ritat område*, med knapparna *Rita ruta*, *Rita polygon* och *Rensa*, och en synlig markering av området på kartan så länge det gäller.
- Omräkning från kartans CRS till WGS 84 med `QgsCoordinateTransform`, för både ruta och polygon.
- Metadata: `provenance.py` beskriver `bbox_wgs84` men inte `polygon_wgs84`. Lagrets metadata och abstract ska säga vilket område som användes.
- Förloppet ska följa det som gäller för övriga sökningar (räkna först, tak på 100 000 fynd, en sekund mellan sidorna).

**Frågor att avgöra först**
- Hur stor polygon klarar WFS-anropet? Filtret skickas i anropet, och en polygon med tusentals hörn kan bli för stor. Provas live mot `sosgeo.artdata.slu.se` innan det byggs, och hörn kan förenklas om det behövs (`QgsGeometry.simplify` med en tolerans som anges för användaren).
- Ska det ritade området kunna kombineras med län, eller ersätta det? Förslag: ersätter, som *Aktuell kartutsträckning* gör idag, så att valen går ut på samma sätt.
- Ska användaren kunna välja en befintlig polygon i ett lager (markerat objekt)? Det är enklare än att rita och vanligt i QGIS. Kan tas med som ett tredje alternativ, men det är inte önskat än.
- Träffar polygonen punkter vars osäkerhetscirkel går utanför området? `Intersects` på fyndets punkt gör det inte. Det ska stå i handledningen, eftersom användare annars kan tro att allt inom cirkeln finns med.
- Hur ser kartverktyget ut för en användare som redan har ett annat verktyg aktivt, och vad händer om pluginet stängs medan verktyget är på?

**Test.** `Query` med polygon ger rätt XML (finns delvis). Nytt: kartverktygets resultat blir en giltig `polygon_wgs84`. Ett live-test mot WFS med en liten polygon, som en manuell kontroll och inte i CI.

**Dokumentation.** Handledningen (avsnittet om Hämta, fliken Geografi), `CHANGELOG.md`, och en rad i README. Metodbeskrivningarna berörs bara om valet av område förändrar vad som kallas en lokal, vilket det inte gör.

## 3. Svar från SLU Artdatabanken om namn, logotyp och beskrivning (inkommet 2026-11-11)

**Status.** Mats har inte bestämt om han följer önskemålen. Punkten är en minnesanteckning, ingenting ska ändras förrän han bestämt sig. Svaret kom från Martin på SLU Artdatabanken efter att han provat pluginet och stämt av med chef och förvaltningsansvarig.

**Vad SLU sa**
- *Bra som det är:* README och handledningen säger tydligt att pluginet är fristående och varken utvecklas, ägs eller stöds av SLU eller SLU Artdatabanken, och anger varifrån data hämtas.
- *Logotyp:* SLU:s och SLU Artdatabankens logotyper får inte användas i pluginet eller för att presentera det, eftersom de kan ge intryck av att SLU står bakom eller samarbetar kring produkten. Kontrollera ikonen (`icon` i `species_data/metadata.txt`, bildfilen i paketet) och eventuella bilder i README och handledningen.
- *Namn:* SLU föreslår att namnet "Species Data (SLU Artdatabanken)" och GitHub-repositoryt byts, eftersom namnen i sig kan ge intryck av ett officiellt tillägg, särskilt i QGIS tilläggskatalog. Förslag: **Species Data (Sweden)** och repositorynamnet `species-data-sweden-qgis-plugin`. Annat namn är fritt. Att nämna SLU Artdatabanken, Artportalen och Species Observation System som datakälla är fortsatt tillåtet.
- *Beskrivning av datakällan:*
  - Handledningen: byt "Fynddata hämtas från SLU Artdatabankens Species Observation System (Artportalen) enligt deras API-villkor." till "Fynddata hämtas från SLU Artdatabankens Species Observation System enligt deras API-villkor. Pluginet använder bara fynd från datasetet Artportalen." Skälet: parentesen får Species Observation System och Artportalen att se ut som samma sak.
  - GitHubs huvudsida (repository-beskrivningen): "Artportalen data" blir "Artportalen dataset", alltså "A QGIS plugin for building dot maps of vascular plant observations from the Artportalen dataset held by SLU Artdatabanken, …".

**Att avgöra först**
- Ska namnet bytas? Ett namnbyte rör många ställen: `name` i `species_data/metadata.txt`, `species_data/plugin.py`, `species_data/dockwidget.py` (fönstertitel), `species_data/i18n/species_data_sv.ts` och `.qm`, `tests/test_metadata.py`, README, handledningen, metodbeskrivningarna, `CHANGELOG.md`, `docs/utveckling/` samt URL:erna `homepage`, `repository` och `tracker` i metadata.
- Repositorynamn: GitHub vidarebefordrar gamla adresser vid omdöpning, men `homepage`, `repository` och `tracker` i metadata och länkar i dokumenten bör uppdateras. Mappnamnet `species_data` (paketet) behöver inte ändras och bör inte det, eftersom det är pluginets id i QGIS.
- Har pluginet redan publicerats på plugins.qgis.org med det gamla namnet? Då gäller andra regler för namnbyte där. Kontrollera innan något ändras.
- Ikonen: om den innehåller SLU:s logotyp eller liknar den, byts den mot en egen. Annars räcker det att svara SLU att den inte gör det.
- Svar till Martin när Mats bestämt sig, oavsett utfall.

**Dokumentation.** Ändrar namnet eller beskrivningen något som metodbeskrivningarna citerar ska de hållas i takt med koden (CLAUDE.md). Posten i `CHANGELOG.md` och i `species_data/metadata.txt` ska vara lika.

## Redan känt och öppet (från tidigare, inte ändrat här)

- En kolumn för källa (`sources`) på samvariationslagret, som lokallagret har. Erbjudet, inte beställt.
- Frågor till SLU om licens per dataset, dubbletter mellan datakällor och en lista över leverantörer.
- Mer än kärlväxter och fler källor: se [kallor-och-organismgrupper.md](kallor-och-organismgrupper.md). iNaturalist har låg prioritet.
- Engelska kolumnnamn i exporten från Fynddata: kräver en export på engelska att jämföra med.
- Skärmbild till README (läggs i `docs/images/`), som Mats tar själv.
