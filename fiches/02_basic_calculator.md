# Fiche synthèse — Chapitre 02 : Basic Calculator

> Objectif : savoir **expliquer** chaque concept à partir des premiers principes, pas juste
> « faire passer les tests ». Chaque section : le principe → la syntaxe → ce que tu as fait → les pièges.

---

## 0. La philosophie de Rust en 5 principes (le fil rouge de tout le chapitre)

Tout ce chapitre découle de ces 5 idées. Si tu sais les expliquer, tu sais expliquer le chapitre.

| # | Principe | Conséquence concrète dans le chapitre |
|---|----------|----------------------------------------|
| 1 | **Tout a un type connu à la compilation** (typage statique) | `u32`, `i8`, `bool`… ; le compilateur refuse ce qu'il ne peut pas typer (ex. 09) |
| 2 | **Rien n'est implicite** (pas de conversion automatique) | `u8` ≠ `u32` (ex. 01), pas de *truthy/falsy* (ex. 03), `as` obligatoire (ex. 10) |
| 3 | **Immuable par défaut** | `let` ne se modifie pas, il faut `mut` (ex. 06, 07) |
| 4 | **Presque tout est une expression** (ça produit une valeur) | dernière ligne sans `;` = valeur de retour ; `if` renvoie une valeur |
| 5 | **Les erreurs ne sont jamais silencieuses sans ton accord** | `panic!` (ex. 04), overflow = panic en dev sauf si tu choisis autre chose (ex. 08, 09) |

**Phrase à savoir dire :** *« Rust déplace un maximum d'erreurs du runtime vers la compilation. Le
compilateur est un vérificateur de preuves : si le code compile, toute une classe de bugs
(types incohérents, variables non initialisées, mutations non prévues, accès mémoire invalides)
est impossible. »*

---

## 1. Le compilateur et ses « checkers » — comment Rust vérifie ton code

Quand tu fais `cargo build` / `cargo test`, `rustc` passe ton code dans plusieurs vérifications
successives. Dans ce chapitre tu en as croisé plusieurs :

```
Code source
   │
   ▼
1. Parsing ............... syntaxe correcte ? (accolades, `;`, mots-clés)
   │
   ▼
2. Résolution de noms .... `factorial` existe ? `use crate::speed` pointe où ?
   │
   ▼
3. Type checker .......... chaque expression a-t-elle UN type cohérent ?
   │                        (E0308 mismatched types, E0689 ambiguous numeric type)
   ▼
4. Borrow checker ........ les règles d'ownership / mutabilité / initialisation sont respectées ?
   │  (sur le MIR)          (E0384 assign twice to immutable, E0381 used uninitialized,
   │                         E0382 use of moved value)
   ▼
5. Lints ................. avertissements (unused import, overflowing_literals, dead_code…)
   │
   ▼
Code machine (+ checks runtime éventuels : overflow, division par zéro → panic)
```

### 1.1 Le type checker — erreurs rencontrées

| Erreur | Où | Explication |
|--------|----|-------------|
| `E0308 mismatched types: expected u32, found u8` | ex. 01 | `a + b * multiplier` avec `multiplier: u8` : les opérateurs arithmétiques exigent **deux opérandes du même type**. Pas de promotion automatique. |
| `E0308 expected bool, found integer` | cours 03 | `if number { }` interdit : la condition doit être un `bool`. |
| `E0689 can't call method saturating_mul on ambiguous numeric type {integer}` | ex. 09 | voir 1.2 |

### 1.2 Inférence de type : pourquoi tu as dû écrire `let mut result: u32 = 1;` (ex. 09)

- Un littéral `1` seul a un type **provisoire** `{integer}` : « un entier, je ne sais pas encore lequel ».
- Le compilateur résout ce type **en regardant les usages** (unification).
- **MAIS** pour appeler une **méthode** (`result.saturating_mul(i)`), il doit savoir *immédiatement*
  sur quel type chercher la méthode (`u8::saturating_mul` ? `u32::saturating_mul` ?). Il ne peut pas
  attendre → erreur **E0689**.
- Dans l'ex. 07 (`let mut res = 1; res *= i;`), pas besoin d'annotation : `*=` est un opérateur, et
  `i: u32` (car `n: u32`) + le retour `-> u32` suffisent à inférer `u32`.
- S'il n'y a **aucun** indice, le défaut est `i32`.

**Règle :** annotations **obligatoires** pour les paramètres et le type de retour des fonctions
(la signature est un contrat lisible sans lire le corps, et ça accélère la compilation) ;
**optionnelles** pour les `let` quand le contexte suffit.

### 1.3 Le borrow checker — ce qu'il a déjà fait pour toi dans ce chapitre

On ne parle officiellement d'ownership/borrowing qu'au chapitre 03 (`06_ownership`, `10_references_in_memory`),
mais le borrow checker a **déjà** travaillé ici. C'est le composant qui vérifie les règles d'**accès aux
valeurs** : initialisation, mutabilité, possession (ownership), emprunts (références).

#### a) Les 3 règles de l'ownership (à connaître par cœur)

1. Chaque valeur a **un seul propriétaire** (owner) — une variable.
2. Quand le propriétaire sort de sa portée (`}`), la valeur est **détruite** (drop). Pas de garbage collector.
3. On peut **emprunter** une valeur via une référence :
   - soit **N références partagées** `&T` (lecture seule),
   - soit **UNE seule référence mutable** `&mut T`,
   - **jamais les deux en même temps**. → *« aliasing XOR mutation »*.

Et une référence ne doit jamais vivre plus longtemps que la valeur qu'elle pointe (**lifetimes**).

#### b) Où tu l'as vu (sans le savoir)

| Situation | Ce que vérifie le borrow checker | Erreur si violé |
|-----------|----------------------------------|-----------------|
| `let x: u32; let y = x + 1;` | une variable doit être **initialisée** avant lecture | `E0381 used binding x isn't initialized` |
| `let sum = 0; sum += i;` (ex. 06) | on ne réassigne pas une variable **non `mut`** | `E0384 cannot assign twice to immutable variable` |
| `fn factorial(mut n: u32)` (ex. 06) | `n` est une variable locale ; `mut` rend **ce binding** modifiable | — |
| `fn intro() -> &'static str` (ex. 00) | la référence retournée doit rester valide ; `'static` = vit toute la durée du programme (le texte est gravé dans le binaire) | erreur de lifetime |
| `let y = x;` puis réutiliser `x` avec `x: u32` | OK car `u32` est **`Copy`** | — |
| `let t = s;` puis réutiliser `s` avec `s: String` | `s` a été **déplacé** (move), il n'est plus propriétaire | `E0382 borrow of moved value: s` |

#### c) `Copy` vs `move` — pourquoi tous tes entiers « marchent tout seuls »

- Les types primitifs (`u32`, `i8`, `bool`, `char`…) sont **`Copy`** : petits, sur la pile, copiés bit à bit.
  Passer `n` à `factorial(n - 1)` ou écrire `start.abs_diff(end)` **copie** la valeur.
  L'original reste utilisable → aucun souci d'ownership.
- Un `String` possède de la mémoire sur le tas → pas `Copy` → une affectation **déplace** la propriété.
  C'est là que le borrow checker devient visible (chapitre 03).

#### d) `mut` : ce qu'il faut savoir expliquer

- `mut` est une propriété du **binding** (le nom), pas du type ni de la valeur.
- `fn f(mut n: u32)` : l'appelant passe une **copie** ; la fonction modifie sa copie locale.
  L'appelant n'est pas affecté. Ce n'est **pas** un passage par référence.
- Pour modifier la variable *de l'appelant* il faudrait `fn f(n: &mut u32)` et `*n -= 1;` (chap. 03).
- Pourquoi immuable par défaut ? Parce que la règle « aliasing XOR mutation » est beaucoup plus facile à
  garantir si la mutation est **explicite et rare**. Ça rend aussi le code plus lisible : un `let` sans
  `mut` = une valeur qui ne changera jamais.

**Phrase à savoir dire :** *« Le borrow checker garantit statiquement qu'aucune valeur n'est lue
non initialisée, utilisée après avoir été déplacée ou détruite, ou modifiée pendant qu'un autre code la
lit. C'est ce qui donne la sécurité mémoire sans garbage collector, avec un coût nul à l'exécution. »*

---

## 2. Syntaxe de base (cheat sheet)

```rust
// Commentaire simple
/// Commentaire de doc (attaché à l'item suivant → `cargo doc --open`)

// fn <nom>(<param>: <Type>, ...) -> <TypeRetour> { <corps> }
fn compute(a: u32, b: u32) -> u32 {
    let multiplier: u32 = 4;     // let <nom>: <Type> = <expr>;
    a + b * multiplier           // PAS de `;` → c'est la valeur de retour
}

fn rien() { }                    // équivaut à `fn rien() -> () { }`  (() = type unité)

let mut x = 0;                   // variable modifiable
x += 1;                          // += -= *= /= %=

if cond { .. } else if cond2 { .. } else { .. }
let v = if cond { 1 } else { 2 };   // if est une EXPRESSION ; branches de même type

while cond { .. }
for i in 1..=n { .. }            // intervalle inclusif
for i in 1..n  { .. }            // intervalle semi-ouvert (n exclu)

panic!("message");               // `!` = macro
todo!()                          // placeholder qui compile mais panic à l'exécution

42u64          // littéral typé par suffixe
1_000_000      // underscores pour la lisibilité
x as u64       // conversion explicite
```

### Statement vs expression (fondamental)

- **Expression** : produit une valeur (`a + b`, `if … { 1 } else { 2 }`, un bloc `{ … }`).
- **Statement (instruction)** : ne produit pas de valeur (`let x = 5;`, ou n'importe quelle expression suivie de `;`).
- Le `;` **transforme une expression en instruction** et jette sa valeur (le bloc vaut alors `()`).
- Donc :
  ```rust
  fn f() -> u32 { 5 }    // ✅ renvoie 5
  fn g() -> u32 { 5; }   // ❌ E0308 : le bloc vaut (), pas u32
  ```
- `return x` = retour **anticipé**. En fin de fonction, le style idiomatique est de juste écrire `x`.

---

## 3. Les exercices un par un

### Ex. 00 — `00_intro` : fonctions et `&'static str`

```rust
fn intro() -> &'static str {
    "I'm ready to build a calculator in Rust!"
}
```
- **Fait :** compléter la chaîne.
- **À expliquer :** un littéral `"..."` est stocké dans le binaire. La fonction renvoie une **référence**
  (`&`) vers une tranche de texte (`str`) dont la **durée de vie** est `'static` (tout le programme).
- **Tests :** `#[cfg(test)]` = module compilé seulement pour `cargo test` ; `#[test]` marque une fonction de
  test ; `assert_eq!(gauche, droite)` panique si différent ; `use crate::intro` importe depuis la racine du crate.

### Ex. 01 — `01_integers` : types entiers, pas de coercition

```rust
let multiplier: u32 = 4;   // était u8 → E0308
a + b * multiplier
```
- **Fait :** passer `multiplier` de `u8` à `u32`.
- **Principe :** Rust refuse `u32 * u8` même si la conversion serait sans perte. **Toute conversion est explicite.**
- **Tableau des entiers :**

| Bits | Signé | Non signé | Plage non signée | Plage signée |
|------|-------|-----------|------------------|--------------|
| 8 | `i8` | `u8` | 0 → 255 | −128 → 127 |
| 16 | `i16` | `u16` | 0 → 65 535 | −32 768 → 32 767 |
| 32 | `i32` | `u32` | 0 → 2³²−1 (~4,29 Md) | −2³¹ → 2³¹−1 |
| 64 | `i64` | `u64` | 0 → 2⁶⁴−1 | −2⁶³ → 2⁶³−1 |
| 128 | `i128` | `u128` | 0 → 2¹²⁸−1 | −2¹²⁷ → 2¹²⁷−1 |

- Non signé sur n bits : `0 … 2ⁿ − 1`. Signé : `−2ⁿ⁻¹ … 2ⁿ⁻¹ − 1` (un bit « sert » au signe, complément à deux).
- Opérateurs : `+ - * / %`, priorité mathématique. ⚠️ `/` sur entiers = **division entière tronquée** : `5 / 2 == 2`.

### Ex. 02 — `02_variables` : `let`, annotation, inférence

```rust
let distance: u32 = start.abs_diff(end);
distance / time_elapsed
```
- **Fait :** calculer la distance avec `abs_diff` (valeur absolue de la différence, toujours ≥ 0, jamais d'underflow).
  Alternative attendue par le cours : `let distance = end - start;`.
- **« Faut-il annoter `distance` ? »** → **Non.** `abs_diff` sur un `u32` renvoie un `u32`, donc le type est inféré.
  L'annotation est légale mais redondante.
- **Test `case3` :** `speed(10, 31, 10)` → `21 / 10 == 2` (division entière).
- ⚠️ **Nettoyage à faire :** `use std::path::absolute;` est inutile (warning `unused import`) — sans doute
  ajouté par l'autocomplétion en tapant « abs ». À supprimer.

### Ex. 03 — `03_if_else` : branchements et booléens

```rust
fn magic_number(n: u32) -> u32 {
    if n % 2 == 0 {
        return 12
    } else if n % 3 == 0 {
        return 13
    } else {
        return 17
    }
}
```
- **Principe :** la condition doit être un **`bool`** — pas de valeurs *truthy/falsy* comme en Python/JS.
- **Ordre des tests = logique métier :** `6` est pair ET divisible par 3 → le test `six` attend `12`,
  car le premier `if` gagne.
- Comparaisons : `== != < > <= >=`. Logique : `&& || !`.
- **Version idiomatique** (if est une expression) :
  ```rust
  fn magic_number(n: u32) -> u32 {
      if n % 2 == 0 { 12 } else if n % 3 == 0 { 13 } else { 17 }
  }
  ```

### Ex. 04 — `04_panics` : erreur irrécupérable

```rust
if time_elapsed == 0 {
    panic!("The journey took no time at all. That's impossible!")
}
(end - start) / time_elapsed
```
- **Principe :** un `panic` = erreur **irrécupérable** : le thread s'arrête, la pile est déroulée (*unwinding*),
  les valeurs sont détruites proprement. Diviser par zéro panique déjà tout seul (`attempt to divide by zero`),
  ici on remplace par un message **métier** clair.
- **Test :** `#[should_panic(expected = "...")]` = le test réussit **seulement si** ça panique avec un message contenant ce texte.
- `panic!` est une **macro** (le `!`). Son type est `!` (« never ») → compatible avec n'importe quel type attendu,
  c'est pour ça qu'on peut la mettre dans un `if` sans `else`.
- ⚠️ **Le second bug caché** (évoqué par le cours) : `end - start` **underflow** si `end < start`
  (un `u32` ne peut pas être négatif). Voir §4. Avec ton `Cargo.toml` actuel (`overflow-checks = false`),
  ce calcul **ne panique plus** : il donne silencieusement un nombre énorme.
- Plus tard (chap. 05) : erreurs **récupérables** avec `Result<T, E>`.

### Ex. 05 — `05_factorial` : récursion

```rust
fn factorial(n: u32) -> u32 {
    if n == 0 { return 1 }
    else { return n * factorial(n - 1) }
}
```
- **Principe :** cas de base (`0! = 1`) + cas récursif (`n! = n × (n−1)!`).
- **Pourquoi c'est sûr :** quand `n == 0`, on ne calcule jamais `n - 1` → pas d'underflow.
- **À savoir dire :** chaque appel empile une *stack frame* ; Rust **ne garantit pas** l'optimisation des
  appels terminaux (TCO) → pour de grandes profondeurs, préférer une boucle (risque de *stack overflow*).
- Idiomatique : `if n == 0 { 1 } else { n * factorial(n - 1) }`.

### Ex. 06 — `06_while` : boucle + `mut`

```rust
pub fn factorial(mut n: u32) -> u32 {
    let mut res: u32 = 1;
    while n > 1 {
        res *= n;
        n -= 1;
    }
    res
}
```
- **Fait :** accumulateur `res` + décrément de `n`. Condition `n > 1` : couvre `0! = 1` et `1! = 1`
  sans entrer dans la boucle, et évite de multiplier inutilement par 1.
- **Principe clé :** `mut` obligatoire sur `res` **et** sur `n` (sinon E0384).
  `mut n` dans les paramètres = la **copie locale** est modifiable (voir §1.3 d).
- `res` renvoyé sans `;` → valeur de retour.
- **Piège classique :** avec `while n >= 0` sur un `u32` → boucle infinie / underflow, car un `u32` est *toujours* `>= 0`.

### Ex. 07 — `07_for` : `for` et ranges

```rust
pub fn factorial(n: u32) -> u32 {
    let mut res = 1;
    for i in 1..=n {
        res *= i;
    }
    res
}
```
- **Principe :** `for` itère sur un **itérateur** ; plus de compteur manuel → moins de bugs (off-by-one, oubli d'incrément).
- `i` n'a pas besoin de `mut` : c'est une **nouvelle variable** à chaque tour.
- `n = 0` → `1..=0` est **vide** → on renvoie 1. ✅
- **Les 5 ranges :**

| Syntaxe | Contenu | Utilisable dans `for` ? |
|---------|---------|-------------------------|
| `1..5` | 1,2,3,4 (5 exclu) | ✅ |
| `1..=5` | 1,2,3,4,5 | ✅ |
| `1..` | 1 → max du type | ✅ |
| `..5` | min → 4 | ❌ (slicing, patterns) |
| `..=5` | min → 5 | ❌ |

- `res` : type **inféré** `u32` (grâce à `i: u32` et au `-> u32`).

### Ex. 08 — `08_overflow` : profils Cargo et `overflow-checks`

**Aucune modification du `lib.rs`** — tout se passe dans le `Cargo.toml` **racine** :
```toml
[profile.dev]
overflow-checks = false
```
- **Problème :** `20! = 2 432 902 008 176 640 000` ne tient pas dans un `u32` (max ≈ 4,29 milliards).
- **Choix de Rust — pas de promotion automatique** vers un type plus grand (principe n°2). Deux options :
  1. **Refuser** → `panic` (`attempt to multiply with overflow`)
  2. **Wrap around** → on repart de 0, comme un compteur kilométrique : on garde les 32 bits de poids faible,
     c.-à-d. le résultat **modulo 2³²** → `2 192 834 560`.
- **Profils :** `dev` (build/run/test par défaut, compile vite, `overflow-checks = true`),
  `release` (`--release`, optimisé, `overflow-checks = false`), `test` hérite de `dev`, `bench` hérite de `release`.
- **Pourquoi le `Cargo.toml` racine ?** Le repo est un **workspace** (`[workspace] members = ["exercises/*/*"]`) :
  les profils sont définis **uniquement** au niveau du workspace et s'appliquent à tous les membres.
- ⚠️ **Conséquence :** ce réglage s'applique à **tous** les exercices du workspace. Le cours recommande en
  général `overflow-checks = true` même en release (« mieux vaut crasher que produire un résultat faux »).
  Le bloc `[profile.dev.package.copy]` existe justement pour forcer les checks sur un exercice futur.

### Ex. 09 — `09_saturating` : choix au cas par cas

```rust
let mut result: u32 = 1;
for i in 1..=n {
    result = result.saturating_mul(i);
}
```
- **Principe :** `overflow-checks` est un réglage **global et grossier**. Les méthodes permettent de choisir
  **opération par opération** :

| Méthode | Comportement si dépassement | `255u8 + 1` |
|---------|------------------------------|-------------|
| `+` (checks on) | panic | panic |
| `+` (checks off) | wrap | 0 |
| `wrapping_add` | wrap, **toujours** | 0 |
| `saturating_add` | bloque à MAX / MIN | 255 |
| `checked_add` | renvoie `Option` (`None`) | `None` |
| `overflowing_add` | `(résultat_wrappé, bool)` | `(0, true)` |

- Le **saturating** ne s'obtient **que** via les méthodes (pas via un profil).
- `20!` saturé → `u32::MAX`.
- **L'annotation `: u32` était indispensable** : appel de méthode sur un `{integer}` ambigu → E0689 (§1.2).
- Une **méthode** = fonction attachée à un type, appelée avec `valeur.methode(args)` ; la valeur devient le
  premier argument (`self`). Ici `self` est un `u32` `Copy`, donc passé par copie.

### Ex. 10 — `10_as_casting` : conversions explicites

```rust
let v: u32 = 47;              assert_eq!(47u16 as u32, v);   // élargissement : sans perte
let y: i8 = -1;               assert_eq!(255 as i8, y);      // réinterprétation des bits
let v: u8 = 1;                assert_eq!(true as u8, v);     // true → 1, false → 0
```
- **`as` est infaillible** : il produit **toujours** une valeur, même absurde.
- **Grand → petit : troncature** — on garde les bits de **poids faible**. `256u16 as u8 == 0`.
- **Même taille, signe différent : les bits ne changent pas, seule l'interprétation change.**
  `255` = `1111 1111`. En `u8` → 255. En `i8` (complément à deux) → **−1**.
  - Complément à deux : bit de poids fort = −2⁷ = −128. `1111 1111` = −128 + 127 = **−1**.
  - Pour obtenir −x : inverser tous les bits puis +1.
- `#[allow(overflowing_literals)]` : on désactive le lint qui refuse `255 as i8` sur un **littéral**.
  Sur une **variable**, le compilateur ne peut rien détecter.
- **Règle d'or :** `as` uniquement **petit → grand**. Pour grand → petit : `TryFrom` / `try_into()` (chap. 05)
  qui renvoie une erreur au lieu de mentir.

---

## 4. Overflow / underflow — le schéma mental

```
u8 vu comme un cercle :

         254  255
      253        0   ← 255 + 1 en wrapping
     ...          1
      ...        2
         ...  3
```
- **Overflow** : résultat > MAX. **Underflow** : résultat < MIN (ex. `3u32 - 10`).
- Selon le contexte : panic (dev), wrap (release / `overflow-checks = false` / `wrapping_*`),
  saturation (`saturating_*`), `Option` (`checked_*`).
- **Pourquoi c'est grave :** le contrat « `u32 op u32 → u32` mathématiquement juste » est violé. En fintech
  (montants, soldes) un wrap silencieux est catastrophique → privilégier `checked_*` et traiter le `None`.

---

## 5. Tests en Rust (vu dans tous les exos)

```rust
#[cfg(test)]            // compilé uniquement pour `cargo test`
mod tests {
    use crate::factorial;       // import depuis la racine du crate

    #[test]
    fn fifth() { assert_eq!(factorial(5), 120); }

    #[test]
    #[should_panic(expected = "impossible")]
    fn by_zero() { speed(0, 10, 0); }
}
```
- `cargo test -p <nom_du_package>` pour un seul exercice (nom dans le `Cargo.toml` de l'exo, ex. `while_`).
- `#[...]` = **attribut** : métadonnée pour le compilateur.
- `[lints.rust] dead_code = "allow"` dans les `Cargo.toml` des exos : les fonctions privées non utilisées hors
  tests ne génèrent pas d'avertissement.

---

## 6. Questions d'oral — sais-tu répondre ?

1. Pourquoi `let m: u8 = 4; a * m` (avec `a: u32`) ne compile-t-il pas, alors qu'aucune information ne serait perdue ?
2. Quelle différence entre `5` et `5;` en fin de fonction ?
3. Pourquoi les paramètres de fonction doivent-ils être typés et pas les `let` ?
4. Pourquoi `let mut r = 1; r = r.saturating_mul(i);` ne compile pas, alors que `r *= i` compile ?
5. Que fait exactement `fn f(mut n: u32)` ? L'appelant voit-il la modification ?
6. Pourquoi `let y = x;` puis utiliser `x` marche avec un `u32` mais pas avec un `String` ?
7. Quelles sont les 3 règles de l'ownership ? Que signifie « aliasing XOR mutation » ?
8. Que vaut `factorial(20)` en `u32` en dev par défaut ? avec `overflow-checks = false` ? avec `saturating_mul` ?
9. Pourquoi faut-il modifier le `Cargo.toml` **racine** et pas celui de l'exercice ?
10. Pourquoi `255 as i8 == -1` ? Explique avec les bits.
11. Pourquoi `as` est dangereux, et que faut-il utiliser à la place pour grand → petit ?
12. Quel bug reste caché dans `speed` de l'ex. 04 ? Comment le corriger ? (`abs_diff`, `checked_sub`…)
13. `panic!` vs `Result` : quand utiliser l'un ou l'autre ?
14. Pourquoi Rust est-il immuable par défaut ?

<details>
<summary>Réponses courtes</summary>

1. Rust ne fait aucune conversion implicite : les opérateurs exigent deux opérandes du même type. Il faut `m as u32` ou typer `m: u32`.
2. `5` est une expression → valeur de retour. `5;` est une instruction → le bloc vaut `()` → erreur de type.
3. La signature est un contrat : lisible sans lire le corps, permet de typer chaque fonction indépendamment (compilation plus rapide, erreurs locales).
4. Un appel de méthode exige un type concret pour savoir quelle méthode appeler ; `{integer}` est ambigu → E0689. Un opérateur laisse l'inférence se terminer.
5. `n` est une copie locale modifiable. L'appelant ne voit rien. Pour modifier sa variable : `&mut u32`.
6. `u32` est `Copy` (copie bit à bit) ; `String` possède de la mémoire sur le tas → move → l'ancienne variable est invalidée (E0382).
7. Un seul propriétaire ; destruction à la fin de la portée ; N références `&` OU une seule `&mut`. On ne peut pas à la fois partager et modifier.
8. Panic (`attempt to multiply with overflow`) ; `2 192 834 560` (modulo 2³²) ; `u32::MAX` = 4 294 967 295.
9. Le repo est un workspace Cargo : les profils ne se configurent qu'à la racine du workspace.
10. Les bits `1111 1111` sont conservés ; lus en complément à deux : −128 + 127 = −1.
11. Il tronque ou réinterprète silencieusement. Utiliser `TryFrom` / `try_into()` qui renvoie une erreur.
12. `end - start` underflow si `end < start`. → `start.abs_diff(end)` ou `end.checked_sub(start)`.
13. `panic!` : bug / état impossible, irrécupérable. `Result` : échec prévisible que l'appelant doit gérer.
14. Rendre la mutation explicite et rare → code plus lisible et garantie « aliasing XOR mutation » facile à vérifier statiquement (et concurrence sûre).
</details>
