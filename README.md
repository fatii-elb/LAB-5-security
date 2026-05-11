# Rapport de Reverse Engineering - UnCrackable Level 2

## 📋 Table des matières

1. [Introduction](#introduction)
2. [Objectifs](#objectifs)
3. [Outils Utilisés](#outils-utilisés)
4. [Méthodologie](#méthodologie)
5. [Analyse Détaillée](#analyse-détaillée)
6. [Résultats](#résultats)
7. [Conclusion](#conclusion)

---

## Introduction

Ce rapport documente le processus complet de reverse engineering de l'application **UnCrackable Level 2**, un challenge de sécurité mobile créé par l'OWASP. L'objectif était d'identifier et d'extraire un secret caché dans le code natif de l'application.

### Contexte

UnCrackable Level 2 est une application Android qui cache sa logique de vérification dans une bibliothèque native (fichier `.so`). Contrairement à UnCrackable Level 1 où le secret est en Java, celui-ci se trouve dans du code compilé en C/C++, rendant l'analyse plus complexe.

---

## Objectifs

-  Analyser le flux de vérification de l'application
-  Identifier le chargement de la bibliothèque native
- Localiser et analyser le fichier `libfoo.so`
-  Trouver la fonction JNI responsable de la vérification
-  Extraire le secret caché dans le code natif
-  Valider le secret trouvé

---

## Outils Utilisés

| Outil | Version | Utilisation |
|-------|---------|-------------|
| **JADX** | GUI | Décompilation du code Java et analyse de l'APK |
| **Ghidra** | 12.0.4 | Analyse et décompilation du code natif (libfoo.so) |
| **adb** | Android Debug Bridge | Installation de l'APK sur l'émulateur |
| **Android Emulator** | Pixel 6 | Plateforme d'exécution de l'application |
| **Python** | 3.x | Décodage des données hexadécimales |
| **PowerShell** | Windows | Gestion des fichiers et exécution des commandes |

---

## Méthodologie

### Phase 1 : Reconnaissance

1. **Installation de l'APK**
   ```bash
   adb install UnCrackable-Level2.apk
   ```

2. **Observation du comportement**
   - Lancement de l'application sur l'émulateur
   - Test avec plusieurs valeurs (test, 1234, hello, android)
   - Confirmation que l'application rejette toutes les entrées invalides

### Phase 2 : Analyse Java

3. **Décompilation avec JADX**
   ```bash
   jadx-gui UnCrackable-Level2.apk
   ```

4. **Exploration de l'architecture**
   - Localisation de `MainActivity` : point d'entrée de la vérification
   - Identification de `CodeCheck` : classe intermédiaire
   - Découverte de `System.loadLibrary("foo")` : charge la bibliothèque native

### Phase 3 : Extraction de la Bibliothèque Native

5. **Extraction de l'APK**
   ```powershell
   Expand-Archive -Path UnCrackable-Level2.zip -DestinationPath uncrackable_l2
   ```

6. **Localisation de libfoo.so**
   ```
   uncrackable_l2/lib/x86/libfoo.so
   uncrackable_l2/lib/x86_64/libfoo.so
   uncrackable_l2/lib/arm64-v8a/libfoo.so
   uncrackable_l2/lib/armeabi-v7a/libfoo.so
   ```

### Phase 4 : Analyse Natif avec Ghidra

7. **Import dans Ghidra**
   - Création d'un nouveau projet
   - Import de `libfoo.so` (version x86)
   - Lancement de l'analyse automatique

8. **Recherche de la fonction JNI**
   - Ouverture du Symbol Table
   - Recherche de fonctions commençant par `Java_`
   - Localisation de `Java_sg_vantagepoint_uncrackable2_CodeCheck_bar`

### Phase 5 : Analyse du Code Décompilé

9. **Lecture du pseudo-code**
   - Inspection de la fonction native
   - Identification de l'appel à `strncmp`
   - Extraction du secret

---

## Analyse Détaillée

### Flux de Données

```
┌─────────────────┐
│   Utilisateur   │
└────────┬────────┘
         │ (saisit du texte)
         ▼
┌─────────────────┐
│   MainActivity  │ (récupère l'entrée)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   CodeCheck.a() │ (appelle la méthode)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ CodeCheck.bar() │ (méthode native JNI)
│  (en Java)      │
└────────┬────────┘
         │
         ▼
┌─────────────────────┐
│  libfoo.so          │
│  (code natif C/C++) │
└────────┬────────────┘
         │
         ▼
┌──────────────────────┐
│ strncmp(input,secret)│ (comparaison)
└────────┬─────────────┘
         │
    ┌────┴────┐
    ▼         ▼
 Succès    Échec
```

### Code Décompilé (Ghidra)

La fonction JNI `Java_sg_vantagepoint_uncrackable2_CodeCheck_bar` contient :

```c
// Ligne 15 - Initialisation du secret
builtin_strcpy(local_30,"Thanks for all the fish",0x18);

// Ligne 19 - Comparaison avec strncmp
iVar1 = strncmp(__s1,local_30,0x17);

// Ligne 20 - Vérification du résultat
if (iVar1 == 0) {
    uVar2 = 1;  // Succès
    goto LAB_00011009;
}
```

### Signification

- **`builtin_strcpy`** : Copie le secret dans un buffer local
- **`strncmp`** : Compare l'entrée utilisateur avec le secret sur 0x17 (23) caractères
- **Résultat** : Si `strncmp` retourne 0 (chaînes identiques), la vérification réussit

---

## Résultats

### Secret Trouvé

```
"Thanks for all the fish"
```

### Preuve

Le secret est visible directement dans le code décompilé par Ghidra (ligne 15) :

```c
builtin_strcpy(local_30,"Thanks for all the fish",0x18);
```

### Validation

La vérification utilise `strncmp` pour comparer :
- **Entrée utilisateur** : fournie par l'utilisateur via l'interface
- **Secret** : `"Thanks for all the fish"` stocké en dur dans le code natif
- **Longueur** : 0x17 (23 caractères)

Si les deux chaînes correspondent exactement, la fonction retourne `true` et l'application affiche un message de succès.

---

## Concepts Clés Appris

### 1. JNI (Java Native Interface)
- Permet à Java d'appeler du code natif (C/C++)
- Les fonctions JNI suivent une convention de nommage : `Java_<package>_<classe>_<méthode>`
- Exemple : `Java_sg_vantagepoint_uncrackable2_CodeCheck_bar`

### 2. Bibliothèques Natives (.so)
- Fichiers ELF compilés pour des architectures spécifiques
- APK contient plusieurs versions : x86, x86_64, ARM, ARM64
- Ghidra peut analyser et décompiler ces fichiers

### 3. Obfuscation vs Sécurité
- Cacher du code en natif **complique** l'analyse mais ne la rend pas impossible
- Utiliser une simple chaîne en dur est une faible sécurité
- Pour une véritable sécurité : cryptographie, hachage, vérification côté serveur

### 4. Méthodologie de Reverse Engineering
- **Observation** : comprendre le comportement visible
- **Décompilation** : convertir le code compilé en code lisible
- **Trace de données** : suivre le flux de l'entrée utilisateur
- **Analyse** : comprendre la logique et extraire les secrets

---

## Outils et Techniques Essentiels

### JADX
- Décompile les fichiers `.dex` (bytecode Java) en code source Java
- Interface graphique intuitive pour naviguer dans la structure de l'app
- Utile pour comprendre le flux Java et l'appel de code natif

### Ghidra
- Analyse et décompile les fichiers binaires ELF (bibliothèques natives)
- Convertit le code machine en pseudo-code C/C++ lisible
- Analyse automatique : identifie les fonctions, symboles, chaînes de caractères
- Supérieur à la simple analyse du code assembleur brut

### Comparaison : strncmp
- Fonction C standard pour comparer deux chaînes
- `strncmp(str1, str2, n)` compare les n premiers caractères
- Retourne 0 si les chaînes sont identiques, différent de 0 sinon

---

---

## Conclusion

Ce lab a démontré avec succès que :

1. **Le code natif n'est pas impénétrable** : Bien que plus complexe que le Java, il peut être analysé avec les bons outil.
2.  **La méthodologie est transférable** : Le processus (observation → décompilation → analyse → extraction) s'applique à d'autres applications
3.  **Ghidra est puissant** : Peut transformer du code machine en pseudo-code compréhensible
4.  **L'obfuscation par complexité est faible** : Cacher du code en natif sans vraie sécurité (crypto, signatures) ne garantit pas la protection

### Recommandations pour les Développeurs

- Utiliser la **cryptographie** pour les données sensibles, pas seulement l'obfuscation
- Implémenter la **vérification côté serveur** plutôt que côté client seul
- Utiliser des **signatures numériques** pour vérifier l'intégrité
- Combiner plusieurs couches de sécurité (défense en profondeur)

---

## Fichiers Générés

- `libfoo.so` : Bibliothèque native extraite
- `ghidra_project/` : Projet Ghidra avec analyse complète
- `RAPPORT_UnCrackable_Level2.md` : Ce rapport

---

**Rapport complété le :** 11 Mai 2026  
**Auteur :** Reverse Engineering Lab  
**Niveau de difficulté :** Intermédiaire  
**Status :** ✅ Complété avec succès
