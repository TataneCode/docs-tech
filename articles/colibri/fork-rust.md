# colibri en C — intérêt d'un fork en Rust

> Catégorie : Notes & dépannage (réflexion d'architecture)

**Date :** 2026-10-28
**Statut :** Terminé

## Objectif

Évaluer si porter (forker) le moteur d'inférence C de [colibri](https://github.com/JustVugg/colibri) en Rust présente un intérêt réel, au-delà de l'attrait « langage moderne ».

## Contexte : ce qui fait la spécificité du code C

Avant de juger le portage, il faut regarder ce que le C fait réellement. Le code n'est pas du C générique : il est délibérément contraint pour des raisons de performance et de portabilité.

- **Zéro dépendance, C pur** : pas de BLAS, pas de framework, pas de runtime Python à l'exécution. Le Makefile compile chaque moteur avec un simple `gcc`/`clang` + OpenMP. C'est un choix de distribution (binaire statique, releases multi-plateformes, build sans outilchain particulier).
- **Un fichier `.c` par famille de modèles** (~29 fichiers `.c`, ~142 fichiers `.c`/`.h`/`.cu`/`.mm`, ~216 k lignes au total) au-dessus de **headers partagés** : `st.h` (safetensors), `quant.h` (décodeurs de conteneurs, noyaux entiers), `expert_ffn.h`/`expert_store.h` (noyau MoE et cache d'experts streamés), `tok.h`, `json.h`, `compat.h` (shims Windows/macOS).
- **Du code « unsafe by design » assumé** : buffers manuels, `pread` sur des offsets calculés, packing de nibbles int4, kernels SIMD écrits à la main, pools de threads maison pour le chargement des experts. La « politique de sécurité mémoire » du projet est la CI : chaque moteur est vérifié token par token contre l'implémentation de référence du modèle sur une fixture tiny (oracle teacher-forcing, `ORACLE_STRICT=1`).
- **Backends GPU optionnels et isolés** : CUDA (`backend_cuda.cu`, compilé par nvcc, avec un shim HIP via `backend_gpu_compat.h`), Metal (`backend_metal.mm`), Vulkan (`backend_vulkan.c` + `vk_tier.c`/`vk_chain.c`), XDNA. Le build par défaut reste CPU pur.
- **Politique stricte** (CONTRIBUTING.md) : `main` reste « known-good », PRs sur `dev`, build sans warning, oracle 30-32/32 en teacher-forcing + 20/20 greedy.

## Ce que Rust apporterait — les vrais gains

| Gain | Pourquoi c'est réel ici |
|---|---|
| **Sûreté mémoire par défaut** | Le domaine de colibri est exactement celui des bugs que Rust élimine : parsing de conteneurs binaires (safetensors, formats de quantification maison), indexation d'experts par offsets, buffers de prefetch concurrents. Un typo d'index ou un dépassement de buffer est le bug classique de ce genre de moteur — et coûte des heures de debug sur des modèles de 400 Go |
| **Concurrence sans data races** | Le moteur a déjà du parallélisme maison : pool de loaders d'experts, thread de prefetch routeur, batching continu sur 16 slots KV, overlap CPU/GPU. Tout ce code est du C avec des primitives manuelles (OpenMP, threads POSIX). Le modèle de possession/emprunt de Rust rend ce code **vérifiable** plutôt que « relu en PR » |
| **Itérateurs et zéro-cost abstractions** | Les boucles de décodage de tokens (route → union → place → overlap → learn) se prêtent bien à des itérateurs sûrs ; le compilateur génère le même code que le C bien écrit |
| **Écosystème** | `wgpu`/`vulkano` pour le backend Vulkan, `candle`-style kernels, `tiktoken`-rs, `safetensors` (crate officiel), `rayon` pour le parallélisme CPU, `memmap2` pour le streaming d'experts depuis le disque. Moins de code maison à maintenir |
| **Distribution** | Un binaire Rust statique reste un seul fichier, cross-compilable — le critère de distribution de colibri est préservé |
| **Testabilité** | Le projet a déjà une culture de tests (oracles, harnesses, fixtures tiny). Rust la renforce nativement (`cargo test`, fuzzing avec `cargo-fuzz`) |

## Ce que Rust coûterait — les vrais freins

| Frein | Détail |
|---|---|
| **Réécriture, pas traduction** | ~216 k lignes de C avec des noyaux SIMD/quantification finement ajustés. Un portage mécanique serait un désastre de performance ; une réécriture propre est un projet de **mois**, pas de semaines. Et la fidélité token-exacte (l'oracle) devrait être re-démontrée moteur par moteur |
| **Performance : parité difficile à prouver** | Le C de colibri est du C *optimisé à la main* (packing int4, noyaux entiers, prefetch logiciel). Rust peut atteindre la parité, mais chaque noyau critique demandera du `unsafe` + SIMD explicite (`std::simd`, `packed_simd` ou intrinsics) et des benchmarks systématiques. Le projet a une règle d'or : une optimisation ne vaut que par une mesure end-to-end reproductible |
| **Backends GPU** | CUDA en Rust existe (`rustacuda`, `cudarc`) mais est jeune ; Metal (`metal-rs`) et surtout le **HIP/ROCm via un shim unique** (le truc astucieux de `backend_gpu_compat.h`) seraient à reconstruire. Le backend Vulkan (`vulkano`/`wgpu`) est le plus accessible |
| **Windows/macOS** | Le C a des shims (`compat.h`) éprouvés ; en Rust il faudra réassembler la matrice de cibles (MSVC, MinGW, ARM64…) |
| **Communauté et rythme** | colibri est un projet actif (169 PRs sur la 2.0.0, 88 contributeurs externes). Un fork Rust diverge : il faut soit porter chaque amélioration upstream (coût permanent), soit upstreamiser (le projet C n'a pas demandé de Rust) |
| **Le « unsafe » ne disparaît pas** | Lecture de fichiers mappés, kernels SIMD, ABI avec les backends GPU : une grande partie du code restera `unsafe`. Rust réduit la surface, ne l'annule pas — le gain est réel mais pas magique |

## Verdict

**Oui, un fork Rust a un intérêt, mais ce n'est pas une évidence.**

L'intérêt est **réel et ciblé** :

1. **Sûreté** sur le parsing de conteneurs et l'indexation mémoire — le cœur du risque de ce type de moteur.
2. **Concurrence vérifiable** pour le streaming d'experts, le prefetch, le batching multi-slots — là où le C maison est le plus fragile.
3. **Maintenabilité** à long terme et onboarding de contributeurs (Rust est plus lisible pour du code unsafe maîtrisé que du C pointer-arithmetic).

Mais le coût est **élevé et structurel** :

- C'est une **réécriture de ~216 k lignes** avec re-validation token-exacte de 13 moteurs, pas un portage incrémental réaliste.
- La **performance brute** ne sera pas un argument de vente : le C actuel est déjà au niveau « noyaux manuels optimisés » ; Rust ne fera pas mieux sans le même travail d'optimisation, et devra le *prouver* par les mêmes benchmarks.
- La **divergence avec upstream** est le risque principal : colibri évolue vite (169 PRs sur une release).

### Quand le fork Rust se justifie

- Si l'objectif est **un moteur maintenable à long terme**, écrit pour durer et contribuer en amont (en proposant le portage au projet, pas en le forkant silencieusement).
- Si l'objectif est **la sûreté** (parsing de formats non fiables, exposition réseau via la passerelle) plutôt que la vitesse.
- Si l'on démarre **un moteur à partir de zéro** sur un sous-ensemble (ex. un seul moteur MoE + Vulkan via `vulkano`), en réutilisant les conteneurs et l'architecture — pas les 13 moteurs d'un coup.

### Quand il ne se justifie pas

- Si l'objectif est **de gagner en performance** : le C est déjà optimisé à la main ; le gain viendrait du travail d'optimisation, pas du langage.
- Si l'on veut **suivre upstream** : le coût de re-portage de chaque PR est prohibitif en l'état.
- Si l'on sous-estime l'effort : le vrai travail est la re-validation de la fidélité numérique, pas la traduction syntaxique.

### Piste pragmatique

Plutôt qu'un fork complet, une approche hybride est plus réaliste :

1. **Garder le C** pour les noyaux de calcul les plus chauds (ils sont déjà au plafond).
2. **Écrire en Rust** les couches périphériques à plus haut risque : parsing de conteneurs/quantification, passerelle API (aujourd'hui Python stdlib), launcher, prefetch/planification — là où la sûreté et la concurrence comptent le plus.
3. **Interfacer par FFI** (C ABI stable, `extern "C"`) : Rust appelle les noyaux C, ou l'inverse.
4. **Contribuer en amont** : le projet est ouvert (Apache 2.0, 88 contributeurs externes sur la dernière release) ; un composant Rust bien ciblé a plus de chances d'être accepté qu'un fork.

## Références

- [Dépôt GitHub colibri](https://github.com/JustVugg/colibri) — C pur, Apache 2.0, v2.0.0
- [CONTRIBUTING.md](https://github.com/JustVugg/colibri/blob/main/CONTRIBUTING.md) — politique de branches (`dev`), oracle CI, fidélité token-exacte
- [GPU_BACKENDS.md](https://github.com/JustVugg/colibri/blob/main/GPU_BACKENDS.md) — architecture des backends CUDA/HIP/Metal/Vulkan
- [Synthèse colibri et mise en place de GLM-5.3-Flash](README.md)
