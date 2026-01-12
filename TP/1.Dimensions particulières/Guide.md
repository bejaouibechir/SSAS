# 📚 Extensions Pédagogiques - Modélisation Avancée

## 🎯 Objectif

Ces scripts **optionnels** ajoutent des concepts avancés de modélisation dimensionnelle au Data Warehouse TechMart, sans modifier la structure de base existante.

**Idéal pour :**

- Formation BI avancée
- Ateliers de modélisation Kimball
- Démonstration de patterns avancés
- Comparaison Star vs Snowflake

---

## 📦 Scripts fournis

### Extension 1️⃣ : Modèle Flocon de Neige (Snowflake Schema)

| Script                        | Description                                   | Durée    |
| ----------------------------- | --------------------------------------------- | -------- |
| `03A_CREATE_DimGeography.sql` | Création de DimGeography + extension DimStore | < 10 sec |
| `03B_LOAD_DimGeography.sql`   | Chargement des géographies + liaison          | < 5 sec  |

### Extension 2️⃣ : Table de Faits sans Mesures (Factless Fact)

| Script                                   | Description                        | Durée   |
| ---------------------------------------- | ---------------------------------- | ------- |
| `03C_CREATE_FactPromotionPerProduct.sql` | Création table Many-to-Many        | < 5 sec |
| `03D_LOAD_FactPromotionPerProduct.sql`   | Chargement relations Promo-Product | < 5 sec |

**Total : < 1 minute** pour installer les deux extensions

---

## 🌟 Extension 1 : DimGeography (Snowflake Schema)

### Concept pédagogique

**Avant (Star Schema) :**

```
FactProductSales → DimStore (City, Country, Region dupliqués)
```

**Après (Snowflake Schema) :**

```
FactProductSales → DimStore → DimGeography (normalisé)
```

### Ce que ça démontre

✅ **Normalisation** : Les données géographiques ne sont plus dupliquées  
✅ **Hiérarchies** : Ville → Pays → Région → Continent  
✅ **Avantages** : Moins de duplication, mises à jour centralisées  
✅ **Inconvénients** : Plus de jointures = légère perte de performance

### Structure DimGeography

```sql
DimGeography
├── GeographySK (PK)
├── GeographyBK
├── City              ← Niveau 1 (détaillé)
├── StateProvince     ← Niveau 2
├── Country           ← Niveau 3
├── CountryCode       ← ISO code
├── Region            ← Niveau 4
└── Continent         ← Niveau 5 (agrégé)
```

### Données chargées

| Ville      | Pays    | Région         | Continent | Population |
| ---------- | ------- | -------------- | --------- | ---------- |
| Tunis      | Tunisia | North Africa   | Africa    | 2.7M       |
| Paris      | France  | Western Europe | Europe    | 12.5M      |
| Berlin     | Germany | Central Europe | Europe    | 3.8M       |
| Casablanca | Morocco | North Africa   | Africa    | 4.3M       |
| Lyon       | France  | Western Europe | Europe    | 2.3M       |

### Installation

```sql
-- Étape 1 : Créer la structure
EXEC :03A_CREATE_DimGeography.sql

-- Étape 2 : Charger les données
EXEC :03B_LOAD_DimGeography.sql
```

### Exemples de requêtes

**Analyse par continent :**

```sql
SELECT 
    g.Continent,
    SUM(f.TotalAmount) AS CA_Total
FROM dw.FactProductSales f
INNER JOIN dw.DimStore s ON f.StoreSK = s.StoreSK
INNER JOIN dw.DimGeography g ON s.GeographySK = g.GeographySK
GROUP BY g.Continent;
```

**Drill-down géographique :**

```sql
SELECT 
    g.Continent,
    g.Region,
    g.Country,
    g.City,
    COUNT(*) AS Nb_Ventes,
    SUM(f.TotalAmount) AS CA
FROM dw.FactProductSales f
INNER JOIN dw.DimStore s ON f.StoreSK = s.StoreSK
INNER JOIN dw.DimGeography g ON s.GeographySK = g.GeographySK
GROUP BY ROLLUP(g.Continent, g.Region, g.Country, g.City);
```

---

## 🎲 Extension 2 : FactPromotionPerProduct (Factless Fact)

### Concept pédagogique

**Table de faits SANS mesures numériques**

Contrairement à `FactProductSales` qui contient :

- ✅ Quantity, UnitPrice, TotalAmount (mesures)

`FactPromotionPerProduct` contient :

- ❌ **Aucune mesure**
- ✅ Juste des **relations** entre dimensions

### Ce que ça démontre

✅ **Relations Many-to-Many** : Un produit → Plusieurs promotions  
✅ **Couverture** : Quels produits sont éligibles à quelles promos  
✅ **Comptage** : Utilisation de COUNT(*) au lieu de SUM()  
✅ **Événements** : Capture d'occurrences sans mesures

### Structure FactPromotionPerProduct

```sql
FactPromotionPerProduct (Factless Fact)
├── PromotionSK (PK, FK → DimPromotion)
├── ProductSK (PK, FK → DimProduct)
├── StartDate
├── EndDate
└── IsActive
```

**Note** : Aucune colonne Quantity, Amount, Price !

### Logique de chargement

**Règle 1** : Promotions "All" → Tous les produits  
**Règle 2** : Promotions ciblées → Produits de la catégorie

Exemple :

- Black Friday (All) → 50 produits = 50 lignes
- Summer Sale Laptops (Laptop) → 15 laptops = 15 lignes

**Total attendu** : ~400-600 relations

### Installation

```sql
-- Étape 1 : Créer la structure
EXEC :03C_CREATE_FactPromotionPerProduct.sql

-- Étape 2 : Charger les relations
EXEC :03D_LOAD_FactPromotionPerProduct.sql
```

### Exemples de requêtes

**Produits éligibles à Black Friday :**

```sql
SELECT 
    prod.ProductName,
    prod.Category,
    prod.ListPrice
FROM dw.FactPromotionPerProduct f
INNER JOIN dw.DimProduct prod ON f.ProductSK = prod.ProductSK
INNER JOIN dw.DimPromotion prom ON f.PromotionSK = prom.PromotionSK
WHERE prom.PromotionName = 'Black Friday 2024';
```

**Comptage de promotions par produit :**

```sql
SELECT 
    prod.ProductName,
    COUNT(*) AS Nb_Promotions
FROM dw.FactPromotionPerProduct f
INNER JOIN dw.DimProduct prod ON f.ProductSK = prod.ProductSK
GROUP BY prod.ProductName
ORDER BY COUNT(*) DESC;
```

**Produits jamais en promotion :**

```sql
SELECT 
    prod.ProductName,
    prod.ListPrice
FROM dw.DimProduct prod
WHERE NOT EXISTS (
    SELECT 1 
    FROM dw.FactPromotionPerProduct f 
    WHERE f.ProductSK = prod.ProductSK
);
```

---

## 📊 Architecture finale du DW

Après installation des extensions :

```
┌─────────────────────────────────────────────────────────┐
│                  TECHMART DATA WAREHOUSE                │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ⭐ STAR SCHEMA (Principal)                            │
│     FactProductSales → DimDate                         │
│                     → DimTime                          │
│                     → DimCustomer                      │
│                     → DimProduct                       │
│                     → DimStore                         │
│                     → DimSalesPerson                   │
│                     → DimPromotion                     │
│                     → DimCampaign                      │
│                                                         │
│  ❄️  SNOWFLAKE SCHEMA (Extension 1)                    │
│     DimStore → DimGeography                            │
│                                                         │
│  🎲 FACTLESS FACT (Extension 2)                        │
│     DimProduct ←→ FactPromotionPerProduct ←→ DimPromotion │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🎓 Utilisation pédagogique

### Atelier 1 : Star vs Snowflake

**Démonstration** :

1. Requête SANS DimGeography (Star) : 1 jointure
2. Requête AVEC DimGeography (Snowflake) : 2 jointures
3. Comparer les plans d'exécution
4. Discuter trade-offs : normalisation vs performance

**Questions** :

- Quand utiliser Star ? Quand utiliser Snowflake ?
- Impact sur les cubes SSAS/Power BI ?
- Comment gérer les hiérarchies ?

### Atelier 2 : Factless Fact Tables

**Démonstration** :

1. Comparer FactProductSales (avec mesures) vs FactPromotionPerProduct (sans mesures)
2. Montrer l'utilisation de COUNT au lieu de SUM
3. Expliquer les cas d'usage (événements, couverture, éligibilité)

**Questions** :

- Quand utiliser une Factless Fact ?
- Différence avec une table de dimension ?
- Comment stocker des événements sans mesures ?

### Atelier 3 : Many-to-Many

**Démonstration** :

1. Problème : Un produit a plusieurs promotions
2. Solution classique (incorrecte) : Dupliquer dans DimProduct
3. Solution correcte : Table de jonction (FactPromotionPerProduct)

---

## 🔧 Désinstallation

Si vous voulez retirer les extensions :

```sql
-- Supprimer FactPromotionPerProduct
DROP TABLE IF EXISTS dw.FactPromotionPerProduct;

-- Supprimer le lien Snowflake
ALTER TABLE dw.DimStore DROP CONSTRAINT IF EXISTS FK_DimStore_DimGeography;
DROP INDEX IF EXISTS IX_DimStore_GeographySK ON dw.DimStore;
ALTER TABLE dw.DimStore DROP COLUMN IF EXISTS GeographySK;

-- Supprimer DimGeography
DROP TABLE IF EXISTS dw.DimGeography;
```

**Note** : Cela ne supprime PAS le DW principal (FactProductSales, DimProduct, etc.)

---

## 📝 Notes importantes

### ⚠️ Ces extensions sont OPTIONNELLES

- Elles ne modifient PAS la structure de base
- Elles ne cassent PAS les requêtes existantes
- Elles peuvent être installées/désinstallées indépendamment

### ✅ Compatibilité

- Power BI : Gère automatiquement les hiérarchies DimGeography
- SSAS : Snowflake supporté mais moins optimal que Star
- SSRS : Aucun impact, juste plus de jointures possibles

### 🎯 Ordre d'exécution recommandé

**Base (obligatoire) :**

1. `01_CREATE_DW_Structure.sql`
2. `02A_LOAD_Dimensions.sql`
3. `02B_LOAD_Facts_2022_2023.sql`
4. `02C_LOAD_Facts_2024.sql`

**Extensions (optionnelles) :** 5. `03A_CREATE_DimGeography.sql` 6. `03B_LOAD_DimGeography.sql` 7. `03C_CREATE_FactPromotionPerProduct.sql` 8. `03D_LOAD_FactPromotionPerProduct.sql`

---

## 📚 Ressources

**Livres recommandés** :

- "The Data Warehouse Toolkit" - Ralph Kimball
- "Building a Scalable Data Warehouse with Data Vault 2.0" - Dan Linstedt

**Concepts couverts** :

- ✅ Star Schema (FactProductSales)
- ✅ Snowflake Schema (DimStore → DimGeography)
- ✅ Factless Fact Table (FactPromotionPerProduct)
- ✅ Slowly Changing Dimensions (SCD Type 2 ready)
- ✅ Many-to-Many relations
- ✅ Hierarchies (Geographic drill-down)

---

## 🎉 Conclusion

Ces extensions transforment votre DW TechMart en un **exemple complet** couvrant tous les patterns de modélisation dimensionnelle de Kimball !

**Parfait pour :**

- Formations BI complètes
- Certifications Microsoft BI
- Ateliers pratiques de modélisation
- Démonstrations client/étudiant

**Bon atelier ! 🚀**


