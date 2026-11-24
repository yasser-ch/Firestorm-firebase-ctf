# 🔥 Lab 18 : FireStorm — Résolution CTF Android + Firebase


---

## Objectif

L'application Android contient une fonction `Password()` qui génère un mot de passe Firebase mais **n'est jamais appelée** dans le flux normal. L'objectif est de forcer son exécution avec Frida, obtenir le mot de passe, puis s'authentifier sur Firebase pour récupérer le flag.

- **Niveau :** Medium
- **Techniques :** Reverse Android (JADX), hooking Java (Frida), authentification Firebase, Python

---

## Flux du Challenge

```
APK FireStorm
      │
      ├── JADX → Identifier Password() + strings.xml (email, apiKey, databaseURL)
      │
      ├── Frida → Java.choose(MainActivity) → instance.Password() → mot de passe
      │
      └── Python (pyrebase) → auth.sign_in() → db.get() → FLAG
```

---

## Étape 1 — Préparation de l'Environnement

```bash
# Installer l'APK
adb install FireStorm.apk

# Vérifier Frida
frida-ps -U
# frida-server doit être en cours d'exécution sur l'appareil
```

---

## Étape 2 — Analyse Statique avec JADX

```
1. Ouvrir l'APK dans JADX-GUI
2. Naviguer : com.pwnsec.firestorm → MainActivity
```

### Méthode Password() — Ce qu'elle fait

```java
public String Password() {
    String s1 = getString(R.string.s1);
    String s2 = getString(R.string.s2);
    // ... récupération d'autres strings depuis strings.xml ...

    // Appel à la fonction native dans libfirestorm.so
    String randomPart = generateRandomStrings(...);

    // Construction du mot de passe final
    String finalPassword = s1 + s2 + randomPart + ... ;

    return finalPassword;
}
```

**Points clés :**

| Observation | Signification |
|-------------|--------------|
| `Password()` existe dans le code | Logique de génération présente |
| Jamais appelée dans `onCreate` ou les listeners | **Inactive par défaut** — c'est le challenge |
| Appel à `generateRandomStrings()` dans `libfirestorm.so` | Partie dynamique native |
| `strings.xml` contient email + config Firebase | Informations d'authentification |

### Informations dans strings.xml

```xml
<string name="firebase_api_key">AIzaSyAXsK0qsx4RuLSA9C8IPSWd0eQ67HVHuJY</string>
<string name="firebase_email">TK757567@pwnsec.xyz</string>
<string name="firebase_database_url">https://firestorm-9d3db-default-rtdb.firebaseio.com</string>
```

> ⚠️ Le mot de passe **change à chaque lancement** à cause de la partie générée par la fonction native

---

## Étape 3 — Script Frida : Forcer l'Exécution de Password()

**Fichier : `frida_firestorm.js`**

```javascript
Java.perform(function() {

    function getPassword() {
        console.log("[*] Début de la recherche d'instances de MainActivity...");

        Java.choose('com.pwnsec.firestorm.MainActivity', {

            onMatch: function(instance) {
                console.log("[+] MainActivity instance trouvée : " + instance);

                try {
                    // Appel manuel de la méthode Password()
                    var pass = instance.Password();
                    console.log("[+] Mot de passe Firebase généré : " + pass);
                } catch (e) {
                    console.log("[-] Erreur lors de l'appel de Password() : " + e);
                }
            },

            onComplete: function() {
                console.log("[*] Recherche des instances terminée.");
            }
        });
    }

    // Délai pour que l'app et la lib native soient chargées
    console.log("[*] Script chargé. Attente de 3 secondes avant exécution...");
    setTimeout(getPassword, 3000);
});
```

### Explication du Script

| Élément | Rôle |
|---------|------|
| `Java.perform(function() {...})` | Initialise l'environnement Java de Frida |
| `Java.choose('com.pwnsec.firestorm.MainActivity', ...)` | Parcourt la mémoire pour trouver les instances vivantes de MainActivity |
| `onMatch: function(instance)` | Appelée pour chaque instance trouvée (équivalent au `this` Java) |
| `instance.Password()` | Appel direct de la méthode — déclenche aussi la fonction native |
| `setTimeout(getPassword, 3000)` | Délai pour laisser l'app démarrer et charger `libfirestorm.so` |

### Lancement

```bash
frida -U -f com.pwnsec.firestorm -l frida_firestorm.js --no-pause
```

**Sortie attendue :**
```
[*] Script chargé. Attente de 3 secondes avant exécution...
[*] Début de la recherche d'instances de MainActivity...
[+] MainActivity instance trouvée : <instance>
[+] Mot de passe Firebase généré : [MOT_DE_PASSE]
[*] Recherche des instances terminée.
```

> 📋 **Copier le mot de passe affiché** — il sera utilisé à l'étape suivante

---

## Étape 4 — Script Python : Authentification Firebase + Récupération du Flag

```bash
# Installer pyrebase
pip install pyrebase4
```

**Fichier : `get_flag.py`**

```python
import pyrebase

config = {
    "apiKey": "AIzaSyAXsK0qsx4RuLSA9C8IPSWd0eQ67HVHuJY",
    "authDomain": "firestorm-9d3db.firebaseapp.com",
    "databaseURL": "https://firestorm-9d3db-default-rtdb.firebaseio.com",
    "storageBucket": "firestorm-9d3db.appspot.com",
    "projectId": "firestorm-9d3db"
}

firebase = pyrebase.initialize_app(config)
auth = firebase.auth()

email = "TK757567@pwnsec.xyz"
password = "LE_MOT_DE_PASSE_OBTENU_AVEC_FRIDA"  # Remplacer par le mot de passe Frida

user = auth.sign_in_with_email_and_password(email, password)
print("[+] Connexion réussie. Token obtenu.")

db = firebase.database()

# Récupération du flag depuis la base de données
flag_data = db.get(user['idToken'])
print("[+] FLAG récupéré :")
print(flag_data.val())
```

### Explication du Script Python

| Étape | Description |
|-------|-------------|
| `pyrebase.initialize_app(config)` | Connexion au projet Firebase |
| `auth.sign_in_with_email_and_password()` | Authentification avec email statique + mot de passe dynamique |
| `db.get(user['idToken'])` | Lecture de la DB (autorisée uniquement après auth valide) |
| `flag_data.val()` | Valeur contenant le flag |

```bash
python get_flag.py
```

---

## Étape 5 — Flag

```
PWNSEC{C0ngr4ts_Th4t_w45_4N_345y_P4$$w0rd_t0_G3t!!!0R!5_!t???}
```

---

## Pourquoi ce Challenge est Intéressant

| Concept | Démonstration |
|---------|--------------|
| Dead code | `Password()` présente mais jamais appelée |
| Frida `Java.choose` | Accès aux instances d'objets en mémoire sans référence directe |
| Fonction native | Partie du mot de passe générée en C++ dans `libfirestorm.so` |
| Firebase auth | Authentification programmatique depuis Python |
| CTF réaliste | Simule une app avec logique de sécurité incomplète |

---

## Checklist de Résolution

- [ ] APK installé et Frida opérationnel (`frida-ps -U`)
- [ ] JADX : `Password()` identifiée + email + apiKey + databaseURL notés
- [ ] Script Frida lancé → mot de passe affiché dans la console
- [ ] Script Python exécuté → authentification Firebase réussie
- [ ] FLAG récupéré depuis la Realtime Database

---

## Référence du Lab

- **Numéro du lab :** 18 (Sécurité Android)
- **Titre :** FireStorm — Résolution CTF Android + Firebase
- **Outils :** JADX-GUI, Frida, Python (pyrebase4), ADB
- **Concepts clés :** Dead code, `Java.choose`, fonctions natives, Firebase auth, CTF Android
