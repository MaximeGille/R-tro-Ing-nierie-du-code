# Analyse du binaire `app01`

> **Type:** ELF 32-bit LSB executable, Intel 80386  
> **Linkage:** Dynamically linked  
> **Stripped:** No

---

## 1. Prologue et Épilogue

### Prologue
```asm
push   ebp           ; sauvegarde ancien base pointer
mov    ebp, esp      ; nouveau frame pointer
```

### Épilogue
```asm
mov    eax, 0x0      ; return 0
leave                ; esp = ebp; pop ebp
ret
```

---

## 2. Calls et Arguments

| Adresse | Fonction | Arguments | Description |
|---------|----------|-----------|-------------|
| `0x804917e` | `puts` | `"Voici le mot de passe à utiliser : ¯\_(ツ)_/¯"` | Affiche le message |

---

## 3. Jumps et Conditions

**Aucun** — Exécution linéaire sans branchement.

---

## 4. Fonctionnement général

Programme **minimaliste** qui affiche un message troll :

1. Affiche la chaîne avec le shrug `¯\_(ツ)_/¯`
2. Retourne 0

### Code C reconstitué

```c
#include <stdio.h>

int main() {
    puts("Voici le mot de passe à utiliser : ¯\\_(ツ)_/¯");
    return 0;
}
```

---

## 5. Informations extraites

| Élément | Valeur |
|---------|--------|
| **Password** | Aucun (troll) |
| **Variables locales** | Aucune |
| **Complexité** | Triviale |

---

## 6. Conclusion

Ce binaire est un **fake** — il ne contient aucune logique de vérification de mot de passe, juste un message humoristique avec un shrug emoji ASCII.
