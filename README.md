# Caixa Comuna

Pressupost compartit de parella. App web independent (sense dependre de Claude): Firebase per desar i sincronitzar les dades entre els dos mòbils, GitHub Pages per allotjar la pàgina.

## 1. Crear el projecte Firebase (~5 min)

1. Vés a [console.firebase.google.com](https://console.firebase.google.com) i inicia sessió amb un compte Google.
2. "Afegeix un projecte" → nom-li com vulguis (p. ex. `caixa-comuna`) → segueix l'assistent (pots desactivar Google Analytics, no cal).
3. Un cop creat, al menú lateral: **Build → Authentication → Get started**.
   - A la pestanya "Sign-in method", activa el proveïdor **Email/Password**.
   - Vés a la pestanya "Users" → **Add user** → crea un usuari per a cadascú (el vostre correu real + una contrasenya). Feu-ho per als dos.
4. Al menú lateral: **Build → Firestore Database → Create database**.
   - Tria una ubicació (qualsevol d'Europa, p. ex. `eur3`) → mode **Production**.
5. Un cop creada la base de dades, ves a la pestanya **Rules** i enganxa el contingut del fitxer [`firestore.rules`](firestore.rules) d'aquest projecte, **substituint els dos correus placeholder pels vostres correus reals** (els mateixos que has fet servir al pas 3). Publica els canvis.
6. Torna a **Project settings** (icona d'engranatge, dalt de l'esquerra) → pestanya **General** → secció "Your apps" → **`</>`  (Web)** → registra una app (nom qualsevol, no cal Hosting de Firebase).
7. Copia l'objecte `firebaseConfig` que et mostra (apiKey, authDomain, projectId, etc.).

## 2. Enganxar la configuració a l'app

Obre [`index.html`](index.html) i busca aquest bloc (a prop de la línia 210):

```js
const firebaseConfig = {
  apiKey: "PASTE_API_KEY",
  authDomain: "PASTE_PROJECT_ID.firebaseapp.com",
  ...
};
```

Substitueix els valors pels que has copiat del pas 1.7. Desa el fitxer.

## 3. Publicar-ho a GitHub Pages

1. Crea un repositori nou a GitHub (per exemple `caixa-comuna`), **privat** és el més recomanable ja que hi haurà dades personals.
2. Des d'aquesta carpeta:
   ```bash
   git add -A
   git commit -m "Caixa Comuna: app independent amb Firebase"
   git remote add origin <URL_DEL_TEU_REPO>
   git push -u origin main
   ```
3. Al repositori de GitHub: **Settings → Pages** → "Build and deployment" → Source: **Deploy from a branch** → branch `main`, carpeta `/ (root)` → Save.
4. Al cap d'un parell de minuts, GitHub et donarà una URL del tipus `https://<el-teu-usuari>.github.io/caixa-comuna/`. Aquesta és la vostra app, per sempre, sense passar per Claude.

## 4. Fer-la servir al mòbil

1. Obre la URL de GitHub Pages al navegador del mòbil.
2. Inicia sessió amb el correu i contrasenya que vau crear al pas 1.3 (cadascú amb el seu).
3. **Afegeix-la a la pantalla d'inici**: a Safari (iPhone) → botó de compartir → "Afegeix a la pantalla d'inici". A Chrome (Android) → menú (⋮) → "Afegeix a la pantalla d'inici".

A partir d'aquí, els dos mòbils veuran sempre les mateixes dades en temps real, i com que Firestore desa una còpia local al dispositiu, l'app també funciona sense connexió — els canvis es sincronitzen automàticament quan torneu a tenir internet.

## Notes

- El repositori de GitHub hauria de ser **privat**: encara que `firestore.rules` restringeix qui pot llegir/escriure les dades, un repo públic deixaria veure la configuració de Firebase (l'`apiKey` no és cap secret real, però és bona pràctica no exposar-ho innecessàriament).
- Si mai voleu afegir una tercera persona amb accés, creeu-li un usuari a Firebase Authentication i afegiu el seu correu a `firestore.rules`.
- Aquesta app no té cap servidor propi: tot el "backend" és Firebase (gratuït per aquest volum d'ús).
