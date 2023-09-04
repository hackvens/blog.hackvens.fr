---
layout: post
title: Triangle Wars
category: Lehack-2023
---


# Triangle Wars

L'énoncé du problème demande le périmètre, inférieur à 1 000 000, qui présente le plus grand nombre de formations de triangles de Pythagore.

Un triangle de Pythagore est un triangle rectangle dont les longueurs des trois côtés (a, b, c) satisfont au théorème de Pythagore (a² + b² = c²). Notre tâche consiste à énumérer efficacement les triangles possibles et à rechercher les périmètres qui produisent le plus grand nombre de triangles de Pythagore uniques.

Pour une approche optimisée, nous utilisons la formule d'Euclide pour générer des triangles de Pythagore primitifs. La formule stipule que pour chaque paire d'entiers coprimes `m` et `n` où :
 - `m > n > 0`
 - `a = m² - n²`, `b = 2mn`, et `c = m² + n²` 
 - `a² + b² = c²`
 - `m` et `n` ne doivent pas être impairs tous les deux 
 - `perimeter = a + b + c`

```rust
/// Algorithme Euclidien pour trouver le PGCD
fn gcd(a: u64, b: u64) -> u64 {
    match b {
        0 => a,
        _ => gcd(b, a % b),
    }
}

fn main() {
    let limit: u64 = 1_000_000;

    // On initialise un vecteur afin de stocker le nombre de triangles de
    // Pythagore par périmètre
    let mut perimeter_count: Vec<u64> = vec![0; limit as usize + 1];

    let mut result = 0;
    let mut result_count = 0;

    // Valeur maximale de m, pour laquelle a + b + c <= 1 000 000
    let mlimit = (limit as f64).sqrt() as u64;
    for m in 2..=mlimit {
        for n in 1..m {
            // On check si m et n sont coprimes et ne sont pas tous les deux impairs
            if (n + m) % 2 == 1 && gcd(n, m) == 1 {
                // Calculs du triplet de Pythagore
                let mut a = m * m - n * n;
                let mut b = 2 * m * n;
                let c = m * m + n * n;

                if a > b {
                    std::mem::swap(&mut a, &mut b);
                }
                // Calcul du périmètre
                let mut p = a + b + c;

                // On va ajouter les multiples du périmètre, et maj la valeur maximale si besoin
                while p <= limit {
                    perimeter_count[p as usize] += 1;
                    if perimeter_count[p as usize] > result_count {
                        result = p;
                        result_count = perimeter_count[p as usize];
                    }
                    p += a + b + c;
                }
            }
        }
    }

    println!("Result is: {}", result);
}
```

