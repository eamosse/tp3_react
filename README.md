# TP Jour 3 — Chirpy : branchement au backend réel

**Durée : 1 h 15** · **Prérequis : TP jour 2 terminé + backend lancé** · **Solution de référence : `solutions/jour-3/`**

## Objectif

Brancher Chirpy au vrai backend Express. **En deux temps :**

1. **v1 (30 min)** : `usePosts` à la main avec `useEffect` + `fetch`. Vous allez souffrir.
2. **v2 (45 min)** : refactor vers **TanStack Query** — cache, invalidation, mutation. Vous allez apprécier.

À la fin, le feed est réellement persisté côté serveur, le formulaire crée des posts qui restent au reload, et les likes sont synchronisés.

## Ce qu'on ajoute

- `src/api/client.ts` : fonction `apiFetch` centralisée.
- `src/hooks/usePosts.ts` : custom hook de lecture (v1 useEffect, puis v2 Query).
- `src/hooks/useCreatePost.ts` : mutation de création.
- `src/hooks/useToggleLike.ts` : mutation de like.
- Dans `main.tsx` : `<QueryClientProvider>` pour la v2.

## Étape A — démarrer le backend

**Dans un terminal séparé** :

```bash
cd chirpy/backend
npm install
DELAY_MS=400 npm run dev
```

Le `DELAY_MS=400` est **volontaire** : vous allez voir les spinners.

Test rapide :

```bash
curl http://localhost:3001/posts | head -c 300
```

Vous devez voir les 15 posts du seed.

## Étape B — utiliser un faux user (pour cette étape)

L'auth arrive jour 4. Pour **ce TP**, créez un token une fois pour toutes :

```bash
curl -s -X POST http://localhost:3001/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"alice123"}' \
  | python3 -c "import sys, json; print(json.load(sys.stdin)['token'])"
```

Copiez le token et mettez-le dans `chirpy/frontend/.env.local` :

```text
VITE_API_URL=http://localhost:3001
VITE_DEV_TOKEN=eyJhbGciOi...
```

Redémarrez `npm run dev` (Vite relit `.env.local` au boot).

## Étape C — v1 : `usePosts` avec `useEffect`

Créer `src/api/client.ts` :

```tsx
const API = import.meta.env.VITE_API_URL;
const DEV_TOKEN = import.meta.env.VITE_DEV_TOKEN;

export async function apiFetch(path: string, options: RequestInit = {}): Promise<Response> {
  const res = await fetch(`${API}${path}`, {
    ...options,
    headers: {
      "Content-Type": "application/json",
      ...(DEV_TOKEN ? { Authorization: `Bearer ${DEV_TOKEN}` } : {}),
      ...options.headers,
    },
  });
  if (!res.ok) {
    const body = await res.json().catch(() => ({}));
    throw new Error(body.error ?? `HTTP ${res.status}`);
  }
  return res;
}
```

Puis `src/hooks/usePosts.ts` **(v1 : à écrire vous-même)** :

```tsx
export function usePosts(): {
  posts: Post[];
  loading: boolean;
  error: Error | null;
} {
  // TODO : useState pour posts, loading, error
  // TODO : useEffect qui fetch /posts au mount
  // TODO : gérer AbortController (cleanup)
  // TODO : retourner { posts, loading, error }
}
```

Utilisez-le dans `App.tsx` :

```tsx
const { posts, loading, error } = usePosts();
if (loading) return <p>Chargement…</p>;
if (error) return <p>Erreur : {error.message}</p>;
// puis passer `posts` à <Feed>
```

### Commit et push vos changements avant de continuer 

## Étape D — v2 : TanStack Query

Ajouter la dépendance :

```bash
npm install @tanstack/react-query
npm install --save-dev @tanstack/react-query-devtools
```

Modifier `src/main.tsx` :

```tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";

const queryClient = new QueryClient();

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <QueryClientProvider client={queryClient}>
      <App />
      <ReactQueryDevtools />
    </QueryClientProvider>
  </React.StrictMode>,
);
```

Réécrire `src/hooks/usePosts.ts` :

```tsx
import { useQuery } from "@tanstack/react-query";
import { apiFetch } from "../api/client";
import type { Post } from "../types/post";

export function usePosts() {
  return useQuery<Post[], Error>({
    queryKey: ["posts"],
    queryFn: async ({ signal }) => {
      const r = await apiFetch("/posts", { signal });
      return r.json();
    },
  });
}
```

Écrire `src/hooks/useCreatePost.ts` :

```tsx
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { apiFetch } from "../api/client";
import type { Post } from "../types/post";

export function useCreatePost() {
  const qc = useQueryClient();
  return useMutation<Post, Error, string>({
    mutationFn: async (content) => {
      const r = await apiFetch("/posts", {
        method: "POST",
        body: JSON.stringify({ content }),
      });
      return r.json();
    },
    onSuccess: () => qc.invalidateQueries({ queryKey: ["posts"] }),
  });
}
```

Écrire `src/hooks/useToggleLike.ts` (similaire, endpoint `POST /posts/:id/like`).

Adapter `App.tsx` :

```tsx
const { data: posts = [], isPending, error } = usePosts();
const createPost = useCreatePost();
const toggleLike = useToggleLike();

// plus de setPosts local !
// handleCreate = (content) => createPost.mutate(content);
// handleToggleLike = (id) => toggleLike.mutate(id);
```

--- 

## Étape 3 — Enrichir votre application 

Ajouter les fonctions suivantes : 
- Edition d'un post
- Mettre un post en favori
- Supprimer un post 

>> Attention : tout doit être synchronisé avec le backend


## Critères de validation

| | Critère |
|---|---|
| ✅ | **v1** : l'app affiche un état "Chargement…" pendant ~400 ms, puis le feed. |
| ✅ | **v1** : couper le backend → message d'erreur, pas d'écran blanc. |
| ✅ | **v2** : le feed s'affiche pareil, mais le cache persiste entre navigations. |
| ✅ | **v2** : créer un post l'ajoute au feed (invalidation de `['posts']`). |
| ✅ | **v2** : liker un post incrémente le compteur dans le feed. |
| ✅ | **v2** : F5 → les posts sont bien toujours là (persistence backend). |
| ✅ | **v2** : React Query devtools visibles en bas de page (en dev). |
| ✅ | `npm run typecheck` vert, aucun `any`. |
| ✅ | **v2** : On peut supprimer un post. |
| ✅ | **v2** : On peut editer un post. |
| ✅ | **v2** : On peut ajouter un post en favori. |

## Pièges à surveiller

- **CORS** : le backend autorise `http://localhost:5173` uniquement. Changez `.env` si besoin.
- **Reset du seed** : à chaque redémarrage du backend, les posts reviennent au seed initial.
- **StrictMode double-fire** : en dev, les effets tirent 2× au mount. Votre cleanup doit être propre (c'est justement le but).
- **Invalidation oubliée** : si après un `createPost`, le feed ne se met pas à jour, vérifiez `onSuccess: () => qc.invalidateQueries({ queryKey: ["posts"] })`.

## Bonus

1. **Optimistic update** sur le like : incrémenter le compteur **avant** la réponse du serveur, rollback si erreur. Voir `useMutation({ onMutate, onError, onSettled })`.
2. **Pagination infinie** : la route `/posts` peut être modifiée pour supporter `?limit=10&offset=N`, utilisez `useInfiniteQuery`.
3. **Retirer le `DEV_TOKEN` en dur** : préparez le terrain pour jour 4 en créant un simple module `src/api/auth.ts` qui lit le token depuis le localStorage.

## Aide

**v1 qui re-fetch en boucle ?** Dépendances manquantes. Mettez `[]`.

**v2 qui ne met pas à jour après création ?** Vous avez oublié `invalidateQueries` dans `onSuccess`.

**Erreur `QueryClient not found` ?** Vous n'avez pas enveloppé `<App>` dans `<QueryClientProvider>`.
