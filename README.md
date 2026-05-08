# Germana A2 - Tools

Aplicatie web pentru invatarea vocabularului si frazelor de germana nivel A2 (Goethe-Zertifikat). Functioneaza 100% in browser - fara backend, fara cookie-uri, datele sunt salvate doar in `localStorage`.

**Demo live:** https://mariusneculoiu.github.io/germana-a2/

## Ce contine

| Pagina | Descriere |
|--------|-----------|
| `index.html` | Aplicatia principala de vocabular: 1254 cuvinte A2 preincarcate, 4 moduri (flashcards, quiz alegere, quiz scriere, matching), import CSV/TXT, filtrare pe sectiuni, tracking progres |
| `fraze.html` | Quiz pentru fraze A2 (14 fraze pe tema Person/Familie/Eigenschaften) cu explicatii gramaticale (weil, wenn, damit, obwohl, der/die/das, wo) |
| `gramatica.html` | **Modul de gramatica nou:** 15 topicuri esentiale (5 A1 + 6 A2 + 4 B1) cu teorie in romana si 90 exercitii (fill-in, alegere, transformari, ordonare cuvinte). Filtrare pe nivel si categorie. |
| `data/vocabular_a2.csv` | Vocabularul complet ca CSV editabil in Excel |
| `descarcari/flashcards.xlsx` | Flashcards in Excel cu auto-color si formule de progres |
| `descarcari/ghid-studiu.docx` | Ghid Word de 7 pagini cu fraze, gramatica detaliata si mnemonice |

## Topicuri de gramatica acoperite

**A1 (5):** Pronume personale, sein/haben, Verbe modale, Akkusativ, Dativ

**A2 (6):** Perfekt (haben/sein), Verbe reflexive, Verbe separabile, Wechselpräpositionen, Subordonate (weil/dass/wenn/obwohl), Declinarea adjectivelor

**B1 (4):** Präteritum, Konjunktiv 2, Pasiv (werden), Propozitii relative

## Cum se foloseste

### Vocabular (index.html)

- **Flashcards:** click pe carte sau SPACE pentru flip. Sageti pentru navigare. Tastele 1/2/3 marcheaza ca greu/mediu/usor.
- **Quiz alegere:** 4 raspunsuri, click pe cel corect. Cuvintele gresite se marcheaza automat ca "de repetat".
- **Quiz scriere:** scrii raspunsul. Articolele (der/die/das) sunt optionale. Caracterele speciale acceptate: `a`=ä, `o`=ö, `u`=ü, `ss`=ß.
- **Matching:** gaseste 5 perechi RO ↔ DE.
- **Sectiuni:** filtreaza pe categorii (Profesii, Familie, Culori, A-Z etc.).

### Import propriu

Tab "Incarca" -- accepta:

- CSV cu separator `;` -- `cuvant_ro;cuvant_de;categorie`
- CSV cu virgula -- `cuvant_ro, cuvant_de`
- TXT cu sageata -- `cuvant_ro → cuvant_de`
- TXT cu tab sau liniuta
- Headere de sectiune: `## Familie`, `[Familie]`, `SECȚIUNEA 1: FAMILIE`

## Deployment pe GitHub Pages

### Optiunea A - Drag and drop (fara terminal)

1. Mergi pe https://github.com/new si creeaza un repo public numit **germana-a2**
2. Pe pagina noua a repo-ului, click pe **"uploading an existing file"**
3. Trage in browser tot continutul folder-ului `germana-a2/` (sau zip-ul)
4. Commit
5. Mergi la **Settings -> Pages**
6. La "Source" alege **Deploy from a branch -> main -> / (root)** -> Save
7. Asteapta 1-2 minute. Site-ul va fi la: `https://mariusneculoiu.github.io/germana-a2/`

### Optiunea B - Cu git din terminal

```bash
# Intra in folder
cd germana-a2

# Initializeaza repo-ul
git init -b main
git add .
git commit -m "Initial commit: Germana A2 tools"

# Conecteaza la GitHub (creeaza intai repo-ul gol pe github.com/new)
git remote add origin https://github.com/mariusneculoiu/germana-a2.git
git push -u origin main
```

Apoi mergi la **Settings -> Pages -> Source: main / (root)** si salveaza.

### Optiunea C - GitHub Desktop

1. Deschide GitHub Desktop -> File -> Add Local Repository -> alege folderul `germana-a2/`
2. Cand iti spune "Repository does not exist", click **Create a Repository**
3. Click **Publish repository** (alege Public)
4. Pe github.com mergi la repo -> Settings -> Pages -> Source: main / (root)

## Update-uri ulterioare

Daca vrei sa adaugi cuvinte noi:

1. Editeaza `data/vocabular_a2.csv` in Excel/LibreOffice (separator `;`)
2. Salveaza si fa push:
   ```bash
   git add data/vocabular_a2.csv
   git commit -m "Adaug cuvinte noi"
   git push
   ```

Notă: vocabularul preincarcat este *embedded* in `index.html` ca JSON. Daca vrei ca CSV-ul sa fie sursa de adevar, deschide `index.html` si inlocuieste array-ul `DEFAULT_VOCAB` cu un fetch la `data/vocabular_a2.csv` (ti pot da scriptul daca vrei).

## Tehnic

- HTML5/CSS3/JS vanilla, fara dependinte
- Persistenta in `localStorage`
- Mobile-responsive
- Fara analytics, fara tracking, fara backend

## Licenta

MIT - vezi `LICENSE`. Folositi liber pentru invatare personala sau scolara.

## Multumiri

Vocabularul provine din lista oficiala Goethe-Zertifikat A2.
