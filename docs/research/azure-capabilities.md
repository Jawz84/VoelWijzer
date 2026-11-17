# Azure-mogelijkheden en gegevensverwerking verifiëren

**Onderzoeks- en raadpleegdatum: 2026-11-17.** Afgebakend bronnenonderzoek, geen implementatie-, leveranciers- of productbesluit.

## Reikwijdte en methode

Uitgangspunt uit de opdracht: een lokale, Nederlandstalige reflectietool voor één gebruiker; externe AI uitsluitend na opt-in en via een server. De applicatiebackend bewaart of logt reflecties niet permanent. Alle niet-AI-paden blijven zelfstandig functioneren. Dit zijn eisen, geen hier getoetste implementatiefeiten.

Alleen openbare primaire Microsoft- en OpenAI-bronnen geraadpleegd; zoekopdrachten waren generiek en bevatten geen privéteksten. Geen Azure-resources, abonnementen, credentials of eigen endpoints benaderd; geen provisioning, SDK-installatie of uitvoering van voorbeeldcode. Onderstaande bevestigingen betreffen de daadwerkelijk gelezen documentatie, niet een live beschikbaarheids- of werkingstest. Ontbrekende documentatie bewijst noch beschikbaarheid, noch onbeschikbaarheid.

## Geverifieerde feiten

### Model, API en client afzonderlijk

| Onderwerp | Wat de bron bevestigt | Begrenzing |
|---|---|---|
| Bestaan van het model | OpenAI documenteert expliciet **GPT-6.1 Sol**, model-ID `gpt-6.1-sol`, met Responses, streaming en structured outputs. [1] | OpenAI-ondersteuning is op zichzelf geen Azure-bewijs. |
| Azure-modelondersteuning | Microsoft noemt afzonderlijk `gpt-6.1-sol`, versie **2026-09-29**, als Azure OpenAI-model met Responses, streaming, structured outputs en instelbare reasoning effort/verbosity. Ook de Azure Responses-lijst noemt dit exacte model en deze versie. [2][3][4] | Geen bevestiging dat een bepaald abonnement het nu kan deployen of aanroepen. |
| Responses API | Azure documenteert de v1 Responses API, inclusief generatie, streaming, opgeslagen responses ophalen/verwijderen en stateless gebruik. [3] | API-regio, modelversie en deploymenttype moeten samen passen. Een generiek API-voorbeeld bewijst niet elke modelcombinatie. |
| Streaming | Azure toont `stream: true`, tekstdelta-events en C#-voorbeelden. Een fout midden in de stream kan bij HTTP 200 alsnog als fout-event arriveren. [3] | Een gedeeltelijke stream is geen gegarandeerd volledig antwoord of volledig JSON-document. |
| Structured outputs | Azure documenteert JSON Schema via `text.format` voor Responses; Chat Completions gebruikt `response_format`. Het exacte Sol-model heeft expliciete featurebevestiging. [2][4][5] | Schema-ondersteuning is niet hetzelfde als inhoudelijke juistheid; valideer het eindresultaat en behandel weigering, afbreking en fouten apart. |
| Reasoning | OpenAI noemt voor Sol `low`, `medium` (standaard), `high`, `xhigh`, `max`. Azure bevestigt modelgebonden reasoning-ondersteuning; de Azure-gids beschrijft `reasoning.effort`, met `max` voor GPT-6 alleen via Responses. Reasoning-tokens tellen mee voor kosten en uitvoerlimieten. [1][4] | De algemene keuzelijst bevat ook waarden voor andere modellen. Daaruit volgt niet dat Sol `none` of `minimal` accepteert. Exacte Azure-defaults en parametercombinaties blijven later te toetsen. |
| .NET-haalbaarheid | Microsoft toont Responses en streaming met het NuGet-pakket `OpenAI`, `Azure.Identity` en Entra-authenticatie. OpenAI onderhoudt de officiële .NET-library in samenwerking met Microsoft. [3][11] | Documentair haalbaar, niet gebouwd of getest. De gelezen gidsen gebruiken verschillende clienttypen (`ResponsesClient` versus `OpenAIResponseClient`) en experimentele `OPENAI001`-onderdelen: pakketversie en API-oppervlak moeten later expliciet worden vastgezet. [3][4] |
| Lokale identiteit | Azure Identity ondersteunt lokaal aangemelde ontwikkelaarsaccounts, onder meer via Visual Studio of Azure CLI. Een token vervangt een API-key, maar verleent niet vanzelf rechten. [8][10] | Aanmelden, tenantkeuze en passende inference-RBAC zijn latere randvoorwaarden; hier niet uitgevoerd. |
| Managed identity | Voor een Azure-gehoste server documenteert Microsoft system-/user-assigned managed identity zonder zelf beheerd applicatiegeheim. Azure OpenAI beschrijft zowel ontwikkelaarsidentiteit als managed identity en bijbehorende rollen. [9][10] | Dit is niet automatisch de identiteit van een lokale desktopapp. Hostkeuze en autorisatie zijn niet besloten. |

### Regio en deployment

- De publieke modelmatrix vermeldt Sol-versie `2026-09-29` onder **Global Standard** en **Data Zone Standard**, onder andere in `westeurope`, `swedencentral` en `francecentral`. Die regio's staan ook in de Responses-regiolijst. De matrix vermeldt daarnaast Global Provisioned en Data Zone Provisioned, onder andere in `westeurope`. Dit zijn gedocumenteerde combinaties, geen capaciteitsreserveringen. [3][6]
- Generiek onderscheid: Standard betaalt per token; Provisioned reserveert capaciteit. Global kan buiten Europa verwerken, ook bij een Europese resource. Data Zone beperkt verwerking tot de Microsoft-gedefinieerde zone; opgeslagen gegevens en inferentieverwerking hebben afzonderlijke locatievoorwaarden. Niet ieder model ondersteunt ieder deploymenttype. [7][12]
- **Bronnuance:** de privacytekst beschrijft EU-DataZone als EU-lidstaten; de deploymentgids beschrijft de EU Data Boundary inclusief mogelijke EFTA-landen zoals Noorwegen en Zwitserland. Ook spreekt de regiomatrix bij Standard/Regional over een regio, waar privacy- en deploymenttekst ruimte noemen voor verwerking binnen de geografie. Daarom geen belofte “uitsluitend Nederland”, “uitsluitend deze regio” of “uitsluitend EU-lidstaten” zonder nadere bevestiging. [6][7][12]
- Voor Sol is in de gelezen Standard/Regional-tabellen geen rij aangetroffen. Dat is **geen** vaststelling van onbeschikbaarheid; die optie blijft onbevestigd. [6]

## Gegevensverwerking: vier verschillende lagen

| Laag | Geverifieerd of expliciet uitgangspunt | Betekenis voor de reflectietool |
|---|---|---|
| **Applicatieopslag en -logging** | Geen permanente backendopslag/-logging is een opdrachteis, geen Azure-eigenschap. | Later ook infrastructuurlogs, HTTP-bodylogging, tracing, foutmeldingen en dumps beoordelen. Een eigen “niet opslaan”-beleid schakelt providerprocessen niet uit. |
| **Provider-API-state** | Azure Responses bewaart responsegegevens standaard **30 dagen**. `store=false` ondersteunt stateless aanroepen; de applicatie levert vervolgcontext zelf aan. Azure background mode vereist `store=true`. [3] | Standaardgedrag past niet zonder meer bij minimale bewaring. `store=false` is een afzonderlijk te beoordelen instelling, geen universele zero-retentiongarantie. Opgeslagen responses verwijderen is mogelijk, maar bewijst niet dat alle andere lagen zijn gewist. |
| **Provider-caching** | Azure documenteert tijdelijke promptcaches, standaard impliciete caching voor GPT-5.6 en later en een standaard-TTL van `30m` als **minimum**, niet als maximum. Voor ondersteunde Standard-deployments gebruikt `explicit` zonder breakpoints geen promptcache. [13] | Ook bij `store=false` caching afzonderlijk beoordelen. Geen maximale Sol-bewaartermijn afleiden uit deze minimum-TTL of uit regels voor oudere modellen. |
| **Abuse monitoring en contractuele uitzonderingen** | Azure kan verdachte prompts/outputs automatisch beoordelen; daarbij wordt volgens Microsoft geen inhoud in het abuse-monitoringsysteem opgeslagen. Voor noodzakelijke menselijke beoordeling kan inhoud wel worden opgeslagen en toegankelijk zijn voor bevoegde Microsoft-medewerkers. Modified abuse monitoring vereist geschiktheid, aanvraag en goedkeuring; dan vervallen de beschreven opslag en menselijke review, maar automatische beoordeling kan blijven plaatsvinden. [12][14] | Niet gelijkstellen aan applicatielogging of Responses-state. Niet aannemen dat deze uitzondering beschikbaar of actief is voor dit abonnement/model. De gelezen Azure-bronnen geven geen algemene maximale bewaartermijn voor deze menselijke-reviewopslag; de 30 dagen van Responses daarop niet projecteren. |

Aanvullend:

- Microsoft stelt voor **Models sold by Azure** dat prompts en outputs niet beschikbaar zijn voor OpenAI of andere klanten en niet zonder toestemming/instructie voor training van generatieve basismodellen worden gebruikt. De modellen worden door Microsoft in Azure gehost; dit is niet de rechtstreekse OpenAI API. “Geen training” betekent dus niet “geen verwerking/opslag”. [12]
- Opgeslagen stateful gegevens worden volgens Microsoft versleuteld bewaard in de geografie van de resource; Responses gebruikt servicebeheerde opslag, geen eigen meegebrachte opslag. Previewvoorwaarden kunnen afwijken. [12]
- OpenAI beschrijft voor **zijn eigen API** afzonderlijk abuse-logs tot 30 dagen, standaard Responses-state van 30 dagen en goedkeuringsgebonden Zero Data Retention/Modified Abuse Monitoring met uitzonderingen. Deze OpenAI-regels zijn **niet** automatisch Azure-regels of een Azure-contractoptie. [15]
- De Microsoft-privacybron verwijst naar de DPA. De publieke DPA-index biedt een editie november 2026 aan; de toepasselijke overeenkomst, Product Terms en eventuele aanvullende goedkeuringen voor deze gebruiker zijn niet juridisch of abonnementspecifiek vastgesteld. [12][16]

## Niet-onderbouwde claims

De volgende uitspraken mogen niet als conclusie worden overgenomen:

- “Sol bestaat niet/is niet op Azure” — strijdig met expliciete primaire modeldocumentatie. [1][2]
- “Omdat Responses dit kan, werkt elke functie op elk model, in elke regio en via iedere SDK-versie” — modelondersteuning en deployment-/clientondersteuning zijn afzonderlijk. [2][3][6]
- “Onze backend logt niets, dus Microsoft bewaart niets” of “`store=false` schakelt alle opslag, caching en menselijke review uit” — verwart de lagen hierboven. [3][12][13][14]
- “Een West-Europese resource garandeert uitsluitend Nederlandse/EU-verwerking”, “geen training betekent zero retention” of “modified monitoring is standaard inbegrepen” — niet gerechtvaardigd door de bronnen. [7][12][14]

## Open punten en implicaties voor latere besluitvorming

1. **Tenant/subscription/deployment:** exact model en versie, toegestane cloud/regio/SKU, toegang, quota, actuele capaciteit, kosten en lifecycle controleren. Publieke tabellen zijn geen subscription-check. Hier niets aangevraagd of gecontroleerd.
2. **Technische verificatie na afzonderlijke toestemming:** compatibele .NET-versies, Entra-identiteit/RBAC, Azure-parameters/defaults, combinatie streaming + schema, annulering, weigeringen en incomplete antwoorden testen met synthetische gegevens. Geen werkende integratie claimen.
3. **Privacyvoorwaarde concretiseren:** “geen permanente applicatieopslag” kan naast tijdelijke provideropslag bestaan. Voor verdergaande beloften moeten Responses-state, promptcache, abuse-monitoringbewaring, verwerkingslocatie en contractuele uitzonderingen afzonderlijk worden bevestigd. De bronverschillen over de Europese zone blijven expliciet open.
4. **Geen productbesluit:** bronnen ondersteunen technische haalbaarheid, niet de geschiktheid, Nederlandse reflectiekwaliteit of privacyacceptatie van Sol voor deze toepassing. AI blijft opt-in via de server; ontbreken van toestemming, configuratie of AI-beschikbaarheid mag de niet-AI-paden niet blokkeren. Dat volgt uit de opdracht, niet uit een providerbelofte.

**Resultaat:** openbare mogelijkheden zijn bevestigd; live beschikbaarheid en een end-to-end zero-retention- of EU-only-garantie zijn niet vastgesteld. Geen onderzoeksblokkade voor deze notitie; genoemde punten zijn voorwaarden voor eventueel vervolgonderzoek.

## Primaire bronnen

Alle onderstaande URLs zijn **geraadpleegd op 2026-11-17**. Toegang op die datum is geen garantie dat elke pagina dezelfde dag is bijgewerkt. Dit document bevat eigen samenvattingen, geen overgenomen documentatie.

1. OpenAI, GPT-6.1 Sol-modelkaart: https://developers.openai.com/api/docs/models/gpt-6.1-sol
2. Microsoft, Azure-modeloverzicht, sectie GPT-6.1: https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure#gpt-61
3. Microsoft, Azure OpenAI Responses API: https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/responses
4. Microsoft, reasoning, modelmatrix en parameters: https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/reasoning
5. Microsoft, structured outputs: https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/structured-outputs
6. Microsoft, model-/versie-/regiomatrix per deploymenttype: https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure-region-availability
7. Microsoft, deploymenttypes en verwerkingslocaties: https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/deployment-types
8. Microsoft, lokale .NET-ontwikkelaarsidentiteit: https://learn.microsoft.com/en-us/dotnet/azure/sdk/authentication/local-development-dev-accounts
9. Microsoft, managed identity voor Azure-gehoste .NET-apps: https://learn.microsoft.com/en-us/dotnet/azure/sdk/authentication/system-assigned-managed-identity
10. Microsoft, Azure OpenAI Entra-authenticatie; portalinstructies expliciet classic: https://learn.microsoft.com/en-us/azure/foundry-classic/openai/how-to/managed-identity
11. OpenAI, officiële .NET-library: https://github.com/openai/openai-dotnet
12. Microsoft, gegevensverwerking, privacy en opslag: https://learn.microsoft.com/en-us/azure/foundry/responsible-ai/openai/data-privacy
13. Microsoft, promptcaching en retentie: https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/prompt-caching
14. Microsoft, abuse monitoring en uitzonderingen: https://learn.microsoft.com/en-us/azure/foundry/openai/concepts/abuse-monitoring
15. OpenAI, eigen API-datacontroles; uitsluitend ter afbakening: https://developers.openai.com/api/docs/guides/your-data
16. Microsoft, DPA-publicatie-index; geen contractbeoordeling: https://www.microsoft.com/licensing/docs/view/Microsoft-Products-and-Services-Data-Protection-Addendum-DPA
