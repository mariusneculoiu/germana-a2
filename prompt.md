Inspect the Angular app and implement server-side category filtering for
All, Cocktail, Shot, Beer, Ordinary Drink and Coffee / Tea.
Use CocktailDB `filter.php`: All uses `a=Alcoholic`; categories use `c=<category>`.
Do not use local filtering, `lookup.php`, ingredient filtering or duplicate requests.
Preserve the existing architecture and UI. Keep request state in the feature store and API mapping in data access.
Reset pagination when the category changes and preserve loading, empty and error states.
Add accessible pill buttons immediately below “What are you sipping today?”, aligned left. Include active, hover and visible focus states, 2.75rem touch targets and clean wrapping without horizontal overflow at 390px.
Do not redesign the existing page or grid. Use TDD. Add focused tests for requests, selection and pagination reset. Inspect the result in a browser at desktop and 390px, then run tests and build. Report commands and results. Do not commit.
