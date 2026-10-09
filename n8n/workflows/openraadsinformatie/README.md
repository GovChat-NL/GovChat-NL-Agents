# Open Besluitvorming: ingestie en zoeken

Deze map bevat uitsluitend de actuele, productiegerichte workflows voor het indexeren en doorzoeken van openbare Open Besluitvorming-documenten.

## Actuele workflows

| Bestand | Functie |
| --- | --- |
| [`openbesluitvorming-document-worker.json`](openbesluitvorming-document-worker.json) | Haalt documentdetails op, chunkt tekst, maakt embeddings, schrijft idempotent naar Qdrant en bewaart een canonieke openbare bronlink. |
| [`openbesluitvorming-ingest-page-runner.json`](openbesluitvorming-ingest-page-runner.json) | Verwerkt één duurzame snapshotpagina per uitvoering, maakt een collectie aan indien nodig en bewaart een checkpoint in Qdrant. |
| [`openbesluitvorming-search.json`](openbesluitvorming-search.json) | Subworkflow voor de orchestrator-tool `zoeken_in_openbesluitvorming`; voert metadatafilters, datumselecties en relevante documentzoekopdrachten uit. |

Oudere validatie-, live-export- en versievarianten zijn bewust uit deze map verwijderd. Gebruik alleen bovenstaande drie workflows als bron voor verdere ontwikkeling.

## Doel en afbakening

De keten indexeert momenteel openbare **documenten** uit Open Besluitvorming in Qdrant. De GovChat-orchestrator gebruikt [`openbesluitvorming-search.json`](openbesluitvorming-search.json) als standaard bronzoektool via [`orchestrator-litellm.json`](../../orchestrator-litellm.json).

De tool is geschikt voor gerichte vragen over openbare besluiten, vergaderstukken, documenten en onderwerpen. Het is geen volledige archiefexport, juridisch register, bron voor niet-openbare informatie of algemene corpus-analyseomgeving.

## Zoeken vanuit de orchestrator

De agent vult de vooraf gedefinieerde toolvelden afzonderlijk in. Metadata hoort niet in het vrije zoekveld `query`.

| Veld | Gebruik |
| --- | --- |
| `query` | Kort inhoudelijk onderwerp of trefwoorden. Verplicht voor normale semantische zoekopdrachten. |
| `source_key` | Open Besluitvorming-bronsleutel van één organisatie, bijvoorbeeld `provincie_limburg`. Zonder organisatiekeuze gebruikt de tool Provincie Limburg. |
| `date_from` | Inclusieve begindatum in `YYYY-MM-DD`. |
| `date_to` | Inclusieve einddatum in `YYYY-MM-DD`. |
| `document_type` | Entiteittype: `document`, `meeting`, `motion` of `recording`. Leeg betekent de standaardwaarde `document`. |
| `document_classification` | Aanvullende documentmetadata, bijvoorbeeld `Moties`, `Amendementen`, `Bijlage`, `Ingekomen stukken`, `Schriftelijke vragen van Statenleden` of `Uitnodigingen`. Gebruik deze wanneer expliciet om een stukcategorie wordt gevraagd. |
| `supplier` | Bronplatform/leverancier, bijvoorbeeld `ibabs` of `notubiz`; alleen gebruiken wanneer expliciet gevraagd. |
| `meeting_id` | Exacte UUID van een vergadering; alleen gebruiken wanneer deze zeker bekend is. |
| `document_id` | Exacte UUID van een document; alleen gebruiken wanneer deze zeker bekend is. |
| `include_content` | `false`: alleen documentmetadata, kort fragment en bronlink. `true`: voeg ook de opgeslagen documentinhoud toe voor inhoudelijke analyse; gebruik dit alleen wanneer metadata en fragment niet voldoende zijn, omdat de tooluitvoer groter wordt. |

Gebruik restrictieve filters alleen wanneer de gebruiker ze noemt of wanneer ze eenduidig uit de vraag volgen. Een onzeker filter kan relevante documenten uitsluiten.

### Beschikbare entiteittypen

`document_type` is een filter op het Open Besluitvorming-entiteittype en kent vier waarden:

- `document` — standaardwaarde; momenteel de enige inhoudelijk geïndexeerde entiteit.
- `meeting` — toekomstige vergadering-entiteit.
- `motion` — toekomstige motie-entiteit.
- `recording` — toekomstige opname-entiteit.

De huidige vectorcollectie bevat 14.552 `Document`-punten en één technisch checkpointpunt. Er zijn momenteel geen afzonderlijk geïndexeerde `Meeting`, `Motion` of `Recording`-punten. Daarom moet de agent `document_type` leeg laten of `document` gebruiken, tenzij een toekomstige ingestie deze andere entiteiten daadwerkelijk toevoegt.

De payload bevat daarnaast het aparte metadata-veld `classification`, met waarden zoals `Bijlage` of `Moties`. Dit veld is via de agentparameter `document_classification` te filteren en mag niet worden verward met `document_type`.

### Relevantiezoekopdracht

Voor een gerichte onderwerpvraag:

1. Qdrant levert maximaal **500 kandidaten**.
2. De workflow behoudt maximaal **100 documenten** die door een adaptieve relevantiedrempel komen.
3. De response bevat onder meer `title`, `date`, `source_key`, `classification`, `meeting_id`, `document_id`, `excerpt` en `public_url`.
4. De agent citeert `public_url` als compacte Markdown-link en niet als lange kale URL.

De agent mag niet beweren dat de gevonden set volledig of uitputtend is. Bij ieder antwoord op basis van deze zoektool hoort de volgende begrijpelijke dekkingsmelding:

> Ik heb de beschikbare geïndexeerde openbare documenten zo breed mogelijk doorzocht. Toch kunnen documenten ontbreken, bijvoorbeeld door vertraging in publicatie, indexering of zoektermen.

### Datumoverzicht zonder onderwerp

Een onderwerp is normaal verplicht. De gecontroleerde uitzondering is een zuivere datumselectie met `date_from` en/of `date_to`:

- Bij maximaal **100 documenten** retourneert de tool alle geselecteerde documenten in chronologische volgorde.
- Bij meer dan 100 documenten retourneert de tool `topic_required`. De agent vraagt dan om een onderwerp toe te voegen of de periode te verkleinen.
- De grens wordt gecontroleerd met een Qdrant-scroll van 101 punten; het is geen semantische schatting.

Een lege zoekuitkomst is nooit bewijs dat er geen besluit of document bestond. De oorzaak kan ontbreken in de bron, vertraagde publicatie, indexering, metadata of afwijkende terminologie zijn.

## Bronlinks

Nieuwe documentpunten bevatten een canonieke openbare URL in `public_url`. Voor Provincie Limburg is dit, wanneer de vergadering- en document-ID beschikbaar zijn:

```text
https://limburg.bestuurlijkeinformatie.nl/Agenda/Document/{meeting_id}?documentId={document_id}
```

De zoekworkflow kan deze URL voor oudere punten ook afleiden uit de aanwezige IDs.

## Eerste gebruik en standaardbron

Een lege vectorstore wordt niet stilzwijgend vanuit een chat gevuld. Als de collectie ontbreekt of geen doorzoekbare punten bevat, retourneert de zoektool `bootstrap_required`.

| Omgevingsvariabele | Standaardwaarde | Doel |
| --- | --- | --- |
| `N8N_ENABLE_OPENBESLUITVORMING` | `true` | Zet zoeken in Open Besluitvorming aan of uit. Bij `false` geeft de tool een duidelijke `disabled`-response terug en voert zij geen retrieval uit. |
| `OPENBESLUITVORMING_COLLECTION` | `openraadsinformatie_v1` | Gedeelde Qdrant-collectie. |
| `OPENBESLUITVORMING_DEFAULT_SOURCES` | `provincie_limburg` | Eén of meer kommagescheiden standaardbronsleutels wanneer de agent geen `source_key` invult. |
| `OPENBESLUITVORMING_DEFAULT_SOURCE` | `provincie_limburg` | Achterwaarts compatibele enkelvoudige standaardbron; wordt gebruikt wanneer `OPENBESLUITVORMING_DEFAULT_SOURCES` ontbreekt. |
| `OPENBESLUITVORMING_QDRANT_URL` | `http://qdrant:6333` | Interne Qdrant-endpoint. |
| `OPENBESLUITVORMING_EMBEDDING_MODEL` | `govchat-embedding` | LiteLLM-embeddingalias met 3072 dimensies. |

Start voor een schone omgeving [`openbesluitvorming-ingest-page-runner.json`](openbesluitvorming-ingest-page-runner.json) handmatig. Deze maakt de collectie aan wanneer nodig, verwerkt één begrensde pagina en schrijft een duurzaam checkpoint. Controleer eerst de eerste pagina, metadata en puntenaantallen voordat een planning wordt geactiveerd.

Een collectie met een andere vectorgrootte of distance-metric faalt bewust snel, zodat vectoren van verschillende modellen niet worden gemengd.

## Ingestie-eigenschappen

- De page-runner werkt één exportpagina per uitvoering af en hervat via een Qdrant-checkpoint.
- De documentworker beperkt werk, tekst en retries per document.
- Documentpunten zijn idempotent: dezelfde bronentity overschrijft hetzelfde Qdrant-punt.
- HTTP-aanroepen hebben time-outs en tijdelijke fouten krijgen begrensde retries.
- Fouten worden als compacte dead letters vastgelegd; individuele fouten blokkeren de pagina niet.
- De Qdrant REST API wordt rechtstreeks gebruikt; er is geen n8n-Qdrant-credential nodig.

## Grenzen voor de agent

De agent moet transparant begrenzen:

- Geen complete inventaris, uitputtende tijdlijn of betrouwbare totaaltelling claimen buiten de gecontroleerde datumselectie van maximaal 100 documenten.
- Bij ontbrekende jaren aangeven dat dit een dekkings- of indexeringshiaat kan zijn; niet suggereren dat er niets is gebeurd.
- Geen vergelijkingen tussen organisaties maken wanneer dekking voor beide bronnen niet aantoonbaar is.
- Bij dubbelzinnige entiteiten of afkortingen eerst verduidelijking vragen, bijvoorbeeld of `MAA` Maastricht Aachen Airport betekent, en vragen of in Open Besluitvorming gezocht moet worden.
- Gevonden documenten niet behandelen als bewijs van de actuele juridische status; verwijs daarvoor naar de bevoegde instantie of het actuele officiële register.
- Duidelijk maken dat private, verwijderde en niet-openbare documenten niet toegankelijk zijn.
- Voor brede corpusvragen een afgebakende beoordeling voorstellen op onderwerp, periode, organisatie of documenttype.

## Toekomstige ontwikkeling

### Volledige corpuscontrole en analyse

Voor aantoonbaar volledige inventarissen, reproduceerbare tellingen of tijdlijnen over grote perioden is een aparte export- of analyseworkflow nodig. Die moet volledig pagineren, dekking per bron/periode rapporteren en resultaten reproduceerbaar opslaan.

### Hybride retrieval

Voor exacte namen, acroniemen, dossiernummers en aliassen is een aanvullende lexicale zoeklaag wenselijk. Een toekomstige hybride laag kan keyword/full-text-kandidaten combineren met vector-kandidaten en daarna reranken. Dit is geschikter voor echte Boolean-`OR`-semantiek dan één lange embeddingquery.

### Statistieken en aggregaties

De huidige zoektool is gericht op relevante documenten en levert daarom geen betrouwbare corpusbrede statistieken. Een doorontwikkeling kan een aparte, reproduceerbare statistiek- of aggregatieworkflow toevoegen die rechtstreeks over de volledig gefilterde collectie telt en groepeert.

Voorbeelden van gewenste statistieken:

- aantal documenten per jaar, organisatie of bronplatform;
- aantal documenten per `document_classification` per jaar, bijvoorbeeld het aantal `Moties`, `Amendementen` of `Ingekomen stukken`;
- aantal documenten per bestuursorgaan, vergadering of agenda wanneer die entiteiten later zijn geïndexeerd;
- ontwikkeling in documentvolumes binnen een onderwerp, met duidelijke scheiding tussen volledige tellingen en semantische zoekresultaten;
- dekkingsoverzicht per organisatie en periode, inclusief ontbrekende jaren en indexeringsstatus.

De statistiekworkflow moet altijd de gebruikte filters, de telmethode, de datumbasis en de brondekking teruggeven. Daarmee kan de agent bijvoorbeeld verantwoord antwoorden op “hoeveel moties zijn er per jaar?”, zonder een semantische topresultatenset ten onrechte als volledige telling te behandelen.

### Ingestie van vergaderingen, agenda's en besluiten

De huidige keten indexeert hoofdzakelijk documententiteiten. Een volgende stap is de ingestie van **vergaderingen**, **agenda's**, **agendapunten** en, waar publiek beschikbaar, **besluitrelaties**.

Daarmee worden vragen mogelijk zoals:

- “Welke vergaderingen over woningbouw staan volgende maand gepland?”
- “Wat staat op de agenda van de Statenvergadering van 12 juni?”
- “Bij welk agendapunt hoort dit Statenvoorstel?”
- “Welke documenten en besluiten horen bij vergadering X?”

Aanbevolen metadata:

| Entiteit | Belangrijke metadata |
| --- | --- |
| Vergadering | `meeting_id`, datum/tijd, organisatie, bestuursorgaan, titel, status en openbare vergaderlink. |
| Agenda | `agenda_id`, `meeting_id`, datum, titel, agenda-URL en volgorde. |
| Agendapunt | `agenda_item_id`, `meeting_id`, `agenda_id`, nummer, titel, onderwerp, status en openbare link. |
| Document | `document_id`, `meeting_id`, eventueel `agenda_item_id`, classificatie, datum en `public_url`. |
| Besluit | `decision_id`, `meeting_id`, eventueel `agenda_item_id`, besluitdatum, tekst/samenvatting, status en openbare link. |

Deze entiteiten moeten stabiele IDs en relaties behouden. Voor agenda- en vergaderoverzichten is een gestructureerde datumgesorteerde queryroute nodig; alleen semantische retrieval is daarvoor niet voldoende.

### Wijzigingssynchronisatie

Na een volledige snapshot moet de keten de `X-Changes-Cursor` van de eerste snapshotresponse bewaren, daarna `/api/export/changes` pollen en de cursor alleen verhogen wanneer de hele pagina succesvol is verwerkt of duurzaam voor replay is vastgelegd. Voor `upsert` kan de bestaande documentworker worden hergebruikt; voor toekomstige vergadering-, agenda- en besluitentiteiten zijn entiteitspecifieke workers nodig.
