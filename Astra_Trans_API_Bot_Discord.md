# API Astra Trans — Intégration du bot Discord

Ce document contient uniquement les informations nécessaires pour connecter le bot Discord à l'API Astra Trans.

## Adresse de l'API

```text
https://iabptnezdzzmogffondv.supabase.co/rest/v1/rpc/
```

## Authentification

Le bot utilise deux clés :

- `ASTRA_PUBLIC_KEY` : `sb_publishable_aopTWv2fNt42Ne9yIL_sgQ_cOZXRXxj`
- `ASTRA_API_KEY` : clé privée du bot commençant par `astra_`, **à transmettre séparément**.

Stocker la clé privée dans les variables d'environnement du serveur du bot. Ne jamais la publier dans un dépôt, un salon Discord ou du code accessible au public.

Les requêtes utilisent **POST**, avec un corps JSON et les en-têtes suivants :

```http
Content-Type: application/json
apikey: <ASTRA_PUBLIC_KEY>
x-astra-api-key: <ASTRA_API_KEY>
```

Ne pas utiliser la clé `astra_` comme Bearer token dans `Authorization`.

## 1. Récupérer les livraisons

**Route :** `POST /astra_v1_deliveries`

Corps de la requête :

```json
{
  "p_after": "0",
  "p_limit": 50
}
```

Principaux champs retournés pour chaque livraison :

| Champ | Signification |
| --- | --- |
| `delivery_id` | Identifiant unique de la livraison |
| `driver_name` | Pseudo du chauffeur |
| `status` | `delivered` (terminée) ou `cancelled` (annulée) |
| `occurred_at` | Date de la livraison |
| `source_city` / `destination_city` | Villes de départ et d'arrivée |
| `source_company` / `destination_company` | Entreprises de départ et d'arrivée |
| `distance_km` | Distance en kilomètres (peut être `null`) |
| `duration_seconds` | Durée en secondes (peut être `null`) |

**Pour les annonces Discord :**

1. Consulter l'API toutes les **30 à 60 secondes**.
2. Conserver `next_cursor` en stockage persistant **comme chaîne de caractères** et l'utiliser comme `p_after` lors de l'appel suivant.
3. Si `has_more` vaut `true`, récupérer la page suivante.
4. Utiliser `delivery_id` pour éviter les annonces en double.
5. Enregistrer le curseur seulement après traitement réussi des livraisons ; prévoir une reprise après redémarrage.

Désactiver les mentions automatiques lors de la publication sur Discord (par exemple `allowedMentions: { parse: [] }`).

## 2. Récupérer les statistiques

**Route :** `POST /astra_v1_stats`

Corps de la requête :

```json
{}
```

Champs retournés :

| Champ | Signification |
| --- | --- |
| `delivered` | Nombre de livraisons terminées |
| `cancelled` | Nombre de livraisons annulées |
| `distance_km` | Somme des kilomètres connus sur les livraisons terminées |
| `duration_seconds` | Somme des durées connues sur les livraisons terminées |
| `distance_known_count` | Nombre de livraisons terminées avec distance renseignée |

Les statistiques concernent les **livraisons synchronisées avec l'API**, pas nécessairement l'historique local complet du Tracker.

## Exemple simple en Node.js

```js
const base = 'https://iabptnezdzzmogffondv.supabase.co/rest/v1/rpc/';

async function callAstra(route, body = {}) {
  const response = await fetch(base + route, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      apikey: process.env.ASTRA_PUBLIC_KEY,
      'x-astra-api-key': process.env.ASTRA_API_KEY
    },
    body: JSON.stringify(body)
  });

  if (!response.ok) throw new Error(`API Astra Trans : HTTP ${response.status}`);
  return response.json();
}

const stats = await callAstra('astra_v1_stats');
const deliveries = await callAstra('astra_v1_deliveries', {
  p_after: '0',
  p_limit: 50
});

console.log('Livraisons terminées :', stats.delivered);
console.log('Livraisons reçues :', deliveries.items.length);
```

Cet exemple illustre un appel seulement. Pour les annonces automatiques, implémenter la boucle de consultation, le stockage persistant du curseur et la déduplication décrits plus haut.

## À retenir

- La clé du bot possède uniquement les permissions `deliveries:read` et `stats:read`.
- Le bot peut consulter les livraisons et statistiques synchronisées de toute la VTC, mais **ne peut pas envoyer ou modifier des livraisons**, ni consulter les profils ou les e-mails des chauffeurs.
- **Aucun webhook Discord n'est fourni en v1** : le bot doit interroger régulièrement les routes de lecture.
- La migration vers les signatures cryptographiques concerne l'envoi des livraisons depuis le Tracker : **les deux routes de lecture ci-dessus restent inchangées**.
- En cas d'erreur `401` ou `403`, vérifier les clés, leurs permissions et leur révocation.
