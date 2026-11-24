# Att göra inför nästa version

Skrivet 2026-11-09, efter att `export-import` mergats till `main`. Version 1.0.1 är den senast släppta. Det som är gjort men inte släppt står under *Unreleased* i [CHANGELOG.md](../../CHANGELOG.md). Mats väntar med nästa version tills det kommit feedback på den första, så listan kan växa.

Reglerna i [CLAUDE.md](../../CLAUDE.md) gäller för varje punkt: källogg, metodbeskrivningarna hålls i takt med koden, och posten i `CHANGELOG.md` och i `species_data/metadata.txt` skrivs i samma incheckning som versionshöjningen.

## 1. Markera alla / avmarkera alla i artlistan på fliken Lokaler — klart (2026-11-24)

Två knappar, **Check all** och **Uncheck all** (`species_data/locality_tab.py`: `_check_all`), vid artlistan. De blockerar signalerna, ändrar alla kryssrutor och anropar `_refresh` en gång, inte en gång per art. Är ingen art ikryssad visar `_refresh` redan "Tick at least one species in the list." och Starta-knappen är avstängd, som tidigare. Test i `tests/test_plugin_load.py` (körs i CI mot QGIS). Texterna är i `species_data/i18n/species_data_sv.ts`/`.qm`, handledningen och `CHANGELOG.md` är uppdaterade.

## 2. Sök inom en polygon eller en ruta som användaren ritar på kartan

**Problem.** Fliken Geografi på Hämta har bara *Aktuell kartutsträckning*, *Län* och *Hela Sverige*. Man kan inte avgränsa sökningen till ett eget område, till exempel ett reservat eller ett avrinningsområde.

**Önskemål.** Användaren ritar en polygon på kartan, eller drar upp en ruta, och sökningen görs inom det området.

### Rutan — klart (2026-11-24)

Mats valde att börja med bara rutan; polygonen återstår (se nedan). Byggt:
- Ny radioknapp **Drawn rectangle** på fliken Geografi, med knapparna **Draw rectangle** och **Clear** och en etikett som visar ytan i hektar (`species_data/dockwidget.py`: `_geography_tab`, `_start_draw_rectangle`, `_set_drawn_rectangle`, `_clear_drawn_rectangle`).
- Kartverktyget är `_RectangleTool` (ärver `QgsMapToolExtent`), som drar en ruta, återställer det tidigare aktiva verktyget när musknappen släpps eller vid Esc (Esc ritar ingenting).
- Arean räknas med `QgsDistanceArea` (geodetisk, oberoende av kartans projektion), inte bara planar SWEREF 99 TM-area som först föreslaget — mer korrekt och redan den metod QGIS egna mätverktyg använder.
- Taket **100 hektar** (`query.MAX_DRAWN_AREA_HA`, beslutat av Mats 2026-11-24) kontrolleras i `_search_hint`/`_can_search`: en för stor ruta stänger av Räkna/Hämta-knapparna och visar varför.
- Rutan görs om till `bbox_wgs84` precis som *Aktuellt kartutsnitt* (`_rectangle_bbox`, delad kod med `_canvas_bbox`) — samma `BoundingBoxWgs84`, samma `_bbox`-filter i `query.py`, ingen URL-längdsrisk eftersom en ruta bara har fyra hörn.
- Test i `tests/test_plugin_load.py` (`test_drawn_rectangle_area_is_capped_and_feeds_the_query`), körs i CI mot QGIS. Kan inte köras lokalt (ingen QGIS-installation i den här miljön) eller testas med en riktig musdragning — **Mats bör pröva rita/rensa/Esc i riktig QGIS innan nästa version.**
- Handledningen (avsnittet om Geografi) och `CHANGELOG.md` uppdaterade. Texterna är i `species_data/i18n/species_data_sv.ts`/`.qm`.

### Polygonen — återstår

**Det som redan finns.** `Query` kan redan ta `polygon_wgs84` och göra om den till ett WFS-filter (`_intersects` i `species_data/query.py`), som tar en enkel polygon med hål och avvisar andra geometrier. Det som saknas är ritverktyget för polygon och raden på fliken Geografi.

**Att bygga**
- Ett polygonverktyg med `QgsRubberBand` (rutans `_RectangleTool`-mönster — återställ föregående verktyg, Esc avbryter — håller inte rakt av för en polygon, som behöver ett eget avslut). Avsluta med högerklick (eller dubbelklick), avbryt med Esc.
- En rad *Rita polygon* bredvid rutans knappar.
- Omräkning från kartans CRS till WGS 84 med `QgsCoordinateTransform`, som redan görs för rutan.
- Samma 100-hektarstak som rutan, med samma `QgsDistanceArea`-beräkning.
- Metadata: `provenance.py` beskriver `bbox_wgs84` men inte `polygon_wgs84`. Lagrets metadata och abstract ska säga vilket område som användes.
- **POST istället för GET.** Provat 2026-11-24 live mot `sosgeo.artdata.slu.se`: felet är inte polygonens area, utan URL-längden när filtret skickas som GET-frågesträng. Ett GET-anrop går sönder vid omkring 160 hörn (~7,3 kB total URL, cirkel med radie 50 m) — över det ger servern `400 Request header is too large` (Tomcat) eller `414 Request-URI Too Large` (nginx framför). Samma filter som en WFS 1.0.0 `GetFeature`-POST (XML-kropp istället för frågesträng, `wfs:GetFeature`/`wfs:Query` runt `ogc:Filter`) klarade 2000, 5000 och 20 000 hörn (upp till ~740 kB kropp) utan problem. `species_data/net.py` använder idag bara `QgsBlockingNetworkRequest.get`; för polygonfiltret behövs ett POST-anrop med XML-kropp istället (`QgsBlockingNetworkRequest.post`, fortsätt med GET för de andra frågorna och växla bara när `polygon_wgs84` är satt). Ingen förenkling av hörnen (`QgsGeometry.simplify`) behövs för att klara en liten ritad yta.

**Frågor att avgöra först**
- Ska det ritade området kunna kombineras med län, eller ersätta det? Förslag: ersätter, som *Aktuell kartutsträckning* gör idag, så att valen går ut på samma sätt. (Gäller även rutan, redan löst så: radioknapparna är ömsesidigt uteslutande.)
- Ska användaren kunna välja en befintlig polygon i ett lager (markerat objekt)? Det är enklare än att rita och vanligt i QGIS. Kan tas med som ett tredje alternativ, men det är inte önskat än.
- Träffar polygonen punkter vars osäkerhetscirkel går utanför området? `Intersects` på fyndets punkt gör det inte. Det ska stå i handledningen, eftersom användare annars kan tro att allt inom cirkeln finns med. (Gäller även rutan — bör nämnas i handledningen redan nu.)
- Hur ser kartverktyget ut för en användare som redan har ett annat verktyg aktivt, och vad händer om pluginet stängs medan verktyget är på? (Rutans verktyg återställer föregående verktyg vid släpp/Esc; vad som händer om pluginet stängs eller panelen döljs medan verktyget är aktivt är inte testat.)

**Test.** `Query` med polygon ger rätt XML (finns delvis). Nytt: kartverktygets resultat blir en giltig `polygon_wgs84`, och att POST-anropet går till samma plats som GET gjorde.

**Dokumentation.** Handledningen (avsnittet om Hämta, fliken Geografi), `CHANGELOG.md`, och en rad i README. Metodbeskrivningarna berörs bara om valet av område förändrar vad som kallas en lokal, vilket det inte gör.

## 3. Svar från SLU Artdatabanken om namn, logotyp och beskrivning (inkommet 2026-11-11)

**Status.** Mats beslutade 2026-11-24 att **inte** byta pluginets namn ändå, efter att ha provat namnbytet samma dag (se nedan). Namnet är **"Species Data (SLU Artdatabanken)"**, precis som innan. GitHub-repot heter fortfarande `species-data-sweden-qgis-plugin` (det kvarstår, se nedan) och beskrivningstexterna SLU bad om är fortfarande ändrade. Svaret kom från Martin på SLU Artdatabanken efter att han provat pluginet och stämt av med chef och förvaltningsansvarig.

**Vad SLU sa**
- *Bra som det är:* README och handledningen säger tydligt att pluginet är fristående och varken utvecklas, ägs eller stöds av SLU eller SLU Artdatabanken, och anger varifrån data hämtas.
- *Logotyp:* SLU:s och SLU Artdatabankens logotyper får inte användas i pluginet eller för att presentera det, eftersom de kan ge intryck av att SLU står bakom eller samarbetar kring produkten. Kontrollera ikonen (`icon` i `species_data/metadata.txt`, bildfilen i paketet) och eventuella bilder i README och handledningen.
- *Namn:* SLU föreslår att namnet "Species Data (SLU Artdatabanken)" och GitHub-repositoryt byts, eftersom namnen i sig kan ge intryck av ett officiellt tillägg, särskilt i QGIS tilläggskatalog. Förslag: **Species Data (Sweden)** och repositorynamnet `species-data-sweden-qgis-plugin`. Annat namn är fritt. Att nämna SLU Artdatabanken, Artportalen och Species Observation System som datakälla är fortsatt tillåtet.
- *Beskrivning av datakällan:*
  - Handledningen: byt "Fynddata hämtas från SLU Artdatabankens Species Observation System (Artportalen) enligt deras API-villkor." till "Fynddata hämtas från SLU Artdatabankens Species Observation System enligt deras API-villkor. Pluginet använder bara fynd från datasetet Artportalen." Skälet: parentesen får Species Observation System och Artportalen att se ut som samma sak.
  - GitHubs huvudsida (repository-beskrivningen): "Artportalen data" blir "Artportalen dataset", alltså "A QGIS plugin for building dot maps of vascular plant observations from the Artportalen dataset held by SLU Artdatabanken, …".

**Namnbytet — provat och sedan återställt (2026-11-24)**
Namnet byttes till "Species data (Sverige)" överallt (kod, metadata, i18n, README, handledningen, `CHANGELOG.md`), men återställdes till **"Species Data (SLU Artdatabanken)"** samma dag, av två skäl:
1. Det visade sig att ett redan publicerat plugins **visningsnamn** inte går att ändra bara genom att höja `name` i `metadata.txt` och släppa en ny version. plugins.qgis.org:s egen FAQ (`https://plugins.qgis.org/docs/faq`, kontrollerad 2026-11-24) säger att det kräver ett separat mejl till QGIS Plugin Administration-teamet, med verifiering. Mats bekräftade (2026-11-24) att pluginet redan är publicerat där, i en tidigare version — så namnbytet hade krävt den extra, långsamma processen (utöver den redan flera dagar långa godkännandetiden för en ny version).
2. Mats vill egentligen inte byta namn: "Species Data (SLU Artdatabanken)" harmoniserar med hans andra plugins (samma mönster: beskrivande del, källan i parentes), och den extra processen är inte värd besväret.

**Paketets id** (`package_name`, här `species_data`) går aldrig att byta (QGIS kan då tolka pluginet som avinstallerat och ge installationsfel) — det var och är oförändrat, ingen risk där oavsett.

**Vad som ändå kvarstår från namnbytes-försöket**
- GitHub-repot heter sedan 2026-11-15 `species-data-sweden-qgis-plugin` (gamla länkar vidarebefordras). Det byts **inte** tillbaka — det är inget som kräver en separat, långsam process, och stör inget.
- Repository-URL:erna i `metadata.txt` (`homepage`, `repository`, `tracker`) pekar på det nya reponamnet.
- Beskrivningstexterna SLU bad om är fortfarande ändrade: handledningens mening är uppdelad, och "Artportalen data" är "Artportalen dataset" i README och metadata.
- Oberoende av namnfrågan: fristående-texten är gjord tydligare och kommer tidigare i README, handledningen, panelens egen text och `metadata.txt`s `about`-fält (se nedan), som en kompensation för att namnet ändå behålls.

**Ikonen** (`species_data/icon.svg`): kontrollerad 2026-11-24, en egen generisk blomikon utan SLU-element. Inget att ändra, men svara SLU att den inte innehåller deras logotyp.

**Svar till Martin.** Redan skickat 2026-11-15 (se `docs/SOURCES.md` och `docs/utveckling/findings.md`), och det mejlet lovar ett namnbyte till "Species Data (Sverige)" / "Artdata (Sverige)" som alltså **inte** blir av. **Mats behöver skicka en uppföljning till Martin** som förklarar att namnet står kvar (med skälen ovan), eftersom mejlet annars säger fel sak. Inte gjort än.

**Dokumentation.** Ändrar namnet eller beskrivningen något som metodbeskrivningarna citerar ska de hållas i takt med koden (CLAUDE.md). Posten i `CHANGELOG.md` och i `species_data/metadata.txt` ska vara lika.

## Redan känt och öppet (från tidigare, inte ändrat här)

- En kolumn för källa (`sources`) på samvariationslagret, som lokallagret har. Erbjudet, inte beställt.
- Frågor till SLU om licens per dataset, dubbletter mellan datakällor och en lista över leverantörer.
- Mer än kärlväxter och fler källor: se [kallor-och-organismgrupper.md](kallor-och-organismgrupper.md). iNaturalist har låg prioritet.
- Engelska kolumnnamn i exporten från Fynddata: kräver en export på engelska att jämföra med.
- Skärmbild till README (läggs i `docs/images/`), som Mats tar själv.
