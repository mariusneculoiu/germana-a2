# Eliq Drinks App — Arhitectura explicată

Acest document explică modul în care aplicația consumă configurația white-label și datele CocktailDB, responsabilitatea fiecărui layer și locul în care se integrează filtrarea server-side pe categorii.

Baseline-ul descris este commit-ul trimis `f22f9be`.

## Ideea principală

Brandurile și datele despre băuturi sunt două fluxuri independente:

```text
                    ELIQ ANGULAR APP
                           │
            ┌──────────────┴──────────────┐
            │                             │
     WHITE-LABEL CONFIG              DRINK DATA
     „Cum arată aplicația?”          „Ce băuturi afișăm?”
            │                             │
            ▼                             ▼
  JSON-uri servite de aplicație      CocktailDB API
  /config/brand-manifest.json        filter.php?a=Alcoholic
  /config/brands/citrus.json         lookup.php?i=15423
  /config/brands/midnight.json
```

Brandul controlează aspectul și comportamentul configurabil. CocktailDB furnizează băuturile și rețetele.

Ambele branduri folosesc aceleași endpointuri CocktailDB. Nu există endpointuri CocktailDB separate pentru Citrus și Midnight.

---

# 1. Pornirea aplicației

Să presupunem că utilizatorul deschide:

```text
https://app.example.com/?brand=citrus#/drinks
```

## Pasul 1 — Angular inițializează configurația

În `src/app/app.config.ts`, Angular înregistrează un application initializer:

```text
Angular bootstrap
  -> ConfigurationService.initialize()
  -> configurația ajunge ready sau error
  -> apoi se publică aplicația normală
```

Scopul este:

- să evităm un flash nebranduit;
- componentele să nu citească o configurație parțială;
- o eroare de configurație să poată fi afișată și reîncercată.

## Pasul 2 — Se încarcă manifestul

Aplicația cere propriul fișier static:

```http
GET /config/brand-manifest.json
```

Manifestul definește:

```json
{
  "defaultBrandId": "citrus",
  "brands": [
    {
      "id": "midnight",
      "configPath": "config/brands/midnight.json"
    },
    {
      "id": "citrus",
      "configPath": "config/brands/citrus.json"
    }
  ]
}
```

Rolul său este să stabilească:

- brandul implicit;
- brandurile permise;
- calea sigură către configurația fiecărui brand.

Valoarea `?brand=citrus` nu este interpolată direct într-un URL. Aplicația caută ID-ul în manifest și folosește numai path-ul deja permis.

## Pasul 3 — Se încarcă JSON-ul brandului

Pentru Citrus:

```http
GET /config/brands/citrus.json
```

Pentru Midnight:

```http
GET /config/brands/midnight.json
```

Acestea sunt fișiere JSON statice livrate împreună cu aplicația, nu endpointuri CocktailDB.

Configurația controlează:

```text
brand
  -> nume, logo, alt text

theme
  -> culori light/dark
  -> fonturi
  -> corner radius

layout
  -> grid/list
  -> comfortable/compact

localization
  -> limbi permise
  -> limba implicită

features
  -> selector de limbă
  -> metadata despre băutură
```

## Pasul 4 — Configurația este validată

Manifestul și JSON-ul brandului intră ca `unknown`:

```text
unknown JSON
  -> runtime-config.validator.ts
  -> AppConfig valid
```

Interfețele TypeScript nu validează date descărcate la runtime. Validatorul verifică structura exactă, valorile și relațiile dintre câmpuri.

Dacă încărcarea sau validarea eșuează:

```text
ConfigurationService.state()
  -> { status: 'error', error: ... }
  -> root component afișează mesaj și Retry
```

Payload-ul invalid nu este expus utilizatorului.

## Pasul 5 — Se aplică tema

`ThemeService` copiază valorile validate într-o listă fixă de CSS custom properties:

```text
config.theme.primary
  -> --color-primary

config.theme.surface
  -> --color-surface

config.theme.typography.bodyFont
  -> --font-body
```

JSON-ul furnizează valori validate, dar nu poate inventa nume de proprietăți CSS.

---

# 2. Lista de băuturi CocktailDB

După ce configurația este `ready`, ruta `/drinks` afișează feature-ul Drinks.

În baseline-ul actual, `AlcoholicDrinksDataService` face:

```http
GET https://www.thecocktaildb.com/api/json/v1/1/filter.php?a=Alcoholic
```

Acest request este identic pentru ambele branduri:

```text
Citrus   ─┐
          ├──> filter.php?a=Alcoholic
Midnight ─┘
```

CocktailDB nu cunoaște brandurile aplicației. Nu există:

```text
/citrus/drinks
/midnight/drinks
filter.php?brand=citrus
```

## Data-access flow

CocktailDB returnează aproximativ:

```json
{
  "drinks": [
    {
      "idDrink": "15423",
      "strDrink": "110 in the shade",
      "strDrinkThumb": "https://..."
    }
  ]
}
```

Fluxul aplicației este:

```text
CocktailDB HTTP response: unknown
  -> parseCocktailDbListResponse()
  -> mapCocktailDbList()
  -> DrinkSummary[]
```

Mapper-ul produce un model al aplicației:

```ts
{
  id: '15423',
  name: '110 in the shade',
  imageUrl: 'https://...'
}
```

Componentele Angular nu cunosc `idDrink`, `strDrink` sau `strDrinkThumb`.

---

# 3. Store-ul listei

`DrinksListStore` se află între serviciul HTTP și UI:

```text
AlcoholicDrinksDataService
          │
          ▼
    DrinksListStore
          │
          ▼
    DrinksListPage
```

Store-ul transformă mecanica `httpResource` într-o stare explicită:

```text
idle
loading
success { data }
empty
error { failure }
```

Template-ul poate afișa behavior-ul corect pentru fiecare stare fără să cunoască `HttpErrorResponse` sau detaliile resursei HTTP.

Store-ul listei este root-scoped. O listă încărcată poate supraviețui navigării listă → detaliu → listă.

---

# 4. Detaliul unei băuturi

Când utilizatorul selectează drink ID `15423`, ruta devine:

```text
#/drinks/15423
```

Fluxul este:

```text
route id "15423"
  -> parseDrinkRouteId()
  -> DrinkDetailStore
  -> DrinkDetailDataService
  -> GET lookup.php?i=15423
  -> validate response
  -> map response
  -> DrinkDetails
  -> DrinkDetailView
```

Endpoint:

```http
GET https://www.thecocktaildb.com/api/json/v1/1/lookup.php?i=15423
```

Este același pentru toate brandurile.

## De ce există un endpoint separat pentru detail?

Endpoint-ul de listă furnizează numai:

```text
ID
nume
imagine
```

`lookup.php` furnizează rețeta completă:

```text
ingrediente
măsuri
instrucțiuni și traduceri
pahar
categorie CocktailDB
clasificare alcoolică
```

## Maparea ingredientelor

CocktailDB folosește câmpuri numerotate:

```json
{
  "strIngredient1": "Rum",
  "strMeasure1": "50 ml",
  "strIngredient2": "Lime",
  "strMeasure2": "20 ml"
}
```

Mapper-ul parcurge sloturile `1–15` și produce:

```ts
ingredients: [
  { name: 'Rum', measure: '50 ml' },
  { name: 'Lime', measure: '20 ml' },
];
```

UI-ul primește un model normalizat, nu câmpurile externe numerotate.

Store-ul de detail este component-scoped. Pagina rutată și drawer-ul desktop nu își împart sau anulează accidental ID-ul și request-ul.

---

# 5. Cum influențează brandul datele afișate

Brandul nu schimbă băutura primită de la CocktailDB. Brandul schimbă prezentarea și behavior-ul configurabil.

```text
                   Același DrinkDetails
                           │
             ┌─────────────┴─────────────┐
             │                           │
          Citrus                      Midnight
             │                           │
 metadata = true                 metadata = false
 languages = en/es/de/fr/it      languages = en
 selector = true                 selector = false
 colors = green/citrus           colors = cyan/blue
 radius = 20px                   radius = 12px
```

Pentru aceeași rețetă:

- Citrus poate afișa metadata și selectorul de limbă;
- Midnight ascunde metadata și selectorul;
- instrucțiunile în engleză rămân vizibile;
- ingredientele și rețeta provin din același response CocktailDB.

---

# 6. Saved și starea locală

Saved nu folosește un endpoint separat:

```text
CocktailDB list
  -> DrinkSummary[]
  -> favorite IDs din localStorage
  -> displayedDrinks = only favorite drinks
```

CocktailDB nu primește requesturi `/saved` sau `/favorites`.

Theme mode și favorite IDs sunt persistate local în browser, nu sincronizate cu un backend.

---

# 7. Secvențele reale de requesturi

## Citrus — listă

```http
GET /config/brand-manifest.json
GET /config/brands/citrus.json
GET https://www.thecocktaildb.com/api/json/v1/1/filter.php?a=Alcoholic
```

## Citrus — detaliu

```http
GET https://www.thecocktaildb.com/api/json/v1/1/lookup.php?i=15423
```

## Midnight — listă și detaliu

```http
GET /config/brand-manifest.json
GET /config/brands/midnight.json
GET https://www.thecocktaildb.com/api/json/v1/1/filter.php?a=Alcoholic
GET https://www.thecocktaildb.com/api/json/v1/1/lookup.php?i=15423
```

Singurul request diferit este fișierul JSON al brandului. Browserul mai poate încărca imaginea băuturii din URL-ul returnat de CocktailDB.

---

# 8. Responsabilitatea layerelor

| Layer                     | Întrebarea la care răspunde                         |
| ------------------------- | --------------------------------------------------- |
| `core/config`             | Ce brand rulează și cum este configurată aplicația? |
| `data-access`             | Este răspunsul CocktailDB valid și cum îl mapăm?    |
| `services`                | Ce request HTTP trebuie făcut?                      |
| `store`                   | Care este starea feature-ului acum?                 |
| routed pages              | Cum coordonăm ruta, state-ul și interacțiunile?     |
| presentational components | Cum afișăm datele și emitem acțiuni?                |

```text
CONFIGURATION

manifest/config JSON
  -> runtime validator
  -> ConfigurationService
  -> AppConfig
  -> theme/layout/localization/features


DRINKS

CocktailDB endpoint
  -> response validator
  -> mapper
  -> data service
  -> feature store
  -> page/container
  -> presentational components
```

---

# 9. Integrarea filtrării server-side pe categorii

În baseline-ul actual, lista folosește numai:

```text
a=Alcoholic
```

Filtrarea cerută se integrează în fluxul listei, nu în configurația brandului și nu în detail:

```text
pill selectat
  -> DrinksListStore.selectedCategory signal
  -> list data service request
       All             -> filter.php?a=Alcoholic
       Cocktail        -> filter.php?c=Cocktail
       Shot            -> filter.php?c=Shot
       Beer            -> filter.php?c=Beer
       Ordinary Drink  -> filter.php?c=Ordinary Drink
       Coffee / Tea    -> filter.php?c=Coffee / Tea
  -> existing list response validator
  -> existing list mapper
  -> store loading/success/empty/error
  -> existing page and grid
```

Această extensie nu trebuie să folosească:

- filtrare locală a listei;
- `lookup.php` pentru rezultate de listă;
- ingredient filtering;
- requesturi duplicate;
- logică HTTP în componentă.

Schimbarea categoriei trebuie să reseteze paginarea vizibilă, iar request state-ul rămâne în store.

Brandurile continuă să utilizeze aceleași endpointuri. Dacă în viitor feature-ul ar deveni configurabil per brand, configurația ar controla doar disponibilitatea UI-ului, nu ar crea endpointuri CocktailDB distincte.

---

# 10. Explicație scurtă pentru interviu

> The application has two independent external-input flows. Runtime brand JSON determines how the product looks and which presentation features are enabled. CocktailDB provides the actual drink and recipe data. Both brands call the same CocktailDB list and detail endpoints.
>
> Brand JSON is loaded first, validated from `unknown`, and converted into a typed `AppConfig`. CocktailDB responses are also validated from `unknown`, mapped from external DTO fields into application-owned models, and then adapted by feature stores into explicit loading, success, empty, error, or not-found states.
>
> This separation means I can add or change a brand without changing data access, and I can extend list querying without mixing CocktailDB logic into the brand or UI layers.

## Varianta într-o singură propoziție

> Brand configuration decides how the application behaves and looks; CocktailDB supplies the drinks; runtime validators, mappers, and feature stores keep those two flows separate and safe.
