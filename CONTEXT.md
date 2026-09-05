# Context per continuar el desenvolupament

Aquest fitxer és perquè qualsevol persona (o qualsevol sessió de Claude, amb qualsevol compte, a qualsevol ordinador) pugui agafar aquest projecte des de zero sense haver de repreguntar res del que ja es va decidir. Viatja amb el codi (és a Git), així que sempre hi és disponible.

## Què és això

**Caixa Comuna**: app de pressupost compartit per a una parella (Daniel i Anna). Cada categoria de despesa té un pressupost mensual, el que no es gasta s'acumula sol pel mes següent, i hi ha "pots d'objectius" (casament, casa/cotxe, etc.) alimentats des d'una categoria especial. Té una vista de Resum amb comparació mes a mes i exportació a CSV/PDF.

## Estat actual (2026-09-05)

Totalment funcional i en ús real pels dos. Un sol fitxer (`index.html`) sense build step, desplegat a GitHub Pages, amb Firebase com a únic "backend".

**Canvis d'aquesta sessió:**
- `fmt()` ja no arrodonia els imports a l'euro sencer — ara mostra decimals quan n'hi ha.
- Vista de Resum: nova secció "Tots els moviments", una taula amb totes les despeses de tots els mesos (no només el mes actual), amb edició in-line (import, data, persona, nota) i el mateix patró d'esborrar-amb-confirmació ("Segur?") que ja hi havia.
- Qualsevol categoria es pot fixar a la secció d'Objectius amb el botó 🎯/📋 de la capçalera de la targeta (`cat.pinnedToGoals`) — segueix funcionant exactament igual (pressupost mensual acumulable + despeses directes), només canvia d'on es renderitza. Per defecte Viatges/Mascota/Objectius hi comencen fixades en una instal·lació nova (`DEFAULT_STATE`), però en un document ja existent (com el de producció) cal fixar-les manualment la primera vegada — no hi ha detecció automàtica per nom/id perquè no és fiable si s'han renombrat o afegit categories.
- **Els objectius (Casament, Casa/Cotxe, Altres) ara admeten despeses, no només aportacions.** Cada objectiu té un `expenses[]` propi (import, nota, data) per registrar quan et gastes els diners ja estalviats (p. ex. pagar el casament un cop hi ha prou estalviat). `goal.saved` ha desaparegut com a camp mutable — ara `goalSaved(goal)` el deriva sempre com `goalContributions(goal.id) - goalWithdrawalsTotal(goal)` (contribucions = despeses amb `goalId` dins la categoria "pot" `goal-source`; retirades = `goal.expenses`), seguint el mateix principi que la resta de l'app (mai un camp que es pugui desincronitzar). Les retirades també apareixen a "Tots els moviments" (marcades "(retirada)"), editables/esborrables igual que qualsevol despesa, però **no compten** a "Gastat per categoria i mes" ni al "Gastat" del resum global — ja es van comptar com a gastat quan van entrar al pot de l'objectiu, no cal comptar-los dues vegades.
- `assignToGoal`/`withdrawFromGoal` abans fallaven en silenci quan l'import superava el disponible — ara retornen el motiu (string) i es mostra sota el formulari (`goalFormErrors`, estat només de la sessió del navegador).
- **Convertir una categoria en objectiu** (`convertCategoryToGoal`, botó 🏆 a la capçalera de la targeta, amb confirmació "Segur?"): pensat per a Viatges/Mascota, que en Daniel va decidir que havien de deixar de ser "pots" amb pressupost mensual propi i passar a rebre assignacions des del pot d'Objectius igual que Casament. **No esborra ni migra res automàticament**: `cat.archived = true` i `cat.budget` es congela a 0 (amb una entrada nova a `budgetHistory` perquè els mesos futurs no acumulin més, però els mesos passats mantenen el seu valor real), i es crea un objectiu nou amb el mateix nom/icona. Tot l'historial de despeses de la categoria arxivada es queda intacte (surt a "Tots els moviments", a l'exportació CSV i a "Gastat per categoria i mes" marcada "(arxivada)"), i el saldo que ja tenia acumulada la categoria es queda comptant a "Disponible conjunt ara" fins que la persona mateixa decideixi traspassar-lo manualment cap al nou objectiu amb un "+ Afegir" (decisió explícita de Daniel: no inventar cap xifra de saldo traspassat).

## Per què aquesta arquitectura (i no una altra)

- **Va començar com un Artifact de Claude** (autopublicant-se amb la capability `artifact`). Es va migrar a Firebase + GitHub Pages perquè l'usuari va demanar explícitament no dependre de Claude per a una eina d'ús indefinit entre dues persones que no tenen per què tenir compte de Claude.
- **Firebase (Auth + Firestore) i no un backend propi**: zero manteniment de servidor, nivell gratuït de sobres per a aquest volum, i Firestore dona sincronització en temps real + persistència offline (`enableIndexedDbPersistence`) de franc.
- **Un sol document Firestore (`budgets/shared`) amb tot l'estat com a JSON**, no una col·lecció normalitzada: la mida de dades és petita (una parella, uns quants anys de despeses) i simplifica moltíssim la sincronia (un `onSnapshot` + un `setDoc`, prou).
- **Sense botó de "mes següent" manual.** Versió inicial en tenia un (com un Tricount clàssic), però l'usuari el va rebutjar explícitament: "no puc tirar enrere, no té sentit". Ara **tot es calcula sobre la marxa** a partir de:
  - `category.budgetHistory` / `person.salaryHistory`: llistes `{date, value}` que es van afegint cada vegada que canvies un pressupost o un sou (mai se sobreescriu sense deixar rastre).
  - Les dates reals de cada despesa (`expense.date`, ISO, posada amb `new Date()` en el moment d'afegir-la).
  - `new Date()` real del dispositiu per saber "quin mes és ara".

  Funcions clau: `valueAtMonth(history, monthKey)` (quin era el valor vigent aquell mes), `categoryAccumulatedBalance(cat, uptoMonthKey)` (acumulat real fins aquell mes), `monthlyGlobalSummary()` / `monthlyByCategory()` / `monthlyByPerson()` (usades tant al Dashboard com al Resum). No hi ha cap camp mutable tipus "balanç actual" desat — tot es deriva, sempre, dels mateixos fets (historial + despeses datades). Això és deliberat: elimina qualsevol possibilitat de quedar desincronitzat i fa que "anar cap enrere" no calgui, perquè no hi ha res a desfer.

- **`normalizeState(s)`**: qualsevol document antic (d'abans d'aquesta funcionalitat) es completa sol la primera vegada que es carrega — afegeix `budgetHistory`/`salaryHistory` si falten (amb el valor actual com a punt de partida) i esborra els camps vells (`balance`, `since`) que ja no es fan servir.

## Bugs reals que es van trobar i corregir (útil saber-ho per no repetir-los)

1. **Calia refrescar la pàgina després d'iniciar sessió.** Causa: `onSnapshot(DOC_REF, ...)` es subscrivia al carregar la pàgina, abans que l'usuari estigués autenticat. Com que les regles de Firestore exigeixen `request.auth != null`, aquesta subscripció rebia un `permission-denied` permanent que Firestore **no reintenta sol** encara que després inicïis sessió correctament. Fix: la subscripció (`subscribeToBudget()`) només s'inicia dins de `onAuthStateChanged` quan ja hi ha un usuari, i es neteja (`unsubscribeSnapshot()`) en tancar sessió.
2. **El botó "📊 Resum" semblava no existir** en un dels mòbils tot i estar desplegat correctament al servidor (verificat amb `curl` directament sobre la URL de GitHub Pages). Causa probable: caché agressiva del mode "Afegit a la pantalla d'inici" (PWA standalone), que no té gest de "refrescar". Mitigació: capçaleres `Cache-Control: no-cache` al `<head>`. Si torna a passar: cal esborrar i tornar a afegir la icona de pantalla d'inici (no n'hi ha prou amb tancar/obrir l'app).
3. **`form.amount.value` no és fiable per llegir camps d'un formulari** en tots els motors (va fallar a jsdom, i és una pràctica fràgil en general) — sempre `form.elements.amount.value`.

## Comptes i identificadors (no són secrets, però són els reals — no inventar-ne d'altres)

- Repo GitHub: `https://github.com/DanielBayerri/Nidito` (compte propietari: `DanielBayerri`, **no** el compte de feina `dbayerrimstech` — un push va fallar amb 403 fins que es va fer servir el compte correcte).
- Web en viu: `https://danielbayerri.github.io/Nidito/`
- Projecte Firebase: `nidito-83171`. Usuaris d'Authentication (Email/Password): `bayerri4tgn@gmail.com` i `annap996@gmail.com` — cap altre compte hi té accés (ni per disseny de l'app, ni per les regles de Firestore a `firestore.rules`).
- L'`apiKey` de Firebase dins de `index.html` **no és cap secret** (és normal que sigui pública en apps web de Firebase); la seguretat real la posen les Firestore Rules, no l'apiKey.

## Com continuar desenvolupant

- És un sol fitxer HTML+JS (`index.html`), sense build ni dependències — obre'l en qualsevol editor i prova'l servint-lo amb qualsevol servidor estàtic (`npx serve`, `python -m http.server`, etc.) o directament des de GitHub Pages després de fer push.
- **Provar canvis abans de publicar**: la manera que s'ha fet servir fins ara és un arnès de proves amb `jsdom` (Node) que carrega `index.html`, substitueix els `import` de Firebase per mocks en memòria (auth fals, `onSnapshot`/`setDoc` fets amb funcions pròpies), i simula clics/formularis reals per comprovar que els números quadren. No forma part del repo (és cosa d'una sessió de treball puntual) — cal recrear-lo (`npm install jsdom` en una carpeta temporal) cada vegada que es vulgui validar un canvi abans de desplegar-lo.
- **Desplegar**: `git add -A && git commit -m "..." && git push`. **Importantment**: el `push` sol necessitar autenticar-se amb un token (Personal Access Token de GitHub) fet servir un sol cop, mai desat al gestor de credencials de Windows (decisió explícita de l'usuari — no tocar la configuració global del PC per a això):
  ```bash
  git -c credential.helper= push https://TOKEN@github.com/DanielBayerri/Nidito.git main:main
  ```
  Un cop fet, revocar el token si s'ha arribat a escriure en algun lloc que no sigui la pròpia terminal.
- GitHub Pages triga 1-2 minuts a servir la versió nova després d'un push.

## Limitacions conegudes i acceptades

- L'historial de pressupostos/sous només és fiable **des que aquesta funcionalitat es va desplegar** (finals d'agost 2026). Per a mesos anteriors a la primera entrada de l'historial, s'assumeix que el valor ja era l'actual — no hi ha manera de reconstruir-ho amb certesa si no es va registrar en el seu moment.
- No hi ha manera d'afegir un tercer usuari des de l'app — cal fer-ho manualment a la consola de Firebase (crear l'usuari a Authentication + afegir el seu correu a `firestore.rules`).
- Les retirades d'un objectiu (`goal.expenses`) sí que surten a l'exportació completa (`exportLedger`, el CSV "tots els moviments") perquè surt del mateix `buildLedger()`, però **no** apareixen desglossades a `exportMonthly` (el pivot per categoria/mes) perquè no són despesa d'una categoria — és una decisió deliberada, no un oblit.
